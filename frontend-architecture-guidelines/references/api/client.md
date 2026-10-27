---
title: client
description: API implementation reference
---

```typescript
import type {
  AxiosInstance,
  AxiosRequestConfig,
  AxiosResponse,
  InternalAxiosRequestConfig,
} from "axios";
import axios, { isAxiosError } from "axios";

import { CircuitBreaker } from "@/api/circuit-breaker";
import { toApiError } from "@/api/errors";
import { tokenManager } from "@/api/token-manager";
import type { ApiResponse, PaginatedApiResponse } from "@/api/types";
import { notifyUnauthorized } from "@/api/unauthorized-handler";
import { env } from "@/lib/config/env";
import { isOnline } from "@/lib/utils/online";

const axiosInstance = axios.create({
  baseURL: env.apiBaseUrl,
  timeout: env.apiTimeoutMs,
  withCredentials: true,
  headers: {
    "Content-Type": "application/json",
  },
});

/**
 * In-flight marker used by the 401-refresh logic so a request is retried at
 * most once. Not declared on axios's public config types; read via a cast.
 */
type RetryableRequestConfig = InternalAxiosRequestConfig & { _retry?: boolean };

function isMarkedRetried(config: InternalAxiosRequestConfig): boolean {
  return (config as RetryableRequestConfig)._retry === true;
}

function markRetried(config: InternalAxiosRequestConfig): void {
  (config as RetryableRequestConfig)._retry = true;
}

/**
 * Endpoints that must never trigger a refresh-and-retry.
 *
 * A 401 from login, registration, OTP or password reset means the credentials
 * were rejected; refreshing first would replace a clear "wrong password" error
 * with a redirect to the login screen.
 */
const AUTH_PATHS = [
  "/auth/login",
  "/auth/register",
  "/auth/refresh",
  "/auth/logout",
  "/auth/password",
  "/auth/otp",
  "/auth/fayda",
] as const;

function isAuthEndpoint(url: string | undefined): boolean {
  if (!url) return false;

  // `config.url` may be absolute, may include the `api/` prefix, and — because
  // `apiClient("auth")` builds it from a relative endpoint — may have no
  // leading slash at all. Normalize all three forms to `/auth/...`.
  const path = (url.split("?")[0] ?? "")
    .replace(env.apiBaseUrl, "")
    .replace(/^\/?(?:api\/)?/, "/");

  return AUTH_PATHS.some(
    (authPath) => path === authPath || path.startsWith(`${authPath}/`),
  );
}

/** Apply a refreshed token to a config's headers. */
function applyToken(config: InternalAxiosRequestConfig, token: string): void {
  if (typeof config.headers.set === "function") {
    config.headers.set("Authorization", `Bearer ${token}`);
  } else {
    Object.assign(config.headers, { Authorization: `Bearer ${token}` });
  }
}

/**
 * Response rejection handler: 401 refresh-once + error normalization.
 *
 * Two cases deliberately do not clear the session:
 * - requests to auth endpoints, where a 401 means "bad credentials"
 * - requests made while offline, where a 401 is stale rather than fatal and
 *   cached reads should keep working (see `docs/offline.md`)
 */
async function handleResponseError(error: unknown): Promise<AxiosResponse> {
  if (isAxiosError(error) && error.response?.status === 401 && error.config) {
    const config = error.config;

    if (!isMarkedRetried(config) && !isAuthEndpoint(config.url)) {
      if (!isOnline()) {
        // No network: refreshing is impossible and the cached session is
        // still valid for offline reads. Surface the original error.
        throw toApiError(error);
      }

      markRetried(config);
      try {
        const accessToken = await tokenManager.refreshAccessToken();
        applyToken(config, accessToken);
        // Not awaited on purpose: a failure of the retried request must
        // propagate as-is, not be mistaken for a refresh failure.
        return axiosInstance(config);
      } catch {
        tokenManager.clearAccessToken();
        notifyUnauthorized();
      }
    }
  }

  throw toApiError(error);
}

// Request interceptor: inject the in-memory JWT when one is present.
axiosInstance.interceptors.request.use((config) => {
  const token = tokenManager.getAccessToken();
  if (token) {
    config.headers.set("Authorization", `Bearer ${token}`);
  }
  return config;
});

// Response interceptor: refresh 401s once, normalize every rejection.
axiosInstance.interceptors.response.use(
  (response) => response,
  (error: unknown) => handleResponseError(error),
);

export interface ApiClientOptions {
  axiosInstance?: AxiosInstance;
  circuitBreaker?: CircuitBreaker;
}

export class ApiClient<T> {
  private readonly axios: AxiosInstance;
  private readonly circuitBreaker: CircuitBreaker;
  private readonly baseEndpoint: string;

  constructor(endpoint: string, options: ApiClientOptions = {}) {
    // Normalize endpoint by removing any trailing slash so we can cleanly build URLs later
    this.baseEndpoint = endpoint.replace(/\/$/, "");
    this.axios = options.axiosInstance ?? axiosInstance;
    this.circuitBreaker = options.circuitBreaker ?? new CircuitBreaker();
  }

  /** GET collection — returns either a flat array response or a paginated response. */
  async getAll(
    params?: Record<string, unknown>,
  ): Promise<ApiResponse<T[]> | PaginatedApiResponse<T>> {
    const queryParams = { ...params };
    if (queryParams && "pageSize" in queryParams && !("limit" in queryParams)) {
      queryParams.limit = queryParams.pageSize;
      delete queryParams.pageSize;
    }
    return this.circuitBreaker.execute(async () => {
      const { data } = await this.axios.get<unknown>(this.baseEndpoint, {
        params: queryParams,
      });
      if (data && typeof data === "object" && "results" in data) {
        return data as PaginatedApiResponse<T>;
      }
      return data as ApiResponse<T[]>;
    });
  }

  /** GET detail. */
  async getById(id: number | string): Promise<ApiResponse<T>> {
    return this.circuitBreaker.execute(async () => {
      const { data } = await this.axios.get<unknown>(
        `${this.baseEndpoint}/${id}`,
      );
      return data as ApiResponse<T>;
    });
  }

  /** POST collection. */
  async create(
    payload: unknown,
    config?: AxiosRequestConfig,
  ): Promise<ApiResponse<T>> {
    return this.circuitBreaker.execute(async () => {
      const { data } = await this.axios.post<unknown>(
        this.baseEndpoint,
        payload,
        config,
      );
      return data as ApiResponse<T>;
    });
  }

  /** PUT detail — full update. */
  async put(
    id: number | string,
    payload: unknown,
    config?: AxiosRequestConfig,
  ): Promise<ApiResponse<T>> {
    return this.circuitBreaker.execute(async () => {
      const { data } = await this.axios.put<unknown>(
        `${this.baseEndpoint}/${id}`,
        payload,
        config,
      );
      return data as ApiResponse<T>;
    });
  }

  /** PATCH detail — partial update. */
  async patch(
    id: number | string,
    payload: unknown,
    config?: AxiosRequestConfig,
  ): Promise<ApiResponse<T>> {
    return this.circuitBreaker.execute(async () => {
      const { data } = await this.axios.patch<unknown>(
        `${this.baseEndpoint}/${id}`,
        payload,
        config,
      );
      return data as ApiResponse<T>;
    });
  }

  /** DELETE detail. */
  async delete(id: number | string): Promise<ApiResponse<null>> {
    return this.circuitBreaker.execute(async () => {
      await this.axios.delete(`${this.baseEndpoint}/${id}`);
      return {
        success: true,
        timestamp: new Date().toISOString(),
        path: `${this.baseEndpoint}/${id}`,
        data: null,
      } as ApiResponse<null>;
    });
  }

  /** Custom/non-CRUD endpoint (e.g. publish, submit, stats, assign_roles). */
  async action<A>(
    path: string,
    method: "get" | "post" | "patch" | "put" | "delete" = "post",
    payload?: unknown,
    config?: AxiosRequestConfig,
  ): Promise<ApiResponse<A>> {
    const cleanPath = path.replace(/^\//, "").replace(/\/$/, "");
    const url = cleanPath
      ? `${this.baseEndpoint}/${cleanPath}`
      : this.baseEndpoint;

    return this.circuitBreaker.execute(async () => {
      let response: AxiosResponse<unknown>;
      switch (method) {
        case "get":
          response = await this.axios.get<unknown>(url, config);
          break;
        case "post":
          response = await this.axios.post<unknown>(url, payload, config);
          break;
        case "patch":
          response = await this.axios.patch<unknown>(url, payload, config);
          break;
        case "put":
          response = await this.axios.put<unknown>(url, payload, config);
          break;
        case "delete":
          response = await this.axios.delete<unknown>(url, config);
          break;
      }
      return response.data as ApiResponse<A>;
    });
  }
}

/** Factory: each feature calls apiClient<Feature>(endpoint) to get a typed client. */
export function apiClient<T>(endpoint: string): ApiClient<T> {
  return new ApiClient<T>(endpoint);
}

```
