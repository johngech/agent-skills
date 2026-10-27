# TanStack Router Architecture & Implementation Guide

This guide details the router architecture for Client-Side SPAs built with **TanStack Router**, directly reflecting the production setup in this codebase.

## 1. Core Principles

1. **Type-Safe File-Based Routing**: Routes are defined in `src/routes/` and compiled to `src/routeTree.gen.ts` via the TanStack Router CLI (`tsr generate` or Vite plugin).
2. **The `-` Prefix for Feature Non-Route Modules**: Files and directories prefixed with `-` (e.g., `-api.ts`, `-queries.ts`, `-schemas.ts`, `-types.ts`, `-components/`) are ignored by the route code generator, enabling true deletion-safe co-location.
3. **URL as Single Source of Truth**: All page filter, search, sort, and pagination states are reflected in URL search parameters, validated with Zod, and linked to TanStack Query loaders.
4. **Declarative Route Lifecycle**: `validateSearch` -> `beforeLoad` (Auth & RBAC) -> `loaderDeps` -> `loader` (data prefetching) -> `component`.

---

## 2. Directory Layout & Routing Conventions

In `src/routes/`:
```text
src/routes/
├── __root.tsx                           # Root route: shell, query client restore, devtools
├── index.tsx                            # Root landing page (redirects to /dashboard or /auth/login)
├── forbidden.tsx                        # 403 Forbidden page displaying missing permission
├── dashboard.tsx                        # Dashboard layout with authentication beforeLoad
├── dashboard/
│   ├── index.tsx                        # Dashboard home
│   ├── -components/                     # Dashboard shell components (sidebar, breadcrumb)
│   └── members/                         # Feature module
│       ├── index.tsx                    # List view (/dashboard/members)
│       ├── new.tsx                      # Create view (/dashboard/members/new)
│       ├── $memberId.tsx                # Detail view (/dashboard/members/:memberId)
│       ├── edit/                        # Edit nested routes
│       ├── kyc.tsx                      # KYC step view
│       ├── -api.ts                      # Feature API client
│       ├── -queries.ts                  # Query options & mutation hooks
│       ├── -schemas.ts                  # Zod validation schemas
│       ├── -types.ts                    # TypeScript types & interfaces
│       └── -components/                 # Feature-private components
```

---

## 3. Route Lifecycle Contract

Every routable file exports a `Route` defined via `createFileRoute`:

```tsx
import { createFileRoute } from "@tanstack/react-router";
import { z } from "zod";
import { requireAccess } from "@/components/guards/route-guards";
import { RouteErrorFallback } from "@/components/error-boundary/RouteErrorFallback";
import { MEMBER_PERMISSIONS } from "@/config/permissions";
import { memberListOptions } from "./-queries";

// 1. Search Params Validation (Zod v4 with .catch())
const searchSchema = z.object({
  page: z.number().catch(1),
  limit: z.number().catch(10),
  search: z.string().optional(),
  sort: z.string().optional(),
  filter: z.record(z.string(), z.string()).optional(),
});

export type MemberListSearch = z.infer<typeof searchSchema>;

export const Route = createFileRoute("/dashboard/members/")({
  // 1. Validate and coerce URL search parameters
  validateSearch: searchSchema,

  // 2. Auth & Permission checks before loading or rendering
  beforeLoad: ({ location }) =>
    requireAccess({
      location,
      permissions: [MEMBER_PERMISSIONS.list, MEMBER_PERMISSIONS.view],
      primary: MEMBER_PERMISSIONS.list,
    }),

  // 3. Declare which search params invalidate the loader cache
  loaderDeps: ({ search }) => search,

  // 4. Prefetch data in router loader so component renders without layout shift
  loader: ({ context: { queryClient }, deps }) =>
    queryClient.query({
      ...memberListOptions(deps),
      staleTime: "static",
    }),

  // 5. Crash protection fallback
  errorComponent: RouteErrorFallback,

  // 6. View component
  component: MembersPage,
});
```

---

## 4. Search Parameter Validation & Handling

TanStack Router parses URLs into JSON-first application state.

### Defensive Defaults with `.catch()`
Never throw a crash screen for a mistyped URL. Use `.catch(default)` to gracefully fall back:

```ts
const searchSchema = z.object({
  page: z.number().catch(1),
  limit: z.number().catch(10),
  search: z.string().optional(),
  sort: z.string().optional(), // "-name" indicates descending
});
```

### Accessing & Mutating Search in Components
```tsx
function MembersPage() {
  const search = Route.useSearch();
  const navigate = Route.useNavigate();

  const handleSearch = (term: string) => {
    navigate({
      search: (prev) => ({
        ...prev,
        search: term || undefined,
        page: 1, // Reset to page 1 on new search
      }),
    });
  };

  return (
    <div>
      <SearchInput value={search.search ?? ""} onSearch={handleSearch} />
    </div>
  );
}
```

---

## 5. Route Protection (`beforeLoad`)

Route guards execute asynchronously before any child loaders or components are evaluated:

```ts
// src/components/guards/route-guards.ts
import { redirect } from "@tanstack/react-router";
import { authActions, authStore } from "@/stores/auth";

export async function protectedRoute({ location }: { location: ParsedLocation }) {
  if (typeof window === "undefined") return;

  const isAuthenticated = await authActions.ensureSession();
  if (!isAuthenticated) {
    throw redirect({
      to: "/auth/login",
      search: { returnTo: location.href },
      replace: true,
    });
  }
}

export async function requireAccess({
  location,
  permissions,
  primary,
}: {
  location: ParsedLocation;
  permissions: readonly string[];
  primary?: string;
}) {
  await protectedRoute({ location });

  const { user } = authStore.state;
  if (!user || user.isSuperuser) return;

  const granted = new Set(user.permissions ?? []);
  if (permissions.some((perm) => granted.has(perm))) return;

  throw redirect({
    to: "/forbidden",
    search: { permission: primary ?? permissions[0] ?? "" },
    replace: true,
  });
}
```

---

## 6. Pre-fetching & Suspense Integration

1. In route `loader`: Call `queryClient.query(...)` or `queryClient.ensureQueryData(...)`.
2. In route `component`: Use `useSuspenseQuery(...)` from `@tanstack/react-query`. Because the loader pre-fetched the data, the query resolves synchronously on navigation, eliminating flashing loaders and layout shifts.

```tsx
function MembersPage() {
  const search = Route.useSearch();
  const { data } = useSuspenseQuery(memberListOptions(search));

  return <DataTable data={data.results} columns={columns} />;
}
```

---

## 7. Golden Rules for TanStack Router

1. **Prefix Non-Route Files with `-`**: All sub-modules in feature route directories (`-api.ts`, `-queries.ts`, `-schemas.ts`, `-types.ts`, `-components/`) MUST start with `-` so TanStack Router does not register them as routes.
2. **Never Hardcode Links**: Use `<Link to="/dashboard/members/$memberId" params={{ memberId: id }} />` for full type safety.
3. **Keep Route Files Thin**: Delegate layout, markup, and form logic to `-components/`.
4. **Catch, Don't Crash in Search**: Use `.catch()` in Zod search schemas so users editing URLs are never stranded on error pages.
