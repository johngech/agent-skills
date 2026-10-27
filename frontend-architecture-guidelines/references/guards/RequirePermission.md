# RequirePermission.tsx

**Why this pattern?**
> Enforces RBAC at the routing level, redirecting users if they lack required permissions.

## Implementation

```tsx
import { Navigate, useLocation } from "react-router";
import { useAuth } from "../useAuth";
import { PageSkeleton } from "@/components/ui/Skeleton";
import type { PropsWithChildren } from "react";

interface RequirePermissionProps {
  /** Permission codename (e.g. `contents.change_news`) required to access the route. */
  perform: string;
}

/**
 * Route guard — blocks access unless the user holds the given permission
 * codename (superusers always pass). Redirects to /admin when unauthorized.
 */
export function RequirePermission({
  perform,
  children,
}: Readonly<PropsWithChildren<RequirePermissionProps>>) {
  const { isAuthenticated, isLoading, hasPermission } = useAuth();
  const location = useLocation();

  if (isLoading) return <PageSkeleton />;

  if (!isAuthenticated) {
    const returnTo = encodeURIComponent(location.pathname + location.search);
    return <Navigate to={`/login?returnTo=${returnTo}`} replace />;
  }

  if (!hasPermission(perform)) {
    return <Navigate to="/admin" replace />;
  }

  return <>{children}</>;
}

```
