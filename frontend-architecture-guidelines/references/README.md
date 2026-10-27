# Master Architecture Reference Catalog

This directory serves as the centralized, authoritative knowledge base for frontend architecture standards across the project. It organizes both common enterprise patterns and framework-specific implementations.

### 🏷️ Searchable Keywords & Categories (skillsmp.com)
`react-19`, `nextjs-app-router`, `tanstack-router`, `tanstack-start`, `tanstack-query-v5`, `tanstack-table-v9`, `tanstack-store`, `domain-driven-design`, `feature-first`, `deletion-safe`, `circuit-breaker`, `token-manager`, `401-refresh-interceptor`, `error-boundary`, `route-error-fallback`, `rbac`, `can-component`, `permissions-catalog`, `zod-v4`, `react-hook-form`, `offline-first`, `indexeddb-persistence`, `vitest`, `typescript`, `agent-skill`

---

## 1. Framework & Routing Architecture

Detailed architecture guides for routing, data loading, mutations, and server execution:

- **[TanStack Router SPA (`lib-tanstack-router.md`)](./lib-tanstack-router.md)**: Client-side SPA architecture used in `ethiomfi-frontend`. Details the `-` prefix for non-route modules, `createFileRoute`, `validateSearch` with Zod `.catch()`, `beforeLoad` route guards, and prefetching in `loader`.
- **[TanStack Start SSR (`framework-tanstack-start.md`)](./framework-tanstack-start.md)**: Full-stack isomorphic architecture with TanStack Start, server functions (`createServerFn().handler()`), RPC data loading, and Nitro engine deployment.
- **[Next.js App Router (`framework-nextjs.md`)](./framework-nextjs.md)**: Full-stack React Server Components (RSC) architecture, Server Actions (`'use server'`), leaf client boundaries (`'use client'`), caching with `revalidateTag(tag, 'max')` / `updateTag`, and edge middleware.

---

## 2. API & Network Resilience Layer (`api/`)

Centralized HTTP client infrastructure built on Axios with enterprise resilience patterns:

- **[client.md](./api/client.md)**: The `ApiClient` factory (`apiClient<T>()`), interceptor chain, automatic 401 token refresh loop prevention, and CRUD wrappers.
- **[circuit-breaker.md](./api/circuit-breaker.md)**: Circuit breaker state machine (Closed, Open, Half-Open) preventing cascading backend failures.
- **[token-manager.md](./api/token-manager.md)**: In-memory JWT access token management, single-flight refresh promises, and `BroadcastChannel` multi-tab sync.
- **[errors.md](./api/errors.md)**: Error normalization (`toApiError`, `extractErrorMessage`) mapping backend validation structures to user-facing strings.
- **[unauthorized-handler.md](./api/unauthorized-handler.md)**: Decoupled registry pattern allowing the API transport to trigger session clear and redirect without importing stores or routes.
- **[unwrap.md](./api/unwrap.md)**: Resilient unpacking of backend `{ success, timestamp, path, message, data }` envelopes.
- **[utils.md](./api/utils.md)**: Resilient collection extraction (`getListData`) and page count extraction (`getPageCount`).
- **[types.md](./api/types.md)**: Standard TypeScript contracts for `ApiResponse<T>`, `PaginatedApiResponse<T>`, and `ApiError`.

---

## 3. Data Table Abstraction (`data-table/`)

Production-grade data table built on TanStack Table v9:

- **[DataTable.md](./data-table/data-table.md)**: Central `DataTable` component handling server-side pagination, search, filters, empty states, and skeleton loading.
- **[DataTablePagination.md](./data-table/data-table-pagination.md)**: Pagination bar with page size selector, jump controls, and page status.
- **[DataTableSearch.md](./data-table/data-table-search.md)**: Debounced search input driving URL search parameters.
- **[DataTableFilter.md](./data-table/data-table-filter.md)**: Dropdown filter integration supporting faceted filters.
- **[Table.md](./data-table/Table.md)**: Styled HTML table primitives (`TableHeader`, `TableBody`, `TableRow`, `TableCell`).

---

## 4. Error Boundaries & Crash Recovery (`error-boundary/`)

Tiered crash protection isolating failures to the smallest possible UI region:

- **[AppErrorBoundary.md](./error-boundary/AppErrorBoundary.md)**: Root application boundary catching fatal boot errors.
- **[ErrorBoundary.md](./error-boundary/ErrorBoundary.md)**: Core React class component boundary supporting `resetKeys` and `onReset`.
- **[ErrorFallback.md](./error-boundary/ErrorFallback.md)**: Full-page error screen with retry and navigation buttons.
- **[RouteErrorFallback.md](./error-boundary/RouteErrorFallback.md)**: Router `errorComponent` handling loader and render failures.
- **[MinimalFallback.md](./error-boundary/MinimalFallback.md)**: Compact fallback for dashboard widgets and region cards.
- **[ErrorReporter.md](./error-boundary/ErrorReporter.md)**: Centralized error logging utility.

---

## 5. Form Architecture & Input Controls (`forms/`)

Declarative forms powered by React Hook Form and Zod schemas:

- **[Form.md](./forms/Form.md)**: Root `<Form>` wrapper handling form instantiation, validation schemas, and submission callbacks.
- **[FormField.md](./forms/FormField.md)**: Unified input field supporting text, number, tel, date, and password types.
- **[SelectField.md](./forms/SelectField.md)**: Radix select dropdown integrated with form validation.
- **[SubmitButton.md](./forms/SubmitButton.md)**: Submit button with automatic loading spinners and pending state disabling.
- **[FormContainer.md](./forms/FormContainer.md)**: Responsive layout container for form grids.
- **[FormImagePicker.md](./forms/FormImagePicker.md)** & **[FormMultipleImagePicker.md](./forms/FormMultipleImagePicker.md)**: Single and multiple image upload fields.
- **[FormVideoPicker.md](./forms/FormVideoPicker.md)** & **[FormMultipleVideoPicker.md](./forms/FormMultipleVideoPicker.md)**: Video upload fields.
- **[FormSwitch.md](./forms/FormSwitch.md)** & **[FormSwitchGroup.md](./forms/FormSwitchGroup.md)**: Switch toggle components.
- **[FormTextarea.md](./forms/FormTextarea.md)**: Multiline text input with character limit support.
- **[FormMarkdownEditor.md](./forms/FormMarkdownEditor.md)**: Rich text / Markdown editor.

---

## 6. Role-Based Access Control & Route Guards (`guards/`)

Multi-layered authorization reflecting the backend permission catalog:

- **[Can.md](./guards/Can.md)**: Declarative `<Can perform={...}>` UI element gating component.
- **[PermissionDenied.md](./guards/PermissionDenied.md)**: 403 Forbidden display component indicating the missing permission codename.
- **[route-guards.md](./guards/route-guards.md)**: TanStack Router `beforeLoad` functions (`protectedRoute`, `requireAccess`, `canAccessDashboard`).
- **[permissions.md](./guards/permissions.md)**: Constants catalog mirroring backend permission seeders.
- **[RolePermissionsDialog.md](./guards/RolePermissionsDialog.md)**: Role permission assignment modal.
- **[PaginatedPermissionTable.md](./guards/PaginatedPermissionTable.md)**: Grid table for toggling permissions.
- **[FormPermissionTable.md](./guards/FormPermissionTable.md)**: Form controller wrapper for permission grids.

---

## 7. Client State & Persistence (`stores/` & `integrations/`)

- **[auth-store.md](./stores/auth-store.md)**: In-memory token management, `@tanstack/store` user profile state, and multi-tab broadcast sync.
- **[query-client.md](./integrations/query-client.md)**: TanStack Query client configuration with offline-first network modes and retry predicates.
- **[query-persistence.md](./integrations/query-persistence.md)**: IndexedDB caching layer for offline resilience and cache purging on logout.

---

## 8. Uploads & UI Primitives (`uploads/` & `ui/`)

- **[uploads.md](./uploads/uploads.md)**: Upload service for multipart file storage.
- **[button.md](./ui/button.md)**: Primary, secondary, outline, and ghost button primitives.
- **[input.md](./ui/input.md)**: Base styled input primitive.
- **[ImageInput.md](./ui/ImageInput.md)** & **[MultipleImageInput.md](./ui/MultipleImageInput.md)**: Image dropzone primitives.
- **[table.md](./ui/table.md)**: Styled table primitives.

---

## 9. Consolidated Peer Skills (`skills/`)

High-level architectural summaries from the 9 specialized frontend skills:
- **[api-service.md](./skills/api-service.md)**
- **[data-table.md](./skills/data-table.md)**
- **[error-boundary.md](./skills/error-boundary.md)**
- **[file-uploader.md](./skills/file-uploader.md)**
- **[query-options.md](./skills/query-options.md)**
- **[rbac-ui.md](./skills/rbac-ui.md)**
- **[react-ddd-architecture.md](./skills/react-ddd-architecture.md)**
- **[react-forms.md](./skills/react-forms.md)**
- **[route-guards.md](./skills/route-guards.md)**
