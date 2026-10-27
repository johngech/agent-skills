# Enterprise Frontend Architecture Guidelines (Agent Skill)

[![Version](https://img.shields.io/badge/version-1.0.0-blue.svg)](https://skillsmp.com)
[![Category](https://img.shields.io/badge/category-Frontend%20Architecture-green.svg)](https://skillsmp.com)
[![Frameworks](https://img.shields.io/badge/frameworks-TanStack%20Router%20%7C%20TanStack%20Start%20%7C%20Next.js%20App%20Router-orange.svg)](https://skillsmp.com)
[![React](https://img.shields.io/badge/react-19.x-61dafb.svg)](https://react.dev)
[![TypeScript](https://img.shields.io/badge/typescript-5.x%20%2F%206.x-3178c6.svg)](https://www.typescriptlang.org)

A production-grade, battle-tested agent skill providing comprehensive architectural rules, boundary enforcement, and design patterns for building scalable enterprise React applications.

Built on **Domain-Driven Design (DDD)**, **Feature-First Co-location**, **Deletion-Safe Isolation**, **Schema-First Validation (Zod)**, **Resilient API Transports (Circuit Breaker & 401 Refresh)**, and **Multi-Framework Strategy** supporting **TanStack Router SPA**, **TanStack Start SSR**, and **Next.js App Router**.

---

## 🏷️ Searchable Keywords & Tags (skillsmp.com)

`react`, `react-19`, `nextjs`, `nextjs-app-router`, `next-15`, `next-16`, `server-components`, `rsc`, `server-actions`, `revalidate-tag`, `tanstack`, `tanstack-router`, `tanstack-start`, `tanstack-query`, `react-query`, `react-query-v5`, `tanstack-table`, `tanstack-table-v9`, `tanstack-store`, `react-router`, `vite`, `tailwindcss-v4`, `zod`, `zod-v4`, `react-hook-form`, `axios`, `circuit-breaker`, `token-manager`, `jwt-refresh`, `401-interceptor`, `error-boundary`, `route-error-fallback`, `rbac`, `role-based-access-control`, `permissions-catalog`, `permission-guard`, `can-component`, `domain-driven-design`, `ddd`, `feature-first`, `deletion-safe`, `clean-architecture`, `hexagonal-architecture`, `ports-and-adapters`, `offline-first`, `indexeddb-persistence`, `vitest`, `msw`, `typescript`, `agent-skill`, `ai-coding-guidelines`, `cursor-rules`, `antigravity-skill`, `enterprise-frontend`

---

## 🌟 Why This Architecture?

Large frontend codebases frequently suffer from:

1. **Accidental coupling**: Features cross-importing from each other, making refactoring or deprecation risky.
2. **Cascading failures**: Backend timeouts or unhandled rendering exceptions crashing the entire viewport.
3. **Flashing UI & spinners**: Client-side data fetching waterfalls and inconsistent URL state management.
4. **Security bypasses**: Hiding UI elements without rigorous route-level and network-level enforcement.

This skill equips AI coding agents and engineering teams with an authoritative architectural standard that guarantees **deletion safety**, **fault isolation**, **instant navigation**, and **consistent type-safety**.

---

## 📐 Architecture Matrix

| Architectural Capability | TanStack Router SPA (Active Codebase) | TanStack Start (Isomorphic Target) | Next.js App Router (Full-Stack Target) |
| :--- | :--- | :--- | :--- |
| **Feature Boundary** | `src/routes/dashboard/{feature}/` | `src/routes/dashboard/{feature}/` | `app/(dashboard)/{feature}/` |
| **Non-Route Co-location** | Prefixed with `-` (`-api.ts`, `-queries.ts`) | Prefixed with `-` (`-api.ts`, `-server.ts`) | Prefixed with `_` (`_components/`) |
| **Data Fetching (Read)** | `loader` + `queryClient.query` | `loader` + `createServerFn().handler()` | Server Components `fetch()` + `tags` |
| **Mutations (Write)** | `useMutation` calling `apiClient` | `useMutation` or `createServerFn()` | Server Actions (`'use server'`) |
| **Cache Invalidation** | `queryClient.invalidateQueries(...)` | `queryClient.invalidateQueries(...)` | `revalidateTag('tag', 'max')` / `updateTag` |
| **URL Search Validation** | `validateSearch` + Zod `.catch()` | `validateSearch` + Zod `.catch()` | `searchParams` prop in `page.tsx` + Zod |
| **Route Protection** | `beforeLoad` + `requireAccess` | `beforeLoad` + `requireAccess` | `middleware.ts` + Server Component checks |
| **UI Element Gating** | `<Can perform={perm}>` | `<Can perform={perm}>` | `<Can perform={perm}>` (Leaf Client Boundary) |
| **Data Tables** | `DataTable` (TanStack Table v9) | `DataTable` (TanStack Table v9) | `DataTable` (Client Boundary) |
| **Forms** | `Form` (React Hook Form + Zod) | `Form` (React Hook Form + Zod) | `Form` (React Hook Form + Zod) |

---

## 📂 Feature Module Anatomy

Every business domain is encapsulated in an autonomous, deletion-safe directory under routes:

```text
src/routes/dashboard/{feature-name}/
├── -api.ts          # Typed API endpoints using apiClient<Entity>('endpoint')
├── -queries.ts      # Query key factories, queryOptions(), mutation hooks
├── -schemas.ts      # Zod validation schemas (single source of truth) + inferred types
├── -types.ts        # TypeScript interfaces, non-schema types, param types, props
├── -stores.ts       # Optional local feature client state (TanStack Store)
├── -components/     # Feature-private components (tables, dialogs, forms)
├── index.tsx        # Feature list route
└── $id.tsx          # Feature detail route
```

> **The `-` Prefix Rule**: Files and directories prefixed with `-` are ignored by TanStack Router route generation, enabling full co-location of data access, queries, validation, and private components alongside routes.

---

## 🛡️ Core Infrastructure & Resilience Highlights

### 1. Centralized HTTP Transport with Circuit Breaker
- **`apiClient<T>()` factory**: Eliminates raw `axios` usage in feature code.
- **Circuit Breaker Pattern**: Automatically trips to `OPEN` state upon consecutive failures, preventing cascading network stalls.
- **Single-Flight 401 Refresh**: In-memory JWT access token rotation with automatic replay of queued requests.
- **Decoupled Unauthorized Handler**: Inverted notification pattern that clears sessions and redirects without coupling transport to routers or stores.

### 2. Standardized Data Tables (TanStack Table v9)
- Zero raw HTML `<table>` elements in feature code.
- URL search parameters as the single source of truth for table state (`page`, `limit`, `search`, `sort`, `filter`).
- Skeletons, loading states, column sorting, and responsive pagination.

### 3. Declarative Forms (React Hook Form + Zod)
- Schema-first validation with runtime type inference.
- Standardized form wrappers: `<Form>`, `<FormField>`, `<SelectField>`, `<SubmitButton>`.
- Automatic server-side validation error mapping with Sonner toast alerts.

### 4. Multi-Tier Error Boundaries
- **App Root Boundary**: Isolates fatal bootstrapping errors.
- **Layout Boundary**: Preserves navigation shells when a page section errors.
- **Route Error Fallback**: Bound to router `errorComponent`.
- **Minimal Widget Fallback**: Isolates individual cards and dashboard widgets.

### 5. Multi-Tab Session Synchronization & Offline Persistence
- In-memory access tokens with `BroadcastChannel` synchronization (signing out in one tab immediately signs out all tabs).
- TanStack Query cache persisted to browser IndexedDB via `@tanstack/query-async-storage-persister`.
- Immediate cache wipe on sign-out to prevent credential or record leakage.

---

## 🚫 Agent Must Never (Guardrails for AI Coding Agents)

1. Never create top-level `src/features/`, `src/services/`, or `src/controllers/` folders.
2. Never cross-import between peer feature directories under `src/routes/`.
3. Never write raw HTML `<table>` elements in features; use `DataTable`.
4. Never write raw HTML `<form>` elements; use `Form` and `FormField`.
5. Never import `axios` directly inside feature modules; use `apiClient<T>()`.
6. Never store authentication tokens in `localStorage` or client stores; keep tokens in memory.
7. Never omit the `-` prefix on non-route files under `src/routes/` in TanStack Router.
8. Never check role strings (e.g. `role === 'admin'`); check permission codenames via `<Can perform={...}>`.

---

## 📚 Master Reference Index

Deep architectural guides are organized in the [`references/`](./references/README.md) catalog:

| Category | Reference Document | Description |
| :--- | :--- | :--- |
| **Frameworks** | [lib-tanstack-router.md](./references/lib-tanstack-router.md) | TanStack Router SPA architecture guide |
| **Frameworks** | [framework-tanstack-start.md](./references/framework-tanstack-start.md) | TanStack Start full-stack SSR guide |
| **Frameworks** | [framework-nextjs.md](./references/framework-nextjs.md) | Next.js App Router, RSC, and Server Actions |
| **API** | [client.md](./references/api/client.md) | Axios setup, ApiClient factory, and 401 retry |
| **API** | [circuit-breaker.md](./references/api/circuit-breaker.md) | Resiliency state machine |
| **API** | [token-manager.md](./references/api/token-manager.md) | In-memory JWT and single-flight refresh |
| **API** | [unauthorized-handler.md](./references/api/unauthorized-handler.md) | Decoupled 401 notification pattern |
| **UI** | [data-table.md](./references/data-table/data-table.md) | TanStack Table v9 component wrapper |
| **UI** | [Form.md](./references/forms/Form.md) | React Hook Form + Zod form provider |
| **Security** | [Can.md](./references/guards/Can.md) | Declarative UI element permission gating |
| **Security** | [route-guards.md](./references/guards/route-guards.md) | TanStack Router beforeLoad guards |
| **Security** | [permissions.md](./references/guards/permissions.md) | Backend permission seeders mirror catalog |
| **Resilience** | [ErrorBoundary.md](./references/error-boundary/ErrorBoundary.md) | Tiered error boundaries & fallbacks |
| **State** | [auth-store.md](./references/stores/auth-store.md) | TanStack Store & BroadcastChannel sync |
| **Persistence** | [query-persistence.md](./references/integrations/query-persistence.md) | IndexedDB cache persister |
| **Catalog** | [README.md](./references/README.md) | Complete reference catalog index |

---

## 📄 License & Attribution

- **Skill Name**: `frontend-architecture-guidelines`
- **Author**: Yohannes Getachew
- **Compatible Environments**: Antigravity, Cursor, Windsurf, Claude Code, GitHub Copilot
- **Repository**: [EthioMFI Banking Frontend Architecture](https://skillsmp.com)
