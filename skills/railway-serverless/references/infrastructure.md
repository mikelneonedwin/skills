# Railway infrastructure and background jobs

Settings, background schedulers, queues, and patterns that keep a service awake even when application request code is correct.

## Contents

- Railway healthchecks
- Background schedulers and in-process crons
- On-demand payment reconciliation
- Queues and workers (Bull / BullMQ)
- Railway dashboard configuration

## Railway healthchecks

Railway healthchecks run exclusively during deployment phase validation to confirm a new deployment can receive traffic before tearing down the previous replica. Railway does not poll healthcheck paths once a deployment is live. Do not disable or remove deployment healthchecks under the misconception that they prevent scale-to-0.

## Background schedulers and in-process crons

Railway puts containers to sleep based on outbound network inactivity. Inactive services have their processes frozen.

- **Frozen processes do not tick**: An in-process `@Cron()` scheduler cannot wake a sleeping service. If a service is asleep at midnight, `@Cron(CronExpression.EVERY_DAY_AT_MIDNIGHT)` will not fire.
- **Background network pings prevent sleep**: A timer that runs a database query or external API call every 5 minutes generates outbound packets, which continuously resets Railway's idle timer and prevents the service from ever sleeping.
- **Native Railway Cron Schedule**: Move scheduled jobs to dedicated batch worker services configured with a Cron Schedule in Railway's dashboard. Railway starts the container on schedule, executes the command, and tears it down when the process exits.
- **External HTTP triggers**: If a sleeping service must perform periodic work, trigger it from an external cron dispatcher or un-sleeping worker via an authenticated HTTP request. The incoming HTTP request wakes the container via Railway's proxy, runs the task, and allows it to sleep after 10 minutes of inactivity.

## On-demand payment reconciliation

Instead of running a periodic background cron every 5 to 10 minutes that queries payment providers and keeps the database awake, reconcile reactively:

1. **On webhook arrival**: When a payment gateway webhook arrives, finalize transaction state immediately.
2. **On user request**: When a user visits their course, invoice, or dashboard, trigger an on-demand reconciliation check for pending transactions. This pattern is proven in production (`patnaar` and `modools`).
3. **Lazy cleanup**: Purge expired tokens, sessions, or temporary records during incoming authenticated requests rather than running a background timer loop.

## Queues and workers (Bull / BullMQ)

A default Bull or BullMQ setup holds persistent Redis subscriber connections and continuous polling loops, keeping both the worker container and the Redis container awake permanently.

To make queue workers compatible with scale-to-0:

1. **Ephemeral workers**: Run worker processes as separate services configured with scale-to-0, or trigger them via webhooks/cron when work exists.
2. **Close idle queue connections**: Bull and BullMQ instances maintain three Redis connections (client, subscriber, and bclient). Ensure that when queues are drained, workers unref or pause their listening sockets after an idle threshold rather than blocking indefinitely with `BRPOP`.
3. **Strict connection parameters in BullModule**: Ensure Redis connections passed to Bull explicitly disable keep-alive (`keepAlive: false as any`), restrict retries, and prevent persistent retries during cold-start phases.
4. **Bull internal timer lifecycle**: Calling `queue.pause(true, true)` stops workers from pulling new jobs from Redis, but it leaves Bull's internal recurring timers (`updateDelayTimer` running every `guardInterval`, and `moveUnlockedJobsToWaitInterval` running every `stalledInterval`) active. These timers run Lua scripts (`updateDelaySet` and `moveStalledJobsToWait`) against Redis continuously, preventing both the application container and the Redis container from sleeping.
5. **Never set zero intervals in Bull settings**: Setting `guardInterval: 0` or `stalledInterval: 0` in Bull settings destroys the state machine. In Bull's codebase, `updateDelayTimer` checks `const delay = Math.min(delayedTimestamp - Date.now(), guardInterval)`. If `guardInterval` is 0, `delay <= 0` is always true, which makes Bull call `this.updateDelayTimer()` synchronously and recursively in an infinite tight loop, flooding Redis with thousands of `evalsha` commands per second and consuming 100% CPU. Setting `stalledInterval: 0` causes Bull's Lua script to send an invalid expiration timestamp, triggering `ERR invalid expire time in 'set' command script`. Always leave Bull's default intervals intact.
6. **Safe idle timer clearing and resume pattern**:
   - Guard `updateDelayTimer` on the queue instance so that it returns `Promise.resolve()` while an idle/paused flag is set (`isPaused === true`).
   - When queues are empty (`drained` event or startup check with 0 waiting/active/delayed jobs), wait for an idle grace period (e.g. 15 seconds) and set `isPaused = true`.
   - Call `await queue.pause(true, true)` to pause local dequeuing, then call `queue._clearTimers()` to cancel existing `delayTimer`, `guardianTimer`, and `moveUnlockedJobsToWaitInterval` instances. Because `updateDelayTimer` is guarded, no recurring timers or Lua commands will run against Redis while idle.
   - When a job arrives on the `waiting` event or an explicit wake endpoint (`POST /wake`) is called, set `isPaused = false`, call `await queue.resume(true)`, invoke `queue.updateDelayTimer()`, and invoke `queue.startMoveUnlockedJobsToWait()` to restore normal processing.

## Railway dashboard configuration

1. Under **Settings -> Scaling**, toggle **Serverless (Scale to 0)** to On and set **Min Replicas = 0**. On an existing service, enabling serverless requires a new deployment to take effect.
2. Disable framework telemetry: Set `NEXT_TELEMETRY_DISABLED=1` for Next.js, and disable similar telemetry mechanisms in other frameworks that phone home periodically.
3. Audit both sides of every network link: When Service A and Service B communicate, inspect both service configurations. If Service B refuses to sleep, verify whether Service A is sending periodic pings or polling endpoints.
