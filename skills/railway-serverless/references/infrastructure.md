# Railway infrastructure checklist

Settings and background activity that keep a service awake even when the application code is correct. Check these first when a service refuses to sleep.

## Contents

- Healthchecks
- Scaling configuration
- Timers and telemetry
- Persistent connections
- Finding what keeps a service awake

## Healthchecks

Open the service's Settings, then Healthcheck. If a path is set (`/`, `/health`, `/docs`), Railway requests it every few seconds, which generates packets and resets the 5 minute idle countdown. Remove the path or disable the healthcheck.

## Scaling configuration

Under Settings, then Scaling, turn on Serverless (Scale to 0) and set Min Replicas to 0. On an existing service, enabling serverless only takes effect after a new deployment.

## Timers and telemetry

- Framework telemetry sends periodic HTTP requests. Set `NEXT_TELEMETRY_DISABLED=1` for Next.js and check the equivalent switch for Prisma and any other framework in use.
- Do not run long-polling `setInterval` loops or cron schedulers inside the service. Use Railway cron services, which start a container, run, and exit.
- Cron expressions inside the app that fire every 1, 5, or 30 minutes should move to a daily schedule or a separate cron service.

## Persistent connections

- Redis `subscribe()` holds a socket open for good, so a service that uses it cannot sleep. Replace it with ephemeral polling or webhooks.
- Any other long-lived connection (websocket servers, database `LISTEN`, message broker consumers) has the same effect.

## Finding what keeps a service awake

Work through these in order. The earlier ones are more common.

1. Healthcheck path configured.
2. In-app crons or `setInterval`.
3. Pool with `min > 0`, or no `idleTimeoutMillis`.
4. Redis client created at import time, or never disconnected when idle.
5. Orphaned `ioredis` instances left running after a failed connect.
6. Redis subscribers or other persistent sockets.
7. Telemetry env vars.
8. Another service polling this one. Private traffic counts for both ends, so a chatty neighbor keeps this service awake too.

Item 8 matters when two services both refuse to sleep. Sleeping one may need the other to stop calling it.
