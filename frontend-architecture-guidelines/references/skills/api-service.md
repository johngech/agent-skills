---
name: project-api-service
description: How API services are built in this project. Use when creating or modifying any feature-specific API integration (e.g., api.md). Details the apiClient factory pattern, common CRUD methods, and custom actions.
---

# API Service Architecture

This project uses a centralized, typed Axios client setup. Feature-specific API services must NEVER create their own Axios instances or configure raw Axios methods directly. Instead, they must use the `apiClient` factory from `@/api/client`.

## The `apiClient` Factory Pattern

The core pattern involves creating a typed client instance for a specific endpoint prefix, and then exporting a service object that wraps its methods.

### 1. Basic Setup

Always import `apiClient` from the shared `api/client.ts`. Pass the entity type to it, and specify the base endpoint (e.g., `news`, `users`).

```typescript
// app/(dashboard)/{featureName}/api.ts
import { apiClient } from "@/api/client";
import type { FeatureEntity, CreateFeatureInput, UpdateFeatureInput } from "./types";

// 1. Create the base client instance
const featureClient = apiClient<FeatureEntity>("feature-endpoint");
```

### 2. Exporting the API Service

Export an object (e.g., `featureApi`) that exposes the necessary operations. The `apiClient` already provides typed, standardized wrappers for standard CRUD operations:

- `getAll(params?)` - Returns `FetchApiResponse<T[]>` with pagination support.
- `getById(id)` - Returns `FetchApiResponse<T>`.
- `create(payload)` - Returns `FetchApiResponse<T>`.
- `patch(id, payload)` - Returns `FetchApiResponse<T>`.
- `put(id, payload)` - Returns `FetchApiResponse<T>`.
- `delete(id)` - Returns `FetchApiResponse<null>`.

```typescript
export const featureApi = {
  // Pass through pagination/filter parameters
  getAll: (params?: Record<string, unknown>) => featureClient.getAll(params),
  
  getById: (id: number) => featureClient.getById(id),
  
  create: (data: CreateFeatureInput) => featureClient.create(data),
  
  patch: (id: number, data: UpdateFeatureInput) => featureClient.patch(id, data),
  
  delete: (id: number) => featureClient.delete(id),
```

### 3. Custom Actions

For non-CRUD operations (like publishing, assigning roles, or custom actions), use the `action<T>` method provided by the client. It handles the URL resolution, typing, and standard response formatting.

```typescript
  // Example of a custom POST action
  assignRoles: (id: number, roles: string[]) => 
    featureClient.action<FeatureEntity>(`${id}/assign-roles`, "post", { roles }),

  // Example of a custom PATCH action
  publish: (id: number) => 
    featureClient.action<FeatureEntity>(`${id}/publish`, "patch"),
};
```

## Golden Rules for API Services

1. **No Raw Axios Configs**: Never import `axios` directly in feature modules. The global `apiClient` handles authentication, 401 retries, circuit breaking, and error normalization.
2. **Type Everything**: Always define and use TypeScript interfaces/types (`CreateInput`, `UpdateInput`, `EntityModel`) in a separate `types.ts` file, and pass the response type to `apiClient<T>`.
3. **Keep it Pure**: The API layer should only handle network requests. It should not contain UI state or React hooks. State management belongs in TanStack Query (see `project-query-options` skill).

## Core Implementation References

The underlying infrastructure that powers this API pattern relies on the following core files. Review these files for a deep dive into the circuit breaker patterns, interceptors, and error handling:

- **Client Implementation:** [client.ts](../api/client.md) (Centralized Axios setup, 401 interceptors, and `ApiClient` class)
- **Circuit Breaker:** [circuit-breaker.ts](../api/circuit-breaker.md) (Resiliency pattern preventing cascading failures)
- **Error Normalization:** [errors.ts](../api/errors.md) (Extracts user-friendly error messages from generic API errors)
- **Token Management:** [token-manager.ts](../api/token-manager.md) (Manages token lifecycle and refresh logic)
- **Global API Types:** [types.ts](../api/types.md) (Base types for `PaginatedResponse`, `ApiError`, etc.)
