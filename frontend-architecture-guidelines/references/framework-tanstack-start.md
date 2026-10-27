# TanStack Start Full-Stack Architecture Guide

TanStack Start is the full-stack, server-side rendering (SSR) framework built on top of TanStack Router, TanStack Query, Vite, and Nitro. This document outlines the architectural patterns for building full-stack applications with TanStack Start.

## 1. Core Concepts

1. **Unified Routing**: Powered by TanStack Router file-based routing (`src/routes/`).
2. **Server Functions (`createServerFn`)**: Type-safe RPC endpoints defined alongside route code that execute exclusively on the server.
3. **Isomorphic Data Loading**: Data can be fetched on the server during SSR and hydrated seamlessly to the client with TanStack Query.
4. **Nitro Engine**: Production deployment targets any JavaScript runtime (Node.js, Bun, Cloudflare Workers, Vercel, AWS Lambda).

---

## 2. Project Directory Structure

```text
src/
├── routes/
│   ├── __root.tsx              # Root route shell, HTML head, and QueryClient hydration
│   ├── index.tsx               # Homepage
│   └── dashboard/
│       ├── members/
│       │   ├── index.tsx       # Route with createServerFn / loader
│       │   ├── -server.ts      # Dedicated server functions
│       │   ├── -queries.ts     # Client query options wrapping server functions
│       │   ├── -schemas.ts     # Shared Zod validation schemas
│       │   ├── -types.ts       # Shared TypeScript types
│       │   └── -components/    # Feature UI components
├── api/                        # Shared backend communication clients
├── components/                 # Shared UI kit (DataTable, Forms, UI primitives)
└── integrations/               # Router and QueryClient configurations
```

---

## 3. Server Functions (`createServerFn`)

Server functions replace traditional REST API route handlers for internal app communication. They automatically handle serialization, CORS, and request dispatching.

### Defining a Server Function
```typescript
// src/routes/dashboard/members/-server.ts
import { createServerFn } from "@tanstack/react-start";
import { z } from "zod";
import { createMemberSchema } from "./-schemas";

const API_BASE = process.env.API_BASE_URL;

export const fetchMembersServerFn = createServerFn({ method: "GET" })
  .validator((d: unknown) =>
    z.object({
      page: z.number().default(1),
      search: z.string().optional(),
    }).parse(d)
  )
  .handler(async ({ data }) => {
    // Executes ONLY on the server: credentials and internal URLs remain private
    const url = new URL("/members", API_BASE);
    url.searchParams.set("page", String(data.page));
    if (data.search) url.searchParams.set("search", data.search);

    const res = await fetch(url.toString(), {
      headers: {
        Authorization: `Bearer ${process.env.INTERNAL_SERVICE_TOKEN}`,
      },
    });

    if (!res.ok) {
      throw new Error(`Failed to load members: ${res.statusText}`);
    }

    return res.json();
  });

export const createMemberServerFn = createServerFn({ method: "POST" })
  .validator((d: unknown) => createMemberSchema.parse(d))
  .handler(async ({ data }) => {
    const res = await fetch(`${API_BASE}/members`, {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify(data),
    });

    if (!res.ok) {
      throw new Error("Failed to create member");
    }

    return res.json();
  });
```

---

## 4. Route Loaders with Server Functions & TanStack Query

Server functions integrate directly with TanStack Query and route loaders:

```tsx
// src/routes/dashboard/members/index.tsx
import { createFileRoute } from "@tanstack/react-router";
import { queryOptions, useSuspenseQuery } from "@tanstack/react-query";
import { z } from "zod";
import { fetchMembersServerFn } from "./-server";
import { DataTable } from "@/components/data-table/DataTable";
import { columns } from "./-components/columns";

const searchSchema = z.object({
  page: z.number().catch(1),
  search: z.string().optional(),
});

export const memberListOptions = (params: z.infer<typeof searchSchema>) =>
  queryOptions({
    queryKey: ["members", "list", params],
    queryFn: () => fetchMembersServerFn({ data: params }),
  });

export const Route = createFileRoute("/dashboard/members/")({
  validateSearch: searchSchema,
  loaderDeps: ({ search }) => search,
  loader: async ({ context: { queryClient }, deps }) => {
    // Ensures data is preloaded during SSR and cached on the client
    await queryClient.ensureQueryData(memberListOptions(deps));
  },
  component: MembersPage,
});

function MembersPage() {
  const search = Route.useSearch();
  const { data } = useSuspenseQuery(memberListOptions(search));

  return (
    <div>
      <h1>Members</h1>
      <DataTable data={data.results} columns={columns} />
    </div>
  );
}
```

---

## 5. Hydration & Root Configuration

In `src/routes/__root.tsx`:
```tsx
import { createRootRouteWithContext, Outlet, ScrollRestoration } from "@tanstack/react-router";
import { QueryClient, dehydrate, HydrationBoundary } from "@tanstack/react-query";
import { Meta, Scripts } from "@tanstack/react-start";

interface RouterContext {
  queryClient: QueryClient;
}

export const Route = createRootRouteWithContext<RouterContext>()({
  head: () => ({
    meta: [
      { charSet: "utf-8" },
      { name: "viewport", content: "width=device-width, initial-scale=1" },
      { title: "EthioMFI Banking" },
    ],
  }),
  component: RootComponent,
});

function RootComponent() {
  const { queryClient } = Route.useRouteContext();

  return (
    <html lang="en">
      <head>
        <Meta />
      </head>
      <body>
        <HydrationBoundary state={dehydrate(queryClient)}>
          <Outlet />
        </HydrationBoundary>
        <ScrollRestoration />
        <Scripts />
      </body>
    </html>
  );
}
```

---

## 6. Mutations in TanStack Start

Client mutations invoke server functions wrapped in `useMutation`:

```typescript
// src/routes/dashboard/members/-queries.ts
import { useMutation, useQueryClient } from "@tanstack/react-query";
import { createMemberServerFn } from "./-server";
import type { CreateMemberInput } from "./-types";

export function useCreateMember() {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (input: CreateMemberInput) =>
      createMemberServerFn({ data: input }),
    onSuccess: () => {
      // Automatic cache invalidation across the app
      queryClient.invalidateQueries({ queryKey: ["members"] });
    },
  });
}
```

---

## 7. Migration Mapping: TanStack Router SPA -> TanStack Start

| SPA Pattern (`ethiomfi-frontend`) | TanStack Start Pattern | Advantage |
| :--- | :--- | :--- |
| `apiClient.getAll()` direct from client | `fetchMembersServerFn` via `createServerFn` | API tokens hidden, no CORS issues |
| Route prefetching in client browser | Isomorphic SSR pre-render | Zero initial loading screen, SEO ready |
| Memory-only token in browser | HTTP-only encrypted session cookies | True server-side authorization |
| `-` prefix for modules | `-` prefix for modules | Exactly identical convention |
