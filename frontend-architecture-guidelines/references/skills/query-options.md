---
name: project-query-options
description: How TanStack Query (React Query) is used in this project. Use when creating or modifying queries, loaders, or mutations in queries.ts. Enforces query options pattern and typed query keys.
---

# TanStack Query Patterns

This project uses TanStack Query v5 with the **query-options pattern**. This approach ensures type safety and allows queries to be easily shared between React components (`useQuery`) and React Router loaders (`queryClient.ensureQueryData`).

## 1. Query Keys Factory

Never use raw string literals or array literals inline for query keys. Define a single `queryKeys` object (e.g., `featureKeys`) to construct all query keys for a feature.

```typescript
// app/(dashboard)/{featureName}/queries.ts
import { queryOptions, useMutation, useQueryClient } from "@tanstack/react-query";
import { featureApi } from "./api";
import type { CreateFeatureInput, UpdateFeatureInput } from "./types";

export const featureKeys = {
  all: ["features"] as const,
  lists: () => [...featureKeys.all, "list"] as const,
  list: (params?: Record<string, unknown>) => [...featureKeys.lists(), params] as const,
  details: () => [...featureKeys.all, "detail"] as const,
  detail: (id: number) => [...featureKeys.details(), id] as const,
};
```

## 2. Query Options Pattern

Use the `queryOptions` helper from `@tanstack/react-query` to define your queries. This returns an object containing the `queryKey` and `queryFn`, which can be passed directly to `useQuery` or `queryClient.fetchQuery`.

```typescript
export const featureListOptions = (params?: Record<string, unknown>) =>
  queryOptions({
    queryKey: featureKeys.list(params),
    queryFn: () => featureApi.getAll(params),
  });

export const featureDetailOptions = (id: number) =>
  queryOptions({
    queryKey: featureKeys.detail(id),
    queryFn: () => featureApi.getById(id),
  });
```

*Usage in a component:*

```typescript
import { useQuery } from "@tanstack/react-query";
import { featureListOptions } from "../queries";

// Inside a component
const { data, isLoading } = useQuery(featureListOptions(filters));
```

## 3. Mutations

Define custom hooks for mutations that encapsulate the `useMutation` hook, interact with the API service, and invalidate or optimistically update relevant queries.

```typescript
export function useCreateFeature() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async (data: CreateFeatureInput) => {
      const response = await featureApi.create(data);
      return response.data;
    },
    onSuccess: () => {
      // Invalidate the entire list query to fetch fresh data
      void queryClient.invalidateQueries({ queryKey: featureKeys.lists() });
    },
  });
}

export function useUpdateFeature(id: number) {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async (data: UpdateFeatureInput) => {
      const response = await featureApi.patch(id, data);
      return response.data;
    },
    onSuccess: (updatedFeature) => {
      // Invalidate list
      void queryClient.invalidateQueries({ queryKey: featureKeys.lists() });
      
      // Seed or optimistically update the detail cache
      queryClient.setQueryData(featureKeys.detail(id), {
        success: true,
        data: updatedFeature,
      });
    },
  });
}

export function useDeleteFeature() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: async (id: number) => {
      await featureApi.delete(id);
    },
    onSuccess: () => {
      void queryClient.invalidateQueries({ queryKey: featureKeys.lists() });
    },
  });
}
```

## Golden Rules for Queries

1. **Colocation**: `queries.ts` should live inside the feature module (`app/(dashboard)/{featureName}/queries.ts`).
2. **queryOptions**: Always use `queryOptions()` for queries; do not inline `useQuery({ queryKey: ..., queryFn: ... })` in components.
3. **Invalidation**: Always invalidate the relevant query keys in the `onSuccess` callback of mutations. Be specific (e.g., `featureKeys.lists()`) rather than invalidating `featureKeys.all` if updating a single list.
4. **Returns**: Extract `.data` from API responses in the `mutationFn` to provide clean data to the `onSuccess` callback.
