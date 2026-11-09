# Redis on Railway serverless

Covers `ioredis` in scale-to-0 environments. Both the application and the Redis container can sleep, but only if all TCP keep-alives and background reconnections are completely severed when idle.

## Contents

- Why default Redis clients prevent sleeping
- The critical rules for `ioredis`
- Reference counted idle disconnection
- Resilient cold-start acquisition loop
- Production reference implementation

## Why default Redis clients prevent sleeping

By default, `ioredis` establishes a persistent TCP socket upon instantiation, enables offline command buffering, and uses background reconnect loops. If an idle application keeps an `ioredis` instance open without tearing down the underlying stream, TCP keep-alives and socket probes generate outbound network traffic that resets Railway's 5 to 10 minute inactivity countdown.

## The critical rules for `ioredis`

1. **`lazyConnect: true`**: Never connect on module evaluation. Connecting at bootstrap wakes Redis when unnecessary and crashes if Redis is currently asleep.
2. **`enableOfflineQueue: false`**: Fail fast rather than buffering commands indefinitely in memory while disconnected.
3. **Explicitly disable TCP keep-alive**: In `ioredis`, passing `keepAlive: 0` enables keep-alive probes with Node's default interval, while omitting it can fall back to client defaults (e.g. 30s). To explicitly disable TCP keep-alive on the underlying stream, pass `keepAlive: false as any`.
4. **Reference-counted idle disconnect**: Track active operations with an in-flight counter. The disconnect timer must only be armed when active operations reach zero, preventing a finished operation from killing the connection while another concurrent operation is still in flight.
5. **In-flight connection deduplication**: Wrap connection attempts in a single shared Promise (`connectPromise`) so concurrent requests share one attempt.
6. **Explicit cold-start retry loop**: In `ioredis` with `lazyConnect: true`, calling `connect()` returns a promise that rejects immediately upon initial failure (emitting `ECONNREFUSED` and transitioning status to `reconnecting` or `end`). Wrapping `connect()` in an explicit cold-start retry loop that retries connection establishment over the wake-up window ensures that an initial socket rejection does not cause the application to discard the client prematurely.
7. **Quiet logging during wake-up**: Suppress error event listeners during reconnect attempts so that initial connection refusals do not trigger false positive crash alerts in monitoring tools.

## Production reference implementation

```ts
import { Injectable, Logger, OnModuleDestroy } from "@nestjs/common";
import { Redis } from "ioredis";

@Injectable()
export class ServerlessRedisService implements OnModuleDestroy {
  private readonly logger = new Logger(ServerlessRedisService.name);
  private client: Redis | null = null;
  private connectPromise: Promise<Redis | null> | null = null;
  private disconnectTimer: ReturnType<typeof setTimeout> | null = null;
  private activeOperations = 0;
  private readonly IDLE_TIMEOUT_MS = 10000;
  private readonly COLD_START_TIMEOUT_MS = 20000;

  onModuleDestroy() {
    this.closeConnection();
  }

  private clearDisconnectTimer() {
    if (this.disconnectTimer) {
      clearTimeout(this.disconnectTimer);
      this.disconnectTimer = null;
    }
  }

  private closeConnection() {
    this.clearDisconnectTimer();
    if (this.client) {
      const clientRef = this.client;
      this.client = null;
      // Forcefully tear down socket; do not use quit() which waits on an unreplied handshake
      clientRef.disconnect();
    }
  }

  private scheduleDisconnectIfIdle() {
    this.clearDisconnectTimer();
    if (this.activeOperations === 0 && this.client) {
      this.disconnectTimer = setTimeout(() => {
        if (this.activeOperations === 0) {
          this.closeConnection();
        }
      }, this.IDLE_TIMEOUT_MS);
      this.disconnectTimer.unref(); // Does not hold the Node.js event loop open
    }
  }

  private createClientInstance(remainingMs: number): Redis {
    const instance = new Redis(process.env.REDIS_URL!, {
      lazyConnect: true,
      enableOfflineQueue: false,
      maxRetriesPerRequest: 2,
      connectTimeout: Math.min(5000, Math.max(1000, remainingMs)),
      // Explicitly disable TCP keep-alive on the underlying stream
      keepAlive: false as any,
      autoResendUnfulfilledCommands: false,
      retryStrategy: null, // Cold-start retries handled by acquireResilientClient
      reconnectOnError: (err) => err.message.includes("READONLY"),
    });

    // Silence initial socket errors during cold start wake-up
    instance.on("error", (err) => {
      this.logger.debug?.(`[REDIS] Background socket status: ${err.message}`);
    });

    return instance;
  }

  private async acquireResilientClient(): Promise<Redis | null> {
    const startTime = Date.now();
    let delay = 300;
    const maxRetries = 6;

    for (let attempt = 1; attempt <= maxRetries; attempt++) {
      const elapsed = Date.now() - startTime;
      const remainingTime = this.COLD_START_TIMEOUT_MS - elapsed;

      if (remainingTime <= 0) {
        break;
      }

      const client = this.createClientInstance(remainingTime);

      try {
        await client.connect();
        this.client = client;
        return client;
      } catch (err) {
        client.disconnect();

        const currentElapsed = Date.now() - startTime;
        const jitter = Math.floor(Math.random() * 150);
        const nextDelay = delay + jitter;

        if (attempt === maxRetries || currentElapsed + nextDelay >= this.COLD_START_TIMEOUT_MS) {
          this.logger.error(
            `[REDIS] Could not connect to Redis after attempt #${attempt} (${currentElapsed}ms elapsed): ${
              err instanceof Error ? err.message : String(err)
            }`,
          );
          return null;
        }

        this.logger.debug?.(
          `[REDIS] Redis waking up (attempt #${attempt}). Retrying in ${nextDelay}ms`,
        );

        await new Promise((resolve) => setTimeout(resolve, nextDelay));
        delay = Math.min(delay * 1.8, 3000);
      }
    }

    this.logger.error(
      `[REDIS] Failed to establish Redis connection within ${this.COLD_START_TIMEOUT_MS}ms budget`,
    );
    return null;
  }

  private async getClient(): Promise<Redis | null> {
    this.clearDisconnectTimer();

    if (this.client && this.client.status === "ready") {
      return this.client;
    }

    if (!this.connectPromise) {
      this.connectPromise = (async () => {
        try {
          return await this.acquireResilientClient();
        } finally {
          this.connectPromise = null;
        }
      })();
    }

    return this.connectPromise;
  }

  /**
   * Universal Execution Wrapper.
   * Ensures the connection is acquired safely and schedules idle disconnect upon completion.
   */
  async execute<T>(operation: (client: Redis) => Promise<T>): Promise<T | null> {
    this.activeOperations++;
    this.clearDisconnectTimer();
    try {
      const client = await this.getClient();
      if (!client) {
        throw new Error("Redis client connection unavailable");
      }
      return await operation(client);
    } finally {
      this.activeOperations = Math.max(0, this.activeOperations - 1);
      this.scheduleDisconnectIfIdle();
    }
  }
}
```
