# Postgres on Railway serverless

Covers `pg.Pool`, Drizzle, and Better Auth. Both the API and the database can sleep, and both only sleep once every socket between them is closed.

## Contents

- Why default pools keep services awake
- Pool settings and reasons
- The callback bypass trap
- Reference implementation
- Background work that touches the database

## Why default pools keep services awake

A pool with `min` above zero, or with no idle timeout, holds TCP sockets open between the API and Postgres. Keep-alives and status checks travel over the private network on those sockets, which counts as activity for both containers.

## Pool settings and reasons

| Setting | Value | Reason |
|---|---|---|
| `min` | `0` | The pool opens no connections when idle. |
| `idleTimeoutMillis` | `10000` | Idle clients are destroyed after 10 seconds, so traffic drops to zero and the 5 minute countdown can run. |
| `connectionTimeoutMillis` | `15000` | A waking Postgres container needs 5 to 8 seconds for disk mount and readiness. A 5 second timeout fails with `ETIMEDOUT` first. |
| `max` | `10` | Caps connections per replica so a burst after wake-up cannot exhaust `max_connections` before idle clients are reaped. |

Add a retry loop on top of the timeout. Use exponential backoff with jitter, at most 5 attempts, and stop once total wait would exceed 15 seconds.

## The callback bypass trap

`pg` exposes `pool.connect()` in two forms, promise (`await pool.connect()`) and callback (`pool.connect((err, client, done) => ...)`). Drizzle and Better Auth run queries through `pool.query()`, and inside `pg`, `pool.query()` always acquires its client with the callback form, `this.connect(cb)`.

If a custom pool adds retries only to the promise form and passes the callback form to `super.connect(cb)`, every ORM query skips the retry loop and fails with `ETIMEDOUT` the first time the database is asleep.

The fix is to send both forms through one retry method (`acquireResilientClient` below) and give the callback a release handler that calls `client.release()`.

When reviewing or editing an existing pool wrapper, confirm both branches of `connect()` reach the retry logic.

## Reference implementation

This is the production version used for NestJS and Drizzle. Adapt the logger and env import to the project. The logic should stay as it is.

```ts
import { InternalServerErrorException, Logger } from "@nestjs/common";
import { drizzle } from "drizzle-orm/node-postgres";
import { Pool, type PoolClient } from "pg";
import { env } from "../env.js";
import * as schema from "./schemas/index.js";

const logger = new Logger("DatabasePool");

const RETRYABLE_CONNECTION_CODES = new Set([
  "ECONNREFUSED",
  "ETIMEDOUT",
  "EHOSTUNREACH",
  "ECONNRESET",
  "ENETUNREACH",
  "57P03", // cannot_connect_now
  "08006", // connection_failure
  "08001", // sqlclient_unable_to_establish_sqlconnection
  "08004", // sqlserver_rejected_establishment_of_sqlconnection
]);

type ConnectCallback = (
  err: Error | undefined,
  client: PoolClient | undefined,
  done: (release?: unknown) => void,
) => void;

class ResilientPool extends Pool {
  private async acquireResilientClient() {
    const maxRetries = 5;
    const maxTotalWaitMs = 15000;
    const startTime = Date.now();
    let delay = 500;
    const backoffMultiplier = 2;

    for (let attempt = 1; attempt <= maxRetries; attempt++) {
      try {
        const client: PoolClient = await super.connect();
        return client;
      } catch (err: unknown) {
        const errObj =
          typeof err === "object" && err !== null
            ? (err as Record<string, unknown>)
            : undefined;
        const errorCode =
          errObj && "code" in errObj ? String(errObj.code) : undefined;

        const isTimeout =
          err instanceof Error && err.message.toLowerCase().includes("timeout");

        const isRetryable =
          (errorCode && RETRYABLE_CONNECTION_CODES.has(errorCode)) || isTimeout;

        const elapsed = Date.now() - startTime;
        const jitter = Math.floor(Math.random() * 200);
        const nextDelay = delay + jitter;

        if (
          !isRetryable ||
          attempt === maxRetries ||
          elapsed + nextDelay > maxTotalWaitMs
        ) {
          logger.error(
            `Database connection failed after attempt #${attempt} (${elapsed}ms elapsed)`,
            err instanceof Error
              ? err
              : new InternalServerErrorException(String(err)),
            { attempt, errorCode, elapsed, isRetryable },
          );
          throw err;
        }

        logger.warn(
          `Database connection attempt #${attempt} failed (${errorCode ?? "timeout"}). Serverless wakeup retry in ${nextDelay}ms`,
          null,
          { attempt, errorCode, nextDelay, elapsed },
        );

        await new Promise((resolve) => setTimeout(resolve, nextDelay));
        delay *= backoffMultiplier;
      }
    }

    const client: PoolClient = await super.connect();
    return client;
  }

  override async connect(): Promise<PoolClient>;
  override connect(callback: ConnectCallback): void;
  override connect(callback?: ConnectCallback) {
    if (callback) {
      this.acquireResilientClient()
        .then((client) => {
          callback(undefined, client, (releaseErr) =>
            client.release(
              releaseErr instanceof Error || typeof releaseErr === "boolean"
                ? releaseErr
                : undefined,
            ),
          );
        })
        .catch((err: unknown) => {
          callback(
            err instanceof Error
              ? err
              : new InternalServerErrorException(String(err)),
            undefined,
            () => {},
          );
        });
      return;
    }

    return this.acquireResilientClient();
  }
}

/**
 * Serverless-optimized PostgreSQL pool configuration.
 * - min: 0 ensures no connections remain open when traffic ceases.
 * - idleTimeoutMillis: 10000 aggressively closes sockets after 10s of inactivity.
 * - connectionTimeoutMillis: 15000 absorbs container wake-up latency before failing.
 */
export const pool = new ResilientPool({
  connectionString: env.DATABASE_URL,
  min: 0,
  max: 10,
  idleTimeoutMillis: 10000,
  connectionTimeoutMillis: 15000,
});

export const db = drizzle(pool, {
  schema,
});
```

## Background work that touches the database

Timers that query Postgres reset the idle countdown the same way an open socket does.

- Move frequent crons (every 1, 5, or 30 minutes) to once a day, for example `@Cron(CronExpression.EVERY_DAY_AT_MIDNIGHT)`, or to a Railway cron service or an external webhook.
- Delete expired rows lazily. Trigger the cleanup from a normal read or write, for example `void this.cleanupExpiredTokens()`, instead of polling.
- Do not poll a queue table (`SELECT ... FOR UPDATE SKIP LOCKED` every 5 seconds). Call something like `worker.trigger()` when work is enqueued, process until the queue is empty, then stop.
