# Postgres on Railway serverless

Covers `pg.Pool`, `postgres.js`, Drizzle, and Better Auth. Both the API and the database can sleep, and both only sleep once every socket between them is closed.

## Contents

- Why default pools keep services awake
- Pool settings and defaults
- The callback bypass trap
- Preventing connection leaks on timeouts
- Reference implementation (`pg.Pool` + Drizzle)
- Reference implementation (`postgres.js`)

## Why default pools keep services awake

A pool configured with a non-zero minimum or with no idle timeout retains open TCP sockets between the application and PostgreSQL. Background status queries and TCP keep-alive probes exchanged across those sockets generate outbound network packets, preventing both the application and database containers from sleeping.

## Pool settings and defaults

In `node-postgres` (`pg`), defaults already specify `min: 0`, `idleTimeoutMillis: 10000`, and `max: 10`. However, `connectionTimeoutMillis` defaults to `0` (indefinite hang).

| Setting | Value | Reason |
|---|---|---|
| `min` | `0` | The pool opens no connections when idle. |
| `idleTimeoutMillis` | `10000` | Idle clients are destroyed after 10 seconds of inactivity. |
| `connectionTimeoutMillis` | `15000` | Waking a PostgreSQL container takes 5 to 8 seconds. A short timeout aborts prematurely; setting zero hangs indefinitely. |
| `max` | `10` | Caps connections per replica so traffic bursts after wake-up cannot exhaust `max_connections`. |

Add an explicit retry loop with exponential backoff and jitter, capping the overall duration across all retries at 25 seconds.

## The callback bypass trap

`pg` exposes `pool.connect()` in two forms: promise (`await pool.connect()`) and callback (`pool.connect((err, client, done) => ...)`). Drizzle and Better Auth run queries through `pool.query()`, and inside `pg`, `pool.query()` always acquires its client using the callback form: `this.connect(cb)`.

If a custom pool adds retries only to the promise form and delegates the callback form directly to `super.connect(cb)`, every ORM query skips the retry loop and immediately fails with `ETIMEDOUT` when the database is asleep.

Both branches must route through `acquireResilientClient`, passing a release handler that invokes `client.release()`.

## Preventing connection leaks on timeouts

Never use a naive `Promise.race([super.connect(), timeout])`. If the timeout fires first, a late-arriving connection from `super.connect()` will resolve into limbo with no caller holding its reference, permanently leaking the client and exhausting the pool. Any client arriving after a timeout must be immediately released or destroyed.

## Reference implementation (`pg.Pool` + Drizzle)

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

export class ResilientPool extends Pool {
  private readonly maxTotalWaitMs = 25000;

  constructor(config?: any) {
    super({
      ...config,
      min: 0,
      max: config?.max ?? 10,
      idleTimeoutMillis: config?.idleTimeoutMillis ?? 10000,
      connectionTimeoutMillis: config?.connectionTimeoutMillis ?? 15000,
    });

    // Suppress noisy uncaught socket drops on idle pool connections
    this.on("error", (err: any) => {
      logger.debug?.(`Idle client socket closed: ${err?.message || err}`);
    });
  }

  private connectWithBudget(timeoutMs: number): Promise<PoolClient> {
    return new Promise<PoolClient>((resolve, reject) => {
      let isSettled = false;
      let timer: ReturnType<typeof setTimeout> | null = null;

      timer = setTimeout(() => {
        if (isSettled) return;
        isSettled = true;
        reject(new Error(`Database connect timeout (${timeoutMs}ms budget exceeded)`));
      }, timeoutMs);

      super.connect()
        .then((client) => {
          if (timer) clearTimeout(timer);
          if (isSettled) {
            // Late connection arrived after timeout; release immediately to avoid leak
            client.release(true);
            return;
          }
          isSettled = true;
          resolve(client);
        })
        .catch((err) => {
          if (timer) clearTimeout(timer);
          if (!isSettled) {
            isSettled = true;
            reject(err);
          }
        });
    });
  }

  private async acquireResilientClient(): Promise<PoolClient> {
    const maxRetries = 6;
    const startTime = Date.now();
    let delay = 300;
    const backoffMultiplier = 1.8;

    for (let attempt = 1; attempt <= maxRetries; attempt++) {
      const elapsed = Date.now() - startTime;
      const remainingBudget = this.maxTotalWaitMs - elapsed;

      if (remainingBudget <= 0) {
        break;
      }

      const attemptTimeout = Math.min(remainingBudget, 15000);

      try {
        const client = await this.connectWithBudget(attemptTimeout);
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
        const currentElapsed = Date.now() - startTime;
        const jitter = Math.floor(Math.random() * 150);
        const nextDelay = delay + jitter;

        if (
          !isRetryable ||
          attempt === maxRetries ||
          currentElapsed + nextDelay >= this.maxTotalWaitMs
        ) {
          logger.error(
            `Database connection failed after attempt #${attempt} (${currentElapsed}ms elapsed)`,
            err instanceof Error
              ? err
              : new InternalServerErrorException(String(err)),
          );
          throw err;
        }

        // Silent debug log during normal cold-start retries
        logger.debug?.(
          `Database waking up (attempt #${attempt}, code: ${errorCode ?? "timeout"}). Retrying in ${nextDelay}ms`,
        );

        await new Promise((resolve) => setTimeout(resolve, nextDelay));
        delay *= backoffMultiplier;
      }
    }

    throw new InternalServerErrorException(
      `Database connection timed out after exceeding ${this.maxTotalWaitMs}ms total budget`,
    );
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
        .catch((err) => {
          callback(err instanceof Error ? err : new Error(String(err)), undefined, () => {});
        });
      return;
    }

    return this.acquireResilientClient();
  }
}

export const pool = new ResilientPool({
  connectionString: env.DATABASE_URL,
  min: 0,
  max: 10,
  idleTimeoutMillis: 10000,
  connectionTimeoutMillis: 15000,
});

export const db = drizzle(pool, { schema });
```

## Reference implementation (`postgres.js`)

```ts
import postgres from "postgres";

export function createResilientPostgres(connectionString: string) {
  return postgres(connectionString, {
    max: 10,
    idle_timeout: 10, // seconds
    connect_timeout: 25, // seconds to absorb container wake-up
    onnotice: () => {}, // silence notice spam
    connection: {
      application_name: "serverless_app",
    },
  });
}
```
