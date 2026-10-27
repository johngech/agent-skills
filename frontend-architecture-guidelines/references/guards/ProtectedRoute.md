# ProtectedRoute.tsx

**Why this pattern?**
> Route-level wrapper that intercepts navigation based on authentication status, preventing layout flashing.

## Implementation

```tsx
import { Navigate, useLocation } from "react-router";
import { useAuth } from "../useAuth";
import { PageSkeleton } from "@/components/ui/Skeleton";
import type { PropsWithChildren } from "react";

/**
 * ProtectedRoute — blocks unauthenticated users from admin pages.
 * Redirects to /login?returnTo=<current-path> so the user lands
 * back on the right page after signing in.
 */
interface ProtectedRouteProps extends PropsWithChildren {
  requireStaff?: boolean;
}

export function ProtectedRoute({
  children,
  requireStaff = false,
}: Readonly<ProtectedRouteProps>) {
  const { isAuthenticated, isLoading, user } = useAuth();
  const location = useLocation();

  if (isLoading) return <PageSkeleton />;

  if (!isAuthenticated) {
    const returnTo = encodeURIComponent(location.pathname + location.search);
    return <Navigate to={`/login?returnTo=${returnTo}`} replace />;
  }

  if (requireStaff && user && !user.is_staff) {
    return <Navigate to="/dashboard" replace />;
  }

  return <>{children}</>;
}

```
