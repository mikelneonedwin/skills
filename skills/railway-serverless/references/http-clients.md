# Inter-service HTTP on Railway serverless

When an upstream service sleeps, incoming requests meet a cold start taking 2 to 8 seconds. During that time, Railway's proxy either drops connections (`ECONNREFUSED`, `ECONNRESET`, `ETIMEDOUT`) or returns HTTP `502` or `503`. Callers must absorb this latency without multiplying timeouts or duplicating mutating requests.

## Contents

- Critical rules
- Native fetch implementation
- Axios / @nestjs/axios implementation

## Critical rules

1. **Bounded overall deadline**: Enforce a single total timeout (e.g. 25 seconds) bounding all retries, backoff delays, and network execution combined. Dynamically adjust each attempt's timeout to the remaining budget.
2. **Strict idempotency guardrails**: Never retry non-idempotent operations (`POST`, `PATCH`) on socket drops, timeouts, or 502/503 responses unless an explicit idempotency key (`Idempotency-Key` or `X-Idempotency-Key`) is present. An upstream application may have processed and committed a transaction before encountering an error.
3. **Transient error classification**: Inspect error codes (including nested `cause.code`) to verify transient network failures (`ECONNREFUSED`, `ECONNRESET`, `ETIMEDOUT`, `EHOSTUNREACH`, `ENETUNREACH`, `UND_ERR_CONNECT_TIMEOUT`). Fail fast on client-side errors like `TypeError` or invalid URLs.
4. **Preserve caller cancellations**: Merge internal timeout abort controllers with caller-supplied `AbortSignal` instances using `AbortSignal.any` or a robust event-listener fallback.

## Native fetch implementation

```ts
export interface FetchRetryOptions extends RequestInit {
  retries?: number;
  backoffMs?: number;
  maxBackoffMs?: number;
  totalTimeoutMs?: number;
}

const IDEMPOTENT_METHODS = new Set(["GET", "HEAD", "OPTIONS", "PUT", "DELETE"]);
const TRANSIENT_NETWORK_CODES = new Set([
  "ECONNREFUSED",
  "ECONNRESET",
  "ETIMEDOUT",
  "EHOSTUNREACH",
  "ENETUNREACH",
  "UND_ERR_CONNECT_TIMEOUT",
  "UND_ERR_SOCKET",
]);

function isTransientNetworkError(err: any): boolean {
  if (!err) return false;
  const code = err.code ?? err.cause?.code;
  if (code && TRANSIENT_NETWORK_CODES.has(code)) return true;
  const msg = String(err.message ?? "").toLowerCase();
  return msg.includes("fetch failed") || msg.includes("network error") || msg.includes("socket");
}

function hasIdempotencyKey(headers?: HeadersInit): boolean {
  if (!headers) return false;
  if (headers instanceof Headers) {
    return headers.has("idempotency-key") || headers.has("x-idempotency-key");
  }
  if (Array.isArray(headers)) {
    return headers.some(([key]) => {
      const lower = key.toLowerCase();
      return lower === "idempotency-key" || lower === "x-idempotency-key";
    });
  }
  return Object.keys(headers).some((key) => {
    const lower = key.toLowerCase();
    return lower === "idempotency-key" || lower === "x-idempotency-key";
  });
}

function combineSignals(signals: (AbortSignal | undefined | null)[]): {
  signal: AbortSignal;
  cleanup: () => void;
} {
  const activeSignals = signals.filter(Boolean) as AbortSignal[];
  if (typeof AbortSignal !== "undefined" && typeof (AbortSignal as any).any === "function") {
    return {
      signal: (AbortSignal as any).any(activeSignals),
      cleanup: () => {},
    };
  }

  const controller = new AbortController();
  const onAbort = (evt: Event) => {
    controller.abort((evt.target as AbortSignal).reason);
    cleanup();
  };

  const cleanup = () => {
    activeSignals.forEach((sig) => sig.removeEventListener("abort", onAbort));
  };

  for (const sig of activeSignals) {
    if (sig.aborted) {
      controller.abort(sig.reason);
      return { signal: controller.signal, cleanup: () => {} };
    }
    sig.addEventListener("abort", onAbort, { once: true });
  }

  return { signal: controller.signal, cleanup };
}

export async function fetchWithRetry(
  url: string | URL,
  options: FetchRetryOptions = {},
): Promise<Response> {
  const {
    retries = 3,
    backoffMs = 1000,
    maxBackoffMs = 4000,
    totalTimeoutMs = 25000,
    signal: callerSignal,
    ...fetchInit
  } = options;

  const method = (fetchInit.method ?? "GET").toUpperCase();
  const isIdempotent = IDEMPOTENT_METHODS.has(method) || hasIdempotencyKey(fetchInit.headers);
  const startTime = Date.now();
  let attempt = 0;

  while (true) {
    attempt++;
    const elapsed = Date.now() - startTime;
    const remainingTime = totalTimeoutMs - elapsed;

    if (remainingTime <= 0) {
      throw new Error(`fetchWithRetry: total timeout of ${totalTimeoutMs}ms exceeded`);
    }

    const timeoutController = new AbortController();
    const timeoutTimer = setTimeout(() => {
      timeoutController.abort(new Error(`Attempt timed out (${remainingTime}ms remaining)`));
    }, remainingTime);

    const { signal: combinedSignal, cleanup: cleanupSignals } = combineSignals([
      timeoutController.signal,
      callerSignal,
    ]);

    try {
      const response = await fetch(url, {
        ...fetchInit,
        signal: combinedSignal,
      });

      const isColdStatus = response.status === 502 || response.status === 503;
      if (isColdStatus && isIdempotent && attempt <= retries) {
        throw new Error(`Upstream cold start HTTP ${response.status}`);
      }

      return response;
    } catch (err: any) {
      if (callerSignal?.aborted) {
        throw err;
      }

      const isColdStatus =
        err?.message?.includes("HTTP 502") || err?.message?.includes("HTTP 503");
      const isTransient = isColdStatus || isTransientNetworkError(err);

      if (!isTransient || !isIdempotent || attempt > retries) {
        throw err;
      }

      const delay =
        Math.min(backoffMs * Math.pow(2, attempt - 1), maxBackoffMs) +
        Math.random() * 200;

      if (Date.now() - startTime + delay >= totalTimeoutMs) {
        throw err;
      }

      await new Promise((resolve) => setTimeout(resolve, delay));
    } finally {
      clearTimeout(timeoutTimer);
      cleanupSignals();
    }
  }
}
```

## Axios / @nestjs/axios implementation

```ts
import { Module, OnModuleInit, Logger } from "@nestjs/common";
import { HttpModule, HttpService } from "@nestjs/axios";
import type { AxiosError, InternalAxiosRequestConfig } from "axios";

interface RetryAxiosRequestConfig extends InternalAxiosRequestConfig {
  __retryCount?: number;
  __startTime?: number;
  __totalTimeoutMs?: number;
}

const IDEMPOTENT_METHODS = new Set(["get", "head", "options", "put", "delete"]);

function getHeaderCaseInsensitive(config: InternalAxiosRequestConfig, name: string): string | undefined {
  if (!config.headers) return undefined;
  if (typeof config.headers.get === "function") {
    const val = config.headers.get(name);
    return val ? String(val) : undefined;
  }
  const target = name.toLowerCase();
  for (const [key, value] of Object.entries(config.headers)) {
    if (key.toLowerCase() === target) {
      return value ? String(value) : undefined;
    }
  }
  return undefined;
}

@Module({
  imports: [
    HttpModule.register({
      maxRedirects: 5,
    }),
  ],
  exports: [HttpModule],
})
export class ResilientHttpModule implements OnModuleInit {
  private readonly logger = new Logger(ResilientHttpModule.name);

  constructor(private readonly httpService: HttpService) {}

  onModuleInit() {
    const axiosInstance = this.httpService.axiosRef;

    axiosInstance.interceptors.request.use((config: RetryAxiosRequestConfig) => {
      config.__startTime = config.__startTime ?? Date.now();
      config.__totalTimeoutMs = config.__totalTimeoutMs ?? 25000;

      const elapsed = Date.now() - config.__startTime;
      const remainingTime = config.__totalTimeoutMs - elapsed;

      // Adjust individual attempt timeout to exact remaining budget
      config.timeout = Math.max(1, remainingTime);
      return config;
    });

    axiosInstance.interceptors.response.use(
      (response) => response,
      async (error: AxiosError) => {
        const config = error.config as RetryAxiosRequestConfig | undefined;
        if (!config || !config.__startTime || !config.__totalTimeoutMs) {
          return Promise.reject(error);
        }

        config.__retryCount = config.__retryCount ?? 0;
        const maxRetries = 3;
        const status = error.response?.status;
        const isColdStartStatus = status === 502 || status === 503;
        const isNetworkError =
          !error.response &&
          (error.code === "ECONNREFUSED" ||
            error.code === "ECONNRESET" ||
            error.code === "ETIMEDOUT" ||
            error.message.includes("timeout"));

        const method = (config.method ?? "get").toLowerCase();
        const hasIdempotencyKey = Boolean(
          getHeaderCaseInsensitive(config, "idempotency-key") ||
            getHeaderCaseInsensitive(config, "x-idempotency-key"),
        );
        const isIdempotent = IDEMPOTENT_METHODS.has(method) || hasIdempotencyKey;

        if (!isIdempotent) {
          return Promise.reject(error);
        }

        const elapsed = Date.now() - config.__startTime;
        if ((isColdStartStatus || isNetworkError) && config.__retryCount < maxRetries) {
          config.__retryCount += 1;
          const delay =
            Math.min(1000 * Math.pow(2, config.__retryCount - 1), 4000) +
            Math.random() * 200;

          if (elapsed + delay >= config.__totalTimeoutMs) {
            return Promise.reject(error);
          }

          this.logger.debug?.(
            `[HTTP] Upstream waking up (${status || error.code}). Retry #${config.__retryCount} in ${Math.round(delay)}ms`,
          );

          await new Promise((resolve) => setTimeout(resolve, delay));
          return axiosInstance(config);
        }

        return Promise.reject(error);
      },
    );
  }
}
```
