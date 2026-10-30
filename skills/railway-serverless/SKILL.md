---
name: railway-serverless
description: Rules for building, configuring, debugging, or reviewing anything that runs on Railway Serverless (scale-to-0, App Sleeping). Use this skill automatically whenever the task touches Railway serverless, including sleeping or scale-to-0 services, services that will not sleep, cold starts, wake-up latency, ETIMEDOUT or ECONNREFUSED after idle, 502/503 from a sleeping upstream, pg.Pool or Drizzle or Better Auth config, ioredis or Redis clients, inter-service HTTP clients (fetch, axios, ky), in-app crons or setInterval timers, healthchecks, or Railway scaling settings. Trigger even when the user only says "Railway" plus Postgres, Redis, API, worker, or cron and never writes the word "serverless", because a default config in any of these will keep the container awake or fail on wake-up.
---

# Railway serverless (scale-to-0)

Railway pauses a container after 5 to 10 minutes with no network packets in or out. Private network traffic counts for both ends, so one chatty connection keeps two services awake. Waking a sleeping service takes 2 to 8 seconds, and requests that arrive in that window fail unless the client expects it.

Every rule below follows from those two facts. Either something generates packets when no one is using the app, or something gives up before the wake-up finishes.

## How to use this skill

1. Find which parts of the task touch Postgres, Redis, HTTP calls between services, background timers, or Railway settings.
2. Read the matching reference file before writing or editing code. The references hold the full reasoning and working implementations to copy from.
3. Check the work against the rules below. If existing code breaks a rule, say so, even when the user did not ask about it.
4. For new code, start from the reference implementation instead of writing from memory. The retry logic has a subtle trap (see Postgres) that is easy to reintroduce.

| Task involves | Read |
|---|---|
| `pg`, Drizzle, Better Auth, any Postgres pool | `references/postgres.md` |
| `ioredis`, any Redis client | `references/redis.md` |
| Calls from one service to another (fetch, axios, ky) | `references/http-clients.md` |
| Crons, timers, workers, healthchecks, telemetry, Railway dashboard settings | `references/infrastructure.md` |

## Core rules

### Postgres (`pg.Pool`)

- Set `min: 0`. Any minimum above zero holds sockets open forever.
- Set `idleTimeoutMillis: 10000` so idle sockets close 10 seconds after the last request.
- Set `connectionTimeoutMillis: 15000`. The database needs 5 to 8 seconds to wake, and a 5 second timeout aborts first.
- Set `max: 10` per replica so a wake-up burst cannot exhaust `max_connections`.
- Wrap connection acquisition in a retry loop with exponential backoff and jitter. Retry on `ECONNREFUSED`, `ETIMEDOUT`, `EHOSTUNREACH`, `ECONNRESET`, `ENETUNREACH`, and Postgres codes `57P03`, `08006`, `08001`, `08004`.
- Override both signatures of `pool.connect()`. Drizzle and Better Auth call `pool.query()`, and `pg` implements that with the callback form of `connect(cb)`. A retry that only covers the promise form is skipped by every ORM query, which then fails immediately with `ETIMEDOUT`.

### Redis (`ioredis`)

- `lazyConnect: true`, so importing the module never opens a socket.
- `enableOfflineQueue: false` and `maxRetriesPerRequest` of 1 or 2, so commands fail fast instead of buffering forever while Redis wakes.
- Share one in-flight `connect()` promise between concurrent callers. A second `connect()` call throws, and commands sent while status is `connecting` throw with the offline queue off. Never reset `connectPromise` in the disconnect path. Only its own `finally` clears it.
- Absorb wake-up with `retryStrategy`, 6 to 8 attempts, 250 ms growing to 3000 ms, plus jitter. Network drops reach `retryStrategy`. `reconnectOnError` only sees command-level error replies, so limit it to `READONLY`.
- Close idle connections with `client.disconnect()` after 5 to 15 seconds, using an unref'd timer. `quit()` waits on a handshake that a half-closed proxy never answers.
- On a failed connect, call `disconnect()` on the failed client before dropping the reference. A dropped but undisconnected client keeps retrying in the background and keeps both services awake.
- `keepAlive: 0` and `autoResendUnfulfilledCommands: false`. Keep-alive probes are packets.
- No `subscribe()`. A subscriber holds a socket open permanently. Use ephemeral polling or webhooks.

### Calls between services

- Default timeout of 20 to 25 seconds.
- Retry up to 3 times on 502, 503, `ECONNREFUSED`, `ECONNRESET`, `ETIMEDOUT`. Back off 1s, 2s, 4s with jitter.

### Timers, workers, and settings

- No in-app `@Cron` or `setInterval` firing every few minutes. Each tick hits the database and resets the idle countdown. Use a daily schedule, a Railway cron service, or an external webhook.
- Clean up expired rows lazily on incoming requests instead of polling for them.
- Trigger queue workers when work is enqueued, drain the queue, then go idle. Do not poll the queue on an interval.
- Remove the healthcheck path in service settings. Railway pings it every few seconds.
- Confirm Serverless is on and Min Replicas is 0. Turning it on for an existing service needs a new deployment.
- Disable framework telemetry, for example `NEXT_TELEMETRY_DISABLED=1`.

## Reviewing existing code

When asked to audit a service, check in this order, because the first hit usually explains a service that never sleeps.

1. Healthcheck path set in Railway.
2. Crons or intervals inside the app.
3. Pool config with `min > 0`, no `idleTimeoutMillis`, or a short connection timeout.
4. Redis client built at import time, or no idle disconnect.
5. Redis `subscribe()` or other long-lived connections.
6. Telemetry env vars.
7. Inter-service clients with short timeouts and no retries.

Report each finding with the file and line, the rule it breaks, and the fix.

## Things to avoid

- Do not raise `min` or add a keep-warm ping to cut cold-start latency. That defeats scale-to-0. The retries and long timeouts exist so the cold start is absorbed instead.
- Do not put network errors in `reconnectOnError`.
- Do not use `quit()` for idle cleanup.
- Do not assume a retry wrapper works until you have checked that it covers `pool.query()`.
