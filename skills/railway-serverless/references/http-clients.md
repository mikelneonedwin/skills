# Inter-service HTTP on Railway serverless

When an upstream service sleeps, the first request meets a cold start. Starting the container, booting Node, NestJS, or Next.js, and running init hooks takes 2 to 8 seconds. During that time Railway's router either drops the connection (`ECONNREFUSED`, `ECONNRESET`, `ETIMEDOUT`) or returns `502` or `503`. A client with a 3 to 5 second timeout and no retries passes that failure straight to the end user.

## Contents

- Rules
- Native fetch
- Axios and @nestjs/axios
- ky

## Rules

1. **Timeout of 20 to 25 seconds** on internal calls. A shorter timeout aborts requests just before the upstream finishes booting.
2. **Retry up to 3 times** on `502`, `503`, `ECONNREFUSED`, `ECONNRESET`, and `ETIMEDOUT`. Back off 1s, 2s, 4s with about 200 ms of jitter. The second or third attempt usually succeeds without the caller noticing.

Retrying non-idempotent requests (POST that creates something) can duplicate work if the first attempt reached the upstream. 502 and 503 from the router mean the request did not reach the app, so retrying them is safe. For timeouts on non-idempotent calls, add an idempotency key or limit retries to the status-code cases.

## Native fetch

```ts
export interface FetchRetryOptions extends RequestInit {
  retries?: number;        // default: 3
  backoffMs?: number;      // default: 1000
  maxBackoffMs?: number;   // default: 4000
  timeoutMs?: number;      // default: 20000 (absorb cold start)
}

export async function fetchWithRetry(
  url: string | URL,
  options: FetchRetryOptions = {},
): Promise<Response> {
  const {
    retries = 3,
    backoffMs = 1000,
    maxBackoffMs = 4000,
    timeoutMs = 20000,
    ...fetchInit
  } = options;

  let attempt = 0;
  while (true) {
    attempt++;
    const controller = new AbortController();
    const timer = setTimeout(() => controller.abort(), timeoutMs);

    try {
      const response = await fetch(url, {
        ...fetchInit,
        signal: controller.signal,
      });

      if ((response.status === 502 || response.status === 503) && attempt <= retries) {
        throw new Error(`Cold-start HTTP ${response.status}`);
      }

      return response;
    } catch (err: any) {
      if (attempt > retries) {
        throw err;
      }
      const delay =
        Math.min(backoffMs * Math.pow(2, attempt - 1), maxBackoffMs) +
        Math.random() * 200;
      await new Promise((resolve) => setTimeout(resolve, delay));
    } finally {
      clearTimeout(timer);
    }
  }
}
```

## Axios and @nestjs/axios

```ts
import { Module, OnModuleInit, Logger } from '@nestjs/common';
import { HttpModule, HttpService } from '@nestjs/axios';
import type { AxiosError, InternalAxiosRequestConfig } from 'axios';

interface RetryAxiosRequestConfig extends InternalAxiosRequestConfig {
  __retryCount?: number;
}

@Module({
  imports: [
    HttpModule.register({
      timeout: 25000, // 25s timeout to absorb cold-start latency
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

    axiosInstance.interceptors.response.use(
      (response) => response,
      async (error: AxiosError) => {
        const config = error.config as RetryAxiosRequestConfig | undefined;
        if (!config) return Promise.reject(error);

        config.__retryCount = config.__retryCount ?? 0;
        const maxRetries = 3;
        const status = error.response?.status;
        const isColdStartStatus = status === 502 || status === 503;
        const isNetworkError =
          !error.response &&
          (error.code === 'ECONNREFUSED' ||
            error.code === 'ECONNRESET' ||
            error.code === 'ETIMEDOUT' ||
            error.message.includes('timeout'));

        if ((isColdStartStatus || isNetworkError) && config.__retryCount < maxRetries) {
          config.__retryCount += 1;
          const delay =
            Math.min(1000 * Math.pow(2, config.__retryCount - 1), 4000) +
            Math.random() * 200;

          await new Promise((resolve) => setTimeout(resolve, delay));
          return axiosInstance.request(config);
        }

        return Promise.reject(error);
      },
    );
  }
}
```

## ky

```ts
import ky from 'ky';

export const apiClient = ky.create({
  timeout: 25000,
  retry: {
    limit: 3,
    statusCodes: [502, 503],
    methods: ['get', 'post', 'put', 'delete', 'patch', 'head'],
    backoffLimit: 4000,
    delay: (attemptCount) =>
      Math.min(1000 * Math.pow(2, attemptCount - 1), 4000) + Math.random() * 200,
  },
});
```
