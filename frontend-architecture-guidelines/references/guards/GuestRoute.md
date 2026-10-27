# GuestRoute.tsx

**Why this pattern?**
> Prevents authenticated users from accessing public auth pages (like Login).

## Implementation

```tsx
import { Navigate, Outlet, useLocation } from "react-router";
import { useAuth } from "../useAuth";
import { PageSkeleton } from "@/components/ui/Skeleton";

/**
 * GuestRoute — layout route that blocks authenticated users from
 * auth pages (e.g. /login). Redirects them to /admin or the
 * ?returnTo= destination they originally had in the URL.
 */
export function GuestRoute() {
  const { isAuthenticated, isLoading } = useAuth();
  const { search } = useLocation();

  if (isLoading) return <PageSkeleton />;

  if (isAuthenticated) {
    const params = new URLSearchParams(search);
    const returnTo = params.get("returnTo");
    const destination =
      returnTo && returnTo.startsWith("/") && !returnTo.startsWith("//")
        ? returnTo
        : "/admin";
    return <Navigate to={destination} replace />;
  }

  return <Outlet />;
}

```
