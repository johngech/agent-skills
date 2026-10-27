---
name: project-route-guards
description: How route guards and UI permissions are handled in this project. Use when protecting routes in react-router or hiding/showing UI elements based on user permissions. Enforces the reuse of ProtectedRoute, RequirePermission, and Can components.
---

# Project Route Guards & Permissions

This project implements a comprehensive Role-Based Access Control (RBAC) model. The frontend enforces this via route-level wrappers (for navigation gating) and component-level wrappers (for UI gating).

Agents **MUST ALWAYS** use these abstractions instead of writing manual `useAuth` checks in their route definitions or UI components.

## 1. Route-Level Guards (React Router)

When defining routes (e.g., inside `app/(dashboard)/{feature}/route.tsx` or the root `routes/index.tsx`), wrap the `element` with the appropriate guard.

### `ProtectedRoute`

Blocks unauthenticated users and optionally enforces the `is_staff` flag. It automatically handles redirecting unauthenticated users to `/login?returnTo=...` so they land back on the requested page after authentication.

```tsx
// Example from a feature's route definition
import { ProtectedRoute } from "@/features/auth/guards/ProtectedRoute";
import { AppLayout } from "@/features/admin/AppLayout";

export const adminRoutes = [
  {
    path: "/admin",
    element: (
      <ProtectedRoute requireStaff={true}>
        <AppLayout />
      </ProtectedRoute>
    ),
    // children routes...
  }
];
```

### `RequirePermission`

Blocks access unless the user holds a specific permission codename (e.g., `contents.change_news`). If unauthorized, it redirects to the `/admin` dashboard. Note that `is_superuser` automatically bypasses this check under the hood.

```tsx
import { RequirePermission } from "@/features/auth/guards/RequirePermission";
import { CreateNewsPage } from "./pages/CreateNewsPage";

export const newsAdminRoutes = [
  {
    path: "news/create",
    element: (
      <RequirePermission perform="contents.add_news">
        <CreateNewsPage />
      </RequirePermission>
    ),
  }
];
```

### `GuestRoute`

The inverse of `ProtectedRoute`. It prevents authenticated users from accessing public auth pages (like Login or Register), redirecting them instead to their dashboard.

## 2. Component-Level Guards (UI)

When you need to hide or show buttons, links, or sections based on user permissions, do not use `if (user.roles.includes(...))` in your components. Always use the `<Can>` component.

### `Can` Component

Renders its children only if the current user holds the required permission codename.

```tsx
import { Can } from "@/features/auth/guards/Can";
import { Button } from "@/components/ui/Button";

export function NewsDetailActions({ newsId }) {
  return (
    <div className="flex gap-2">
      {/* Everyone can view, but only authorized users can Edit/Delete */}
      <Can perform="contents.change_news">
        <Button onClick={() => edit(newsId)}>Edit</Button>
      </Can>
      
      <Can perform="contents.delete_news">
        <Button variant="danger" onClick={() => remove(newsId)}>Delete</Button>
      </Can>
    </div>
  );
}
```

## Golden Rules for Permissions

1. **Never Hardcode Auth Checks**: Do not manually extract `user` from `useAuth` to perform redirects in `useEffect`. Use the route wrappers (`ProtectedRoute`, `RequirePermission`).
2. **Never Hardcode Role Checks**: Avoid checking `user.is_staff` or `user.is_superuser` manually in components. Use `<Can perform="...">` for UI hiding and `<RequirePermission perform="...">` for route blocking.
3. **Backend is Authoritative**: Remember that UI hiding and frontend route guards are purely UX. The backend will reject unauthorized requests regardless.

## Core Implementation References

Review the underlying components that power this permission abstraction:

- [ProtectedRoute.tsx](../guards/ProtectedRoute.md)
- [RequirePermission.tsx](../guards/RequirePermission.md)
- [Can.tsx](../guards/Can.md)
- [GuestRoute.tsx](../guards/GuestRoute.md)
