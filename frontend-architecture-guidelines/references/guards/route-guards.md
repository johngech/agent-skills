# Route Guards (`route-guards.ts`)

Located at: `src/components/guards/route-guards.ts`

## Purpose

Enforces authentication and role-based access control (RBAC) at the routing level before route loaders or components execute.

Uses TanStack Router's asynchronous `beforeLoad` lifecycle hook to evaluate sessions and permissions, short-circuiting with `throw redirect(...)`.

---

## Key Functions

### 1. `protectedRoute({ location })`
Verifies user authentication. Calls `authActions.ensureSession()` to rebuild memory tokens from the HTTP-only refresh cookie (preventing spurious redirects on browser refresh):

```typescript
export async function protectedRoute(opts: {
  location: ParsedLocation;
}): Promise<void> {
  if (typeof window === "undefined") return;

  const isAuthenticated = await authActions.ensureSession();
  if (!isAuthenticated) {
    throw redirect({
      to: "/auth/login",
      search: { returnTo: opts.location.href },
      replace: true,
    });
  }
}
```

### 2. `requireAccess({ location, permissions, primary? })`
Enforces feature-level authorization using any-of semantics. If the user does not hold at least one of the specified permissions (and is not a superuser), redirects to `/forbidden?permission=<codename>`:

```typescript
export async function requireAccess(opts: {
  location: ParsedLocation;
  permissions: readonly string[];
  primary?: string;
}): Promise<void> {
  await protectedRoute({ location: opts.location });

  const { user } = authStore.state;
  if (!user || user.isSuperuser) return;

  const granted = new Set(user.permissions ?? []);
  if (opts.permissions.some((perm) => granted.has(perm))) return;

  throw redirect({
    to: "/forbidden",
    search: { permission: opts.primary ?? opts.permissions[0] ?? "" },
    replace: true,
  });
}
```

### 3. `canAccessDashboard(user)`
Determines whether a user has permission to view the dashboard shell:
```typescript
export function canAccessDashboard(user: {
  isSuperuser?: boolean;
  permissions?: string[];
}): boolean {
  if (user.isSuperuser) return true;
  const granted = new Set(user.permissions ?? []);
  return DASHBOARD_ACCESS_PERMISSIONS.some((perm) => granted.has(perm));
}
```

### 4. `parseReturnTo(value)`
Sanitizes `returnTo` search parameters to prevent Open Redirect attacks. Rejects protocol-relative URLs (`//evil.com`) and `/auth` loops.

---

## Route Usage Pattern

```tsx
export const Route = createFileRoute("/dashboard/members/")({
  beforeLoad: ({ location }) =>
    requireAccess({
      location,
      permissions: [MEMBER_PERMISSIONS.list, MEMBER_PERMISSIONS.view],
      primary: MEMBER_PERMISSIONS.list,
    }),
  component: MembersPage,
});
```
