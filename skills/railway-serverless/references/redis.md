# Redis (`ioredis`) on Railway serverless

Redis sleeps only when no socket to it is open and nothing is retrying in the background.

## Contents

- Rules with reasons
- Reference implementation
- Using the wrapper

## Rules with reasons

1. **`lazyConnect: true`.** A client that connects at import wakes Redis whenever the container starts, even when the request that woke the container never touches Redis. It also throws unhandled rejections if Redis is asleep at that moment.
2. **`enableOfflineQueue: false` and `maxRetriesPerRequest: 1` or `2`.** By default `ioredis` buffers commands in memory while disconnected, so a request can hang with no error while Redis wakes.
3. **One shared `connectPromise`.** Calling `connect()` while already connecting throws "Redis is already connecting/connected". Sending commands while `status === "connecting"` with the offline queue off throws "Stream isn't writeable and enableOfflineQueue options is false". Concurrent callers must all await the same promise. Do not null `connectPromise` in `closeConnection()`. The connect attempt's own `finally` does that, which avoids a race when a disconnect lands mid-connect.
4. **`retryStrategy` for wake-up, not `reconnectOnError`.** While Redis starts, Railway's proxy drops connections (`ECONNREFUSED`, `ECONNRESET`, `ETIMEDOUT`) for 2 to 5 seconds. Socket drops go to `retryStrategy`. `reconnectOnError` only runs for command-level error replies such as `READONLY`. Use 6 to 8 attempts from 250 ms up to 3000 ms with jitter, which gives an 8 to 10 second ceiling.
5. **`disconnect()` after 5 to 15 seconds idle, never `quit()`.** `quit()` sends `QUIT` and waits for `OK`, which stalls on a half-closed proxy and can leave the socket open. `disconnect()` destroys the stream and cancels reconnect timers. Call `timer.unref()` so the idle timer does not hold the process open.
6. **Disconnect failed clients.** After a failed `connect()`, call `failedClient.disconnect()`. Setting the variable to `null` leaves the instance alive in memory, still retrying and sending SYN packets across the private network.
7. **`keepAlive: 0` and `autoResendUnfulfilledCommands: false`.** Keep-alive probes are packets and reset Railway's counter.
8. **No `subscribe()`.** Subscribers need a permanent socket and cannot scale to zero. Use ephemeral polling or webhooks.

## Reference implementation

Framework-agnostic apart from the NestJS decorators. For Express, Fastify, or plain TypeScript, drop `@Injectable()` and `OnModuleDestroy`, replace `Logger` with the project's logger, and call `closeConnection()` on shutdown.

```ts
import { Injectable, OnModuleDestroy, Logger } from "@nestjs/common";
import { Redis } from "ioredis";

@Injectable()
export class ServerlessRedisService implements OnModuleDestroy {
  private readonly logger = new Logger(ServerlessRedisService.name);
  private client: Redis | null = null;
  private connectPromise: Promise<Redis | null> | null = null;
  private disconnectTimer: ReturnType<typeof setTimeout> | null = null;
  private readonly IDLE_TIMEOUT_MS = 10000; // Close socket after 10s of zero activity

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
    // Note: Do NOT clear this.connectPromise here. Let its own `finally` block
    // clear it so an in-flight connection cannot race with a new attempt.
    if (this.client) {
      const clientRef = this.client;
      this.client = null;
      // Rule #5: Forcefully tear down the TCP socket immediately
      clientRef.disconnect();
    }
  }

  private scheduleDisconnect() {
    this.clearDisconnectTimer();
    this.disconnectTimer = setTimeout(() => {
      this.logger.log("Closing idle Redis connection to allow Railway scale-to-0");
      this.closeConnection();
    }, this.IDLE_TIMEOUT_MS);
    // Crucial: Unref timer so it does not keep the Node process alive
    this.disconnectTimer.unref();
  }

  /**
   * Returns a ready Redis client instance, establishing connection if idle.
   * Concurrency-safe: Deduplicates in-flight connection attempts across requests.
   */
  private async getClient(): Promise<Redis | null> {
    this.clearDisconnectTimer();

    // 1. Create client instance if not present
    if (!this.client) {
      this.client = new Redis(process.env.REDIS_URL!, {
        lazyConnect: true,          // Rule #1: Never connect on instantiation
        enableOfflineQueue: false,  // Rule #2: Fail fast rather than hanging
        maxRetriesPerRequest: 2,
        connectTimeout: 5000,
        keepAlive: 0,               // Rule #7: Disable idle keepalive probes
        autoResendUnfulfilledCommands: false,
        retryStrategy: (times) => { // Rule #4: Absorb cold-start wake-up latency
          const maxRetries = 8;
          if (times > maxRetries) {
            this.logger.error(`[REDIS] Exceeded max connection retries (${maxRetries}).`);
            return null; // Stop retrying and trigger connect rejection
          }
          const delay = Math.min(250 * Math.pow(2, times - 1), 3000);
          const jitter = Math.floor(Math.random() * 150);
          return delay + jitter;
        },
        // Rule #4: reconnectOnError is reserved solely for application-level Redis errors (like replica failovers)
        reconnectOnError: (err) => err.message.includes("READONLY"),
      });

      this.client.on("error", (err) => {
        this.logger.error("[REDIS] Socket error:", err.message);
      });
    }

    // 2. Return immediately if ready
    if (this.client.status === "ready") {
      return this.client;
    }

    // 3. Rule #3: Deduplicate in-flight connection promises across concurrent callers
    if (!this.connectPromise) {
      this.connectPromise = (async () => {
        try {
          if (!this.client) return null;
          await this.client.connect();
          return this.client;
        } catch (error) {
          this.logger.error("[REDIS] Failed to connect:", error);
          // Rule #6: Explicitly disconnect failed instance so it does not retry in background
          const failedClient = this.client;
          this.client = null;
          failedClient?.disconnect();
          return null;
        } finally {
          // Promise cleans itself up when finished
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
    try {
      const client = await this.getClient();
      if (!client) {
        throw new Error("Redis client connection unavailable");
      }
      return await operation(client);
    } finally {
      // Schedule socket closure once the operation completes
      this.scheduleDisconnect();
    }
  }
}
```

## Using the wrapper

Send every Redis command through `execute()` so the idle timer restarts after each operation.

```ts
const value = await redisService.execute((client) => client.get("session:123"));
```

`execute()` returns `null` when Redis is unavailable after retries, and callers should handle that case. Decide per call whether a cache miss, a skipped rate limit, or an error is the right outcome.
