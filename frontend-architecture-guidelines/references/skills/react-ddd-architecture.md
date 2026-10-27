---
name: react-ddd-architecture
description: Guidelines for setting up a Frontend React application using Domain-Driven Design (DDD). Use this skill when scaffolding new projects, extracting features, or structuring large-scale React apps.
---

# Frontend Domain-Driven Design (DDD) Architecture

This document outlines the standard Domain-Driven Design (DDD) architecture for React Single Page Applications (SPAs). The core philosophy is **deletion-safe isolation**: deleting a feature's folder should completely remove it from the application without breaking the build or other features.

> **See also:** [frontend-architecture-guidelines](../../SKILL.md) for the unified, Next.js App Router-aligned blueprint with explicit TanStack Start mapping.

## 1. High-Level Directory Layout

The application is split into two distinct areas: the **Common Layer** and the **Feature Modules** (co-located under routes).

```
src/
├── api/          # COMMON: Base HTTP client (Axios), interceptors, error normalizers
├── components/   # COMMON: UI kit (Buttons, Inputs, Modals, DataTables)
├── libs/         # COMMON: Shared utilities, formatters, hooks, env config
├── types/        # COMMON: Global generic types (e.g., PaginatedResponse, ApiError)
├── stores/       # COMMON: Global client state (UI themes, sidebars)
│
└── routes/       # ROOT ROUTER AND FEATURES
    ├── auth/
    │   ├── login.tsx
    │   ├── -api.ts
    │   └── -schemas.ts
    ├── dashboard/
    └── users/
```

## 2. Golden Rules of Feature Isolation

1. **Strict Boundary**: A feature (`src/routes/billing`) **must never** import from another feature (`src/routes/users`).
2. **Inward Dependencies Only**: Features may only import from their own internals or from the global `src/common` layers (`api/`, `components/`, etc.).
3. **Autonomous Routing**: Each feature defines its own route objects. TanStack Router automatically parses and composes them.
4. **Manifest-Driven UI**: Features export metadata (like navigation menu items). The global shell aggregates these manifests. If a feature is deleted, its menu items naturally disappear.

## 3. Anatomy of a Feature Module

A feature directory encapsulates everything related to a specific business domain.

**Next.js App Router (target):**
```
app/(app)/{feature-name}/
├── api.ts         # API client and endpoints
├── queries.ts     # React Query options and mutations
├── schemas.ts     # Zod validation schemas
├── types.ts       # TypeScript interfaces
├── store.ts       # Local client state (if needed)
├── _components/   # Feature-specific components
└── page.tsx       # Routable page component
```

**TanStack Start/Router (current project):**
```
src/routes/{feature-name}/
├── -api.ts        # API client and endpoints
├── -queries.ts    # React Query options and mutations
├── -schemas.ts    # Zod validation schemas
├── -types.ts      # TypeScript interfaces
├── -stores.ts     # Local client state (if needed)
├── -components/   # Feature-specific components
└── index.tsx      # Routable page components
```

**Note:** For Next.js, no `-` prefix is needed—only route files (`page.tsx`, `layout.tsx`, etc.) are routes. Use `_components/` or underscore-prefixed folders for private modules. For TanStack Router, use `-` prefix for non-route files to prevent route registration.

### A. Data Models and Validation (`schemas.ts` / `-schemas.ts` & `types.ts` / `-types.ts`)

**Pattern:** Schema-First Types.

Define validation schemas using Zod as the single source of truth. Schema-inferred types can live alongside schemas for convenience, while other type definitions go in a dedicated types file.

- **`schemas.ts` / `-schemas.ts`** - Define Zod schemas here. Types derived from schemas (`z.infer<typeof ...>`) may be defined and exported directly from this file.
- **`types.ts` / `-types.ts`** - Define all other types here (enums, union types, discriminated unions not schema-derived, component prop types, non-schema API contracts, utility types, etc.). You can also re-export schema-inferred types from `types.ts` if you prefer a single import source.

```typescript
// schemas.ts (-schemas.ts in TanStack)
import { z } from "zod";

export const featureSchema = z.object({
  id: z.number(),
  name: z.string().min(1),
});

export const createFeatureSchema = featureSchema.omit({ id: true });

// Schema-derived types can live here
export type FeatureEntity = z.infer<typeof featureSchema>;
export type CreateFeatureInput = z.infer<typeof createFeatureSchema>;
```

```typescript
// types.ts (-types.ts in TanStack)
import type { FeatureEntity } from "./schemas";

// Non-schema-derived types go here
export type FeatureStatus = "active" | "inactive" | "pending";

export interface FeatureListProps {
  items: FeatureEntity[];
  status?: FeatureStatus;
}
```

### B. API Service Integration (`-api.ts`)

**Pattern:** Factory-Instantiated Clients.
Instead of using raw `axios.get` or `fetch` inside components, features instantiate a typed client connected to a specific base endpoint. This centralizes URL paths and standardizes CRUD operations.

```typescript
// -api.ts
import { apiClient } from "@/api/client"; // The shared, interceptor-wrapped client
import type { FeatureEntity, CreateFeatureInput } from "./-types";

// The base client is locked to the feature's endpoint
const client = apiClient<FeatureEntity>("features");

export const featureApi = {
  getAll: (params?: Record<string, unknown>) => client.getAll(params),
  getById: (id: number) => client.getById(id),
  create: (data: CreateFeatureInput) => client.create(data),
  // Custom non-CRUD actions
  activate: (id: number) => client.action(`${id}/activate`, "patch"),
};
```

### C. Server State Management (`-queries.ts`)

**Pattern:** The Query-Options Pattern.
Keep TanStack Query (React Query) logic out of the UI. Define query keys and query configurations centrally so they can be seamlessly shared between React Router loaders (for prefetching) and React components.

```typescript
// -queries.ts
import { queryOptions, useMutation, useQueryClient } from "@tanstack/react-query";
import { featureApi } from "./-api";
import type { CreateFeatureInput } from "./-types";

// 1. Strict Query Key Factory
export const featureKeys = {
  all: ["features"] as const,
  list: (filters?: Record<string, unknown>) => [...featureKeys.all, "list", filters] as const,
  detail: (id: number) => [...featureKeys.all, "detail", id] as const,
};

// 2. Query Options (usable in Loaders AND hooks)
export const featureListOptions = (filters?: Record<string, unknown>) => 
  queryOptions({
    queryKey: featureKeys.list(filters),
    queryFn: () => featureApi.getAll(filters),
  });

// 3. Custom Mutation Hooks (handles invalidation internally)
export function useCreateFeature() {
  const queryClient = useQueryClient();
  return useMutation({
    mutationFn: (data: CreateFeatureInput) => featureApi.create(data),
    onSuccess: () => {
      // Automatic cache invalidation
      queryClient.invalidateQueries({ queryKey: featureKeys.all });
    },
  });
}
```

### D. Routing (`index.tsx`)

**Pattern:** Feature-Owned Data Routes.
Features declare routes using `@tanstack/react-router` `createFileRoute`. Loaders invoke the `queryOptions` defined above to pre-fetch data before the page renders, eliminating loading spinners on navigation.

```tsx
// index.tsx
import { createFileRoute } from "@tanstack/react-router";
import { queryClient } from "@/libs/queryClient";
import { featureListOptions } from "./-queries";

export const Route = createFileRoute("/features/")({
  component: FeatureListPage,
  loader: async () => {
    // Prefetch data before render
    await queryClient.ensureQueryData(featureListOptions());
    return null;
  }
});

function FeatureListPage() {
  return <div>Features List</div>;
}
```

## Next.js App Router Mapping

| DDD Concept | Next.js App Router | TanStack Start (this project) |
| --- | --- | --- |
| Feature Module | Route segment under `app/` with colocated modules | Feature under `src/routes/` with `-api/-queries/-schemas/-types` |
| Common Layer | `app/_components/`, `components/`, `lib/`, `hooks/` | `src/components/`, `src/api/`, `src/lib/`, `src/hooks/`, `src/stores/` |
| Route Grouping | `(auth)`, `(dashboard)` route groups | Layout routes / folder organization |
| Private Modules | `_components/`, `_lib/` (underscore prefix) | `-components/` or `_components/` with `-` prefix for non-routes |
| Server/Client Split | RSC by default, `'use client'` when needed | Loaders + Query hydration; client components as needed |

## Summary

By enforcing these patterns, your React application becomes highly modular. Teams can work on different features without merge conflicts, code-splitting naturally falls along feature boundaries, and removing a deprecated feature is as simple as deleting its folder and unregistering its route.
