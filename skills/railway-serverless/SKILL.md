---
name: railway-serverless
description: Build, configure, audit, or review services on Railway Serverless (scale-to-0, App Sleeping). Covers mutual resilience between callers and callees, Postgres (pg.Pool and postgres.js), Redis (ioredis), idempotent inter-service HTTP (fetch, Axios), background jobs and native Railway cron schedules, on-demand payment reconciliation, queues (Bull/BullMQ), quiet non-spamming logging during wake-up phases, and Railway dashboard configurations.
---

# Railway serverless (scale-to-0)

Railway pauses containers after 5 to 10 minutes of complete outbound network inactivity. Private networking traffic counts against both communicating services, meaning recurring outbound network packets or active TCP probes keep both ends alive indefinitely. Waking an idle container takes 2 to 8 seconds. Any client making a request during that boot window will encounter connection drops or HTTP 502/503 responses unless designed specifically to absorb the cold start.

Configuring serverless is never an isolated per-service setting. It is an ecosystem contract spanning callers, callees, background jobs, database connection pools, cache clients, and operational logging.

## Core principles and system contracts

Every serverless failure traces back to one of three flaws:

1. Unintended outbound traffic generation: uncoordinated background timers that query databases or remote services, periodic keep-alive probes, persistent subscriptions, or external pings reset the idle countdown before sleep can occur.
2. Fragile or non-idempotent callers: upstream callers timing out or failing immediately instead of giving sleeping dependencies time to boot, or blindly retrying mutating operations and duplicating transactions.
3. Obtrusive logging during cold starts: treating expected initial socket drops as application crashes and spamming logs with false positive error alerts.

### Caller and callee mutual resilience

Scale-to-0 fails when an agent configures Service B to sleep but leaves Service A unprepared for Service B's sleep state.

- Upstream callers must absorb cold starts: If Service A calls Service B, Service A must have an overall HTTP timeout budget of at least 25 to 30 seconds bounding all attempts combined, retrying transient network errors (`ECONNREFUSED`, `ECONNRESET`, `ETIMEDOUT`).
- Idempotency guardrails: Only idempotent methods (`GET`, `HEAD`, `OPTIONS`, `PUT`, `DELETE`) or requests carrying an explicit idempotency key can be retried on socket drops, timeouts, or 502/503 responses. Never retry mutating operations (`POST`, `PATCH`) without an idempotency key, because the upstream service may have committed state before returning an error or dropping the socket.
- Upstream callers must not poll Service B on an interval: A frequent synchronization poll from Service A resets Service B's idle timer and keeps both services awake.
- Callees must not hold long-lived connections back to callers: WebSockets, server-sent events, or persistent RPC subscriptions defeat scale-to-0.

### Quiet logging during cold-start resilience

During a cold start, initial connection failures (`ECONNREFUSED`, `ETIMEDOUT`, `57P03`) are not system outages. They represent normal, expected wake-up latency.

- Suppress error and warning logs during retry phases: Do not emit `logger.error` or `logger.warn` on the first few connection attempts while a service or database is waking up.
- Log only at `debug` level during ongoing retries, or stay completely silent until the retry budget is exhausted.
- Emit `logger.error` only when maximum retries or the total time budget has been exceeded.
- Attach an empty or debug-level error listener to EventEmitter clients (such as `ioredis` or `pg`) to prevent Node from throwing unhandled error events or logging stack traces to stderr during routine reconnect attempts.

## Step-by-step workflow

Follow this sequence whenever building, converting, or auditing a service for Railway serverless.

1. **Reconnaissance**: Identify all external touchpoints across the service:
   - Relational database connections (`pg`, `drizzle-orm/node-postgres`, `postgres`, `drizzle-orm/postgres-js`).
   - Redis or caching clients (`ioredis`, `@nestjs/bull`, `bull`, `bullmq`).
   - Outbound HTTP calls to sibling microservices or upstream dependencies (`fetch`, `axios`, `ky`).
   - Schedulers and intervals (`@nestjs/schedule`, `@Cron`, `setInterval`, queue polling loops).
2. **Consult Reference Implementations**: Read the matching reference file in `references/` before writing code. Reference implementations contain verified concurrency safeguards, timeout bounding, and unref'd timer mechanics:
   - For PostgreSQL pools and ORMs: read `references/postgres.md`.
   - For Redis clients and cache managers: read `references/redis.md`.
   - For internal HTTP calls: read `references/http-clients.md`.
   - For background timers, queues, payment reconciliation, and dashboard settings: read `references/infrastructure.md`.
3. **Configure Upstream Callers**: Inspect services that call this service. Verify their request timeouts absorb 20 to 25 seconds of wake-up latency and that network error retries enforce idempotency.
4. **Audit and Verify**: Verify that no interval loops make outbound network requests, idle sockets close within 10 seconds, healthchecks remain configured for deployment validation, and retry logging remains quiet until failure.

## Quick reference checklist

| Surface | Must do | Must avoid |
|---|---|---|
| PostgreSQL (`pg.Pool`) | Set `connectionTimeoutMillis: 15000`, intercept both promise and callback `connect()`, release late clients if timed out | Leaving `connectionTimeoutMillis: 0` (indefinite hang), using uncancelled `Promise.race()` |
| PostgreSQL (`postgres.js`) | Set `connect_timeout: 25`, `idle_timeout: 10`, `max: 10` | Unbounded connection queues, default infinite connection timeouts |
| Redis (`ioredis`) | Set `lazyConnect: true`, `enableOfflineQueue: false`, `keepAlive: false as any`, reference-count operations, use explicit retry loop for cold starts | Setting `keepAlive: 0` (enables keep-alive), uncoordinated idle timers, relying solely on `retryStrategy` after `connect()` |
| Inter-service HTTP | Bound overall duration across all retries, preserve caller `AbortSignal`, gate retries by idempotency key or method | Retrying mutating `POST`/`PATCH` without idempotency key, multiplying timeout per attempt |
| Background tasks & crons | Use native Railway Cron Schedule or reactive on-demand reconciliation on webhook/request | In-process cron loops running every few minutes, expecting an in-process cron to wake a sleeping container |
| Healthchecks & Dashboard | Leave deployment healthchecks active, toggle Serverless to On with min replicas 0, redeploy | Disabling deployment healthchecks, expecting background health checks that don't exist |
