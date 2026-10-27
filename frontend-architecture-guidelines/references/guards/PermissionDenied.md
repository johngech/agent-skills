# Forbidden & Permission Denied (`PermissionDenied.tsx`)

Located at: `src/components/guards/PermissionDenied.tsx` and `src/routes/forbidden.tsx`

## Purpose

When a user attempts to access a protected route without holding the required permissions, the router redirects them to the `/forbidden` route, which renders `<PermissionDenied>`.

The component displays the exact missing permission codename so that the user and their system administrator know precisely what permission must be granted.

---

## Implementation

```tsx
import { Link } from "@tanstack/react-router";
import { ShieldAlert } from "lucide-react";
import { Button } from "@/components/ui/button";

interface PermissionDeniedProps {
  permission?: string;
  onRetry?: () => void;
}

export function PermissionDenied({ permission, onRetry }: PermissionDeniedProps) {
  return (
    <div className="flex min-h-[400px] flex-col items-center justify-center p-6 text-center">
      <div className="rounded-full bg-destructive/10 p-4 text-destructive mb-4">
        <ShieldAlert className="h-10 w-10" />
      </div>
      <h2 className="text-2xl font-bold tracking-tight">Access Restricted</h2>
      <p className="mt-2 text-sm text-muted-foreground max-w-md">
        You do not have permission to view this resource.
      </p>

      {permission && (
        <div className="mt-4 rounded border bg-muted/50 px-3 py-1 font-mono text-xs text-muted-foreground">
          Required permission: <code>{permission}</code>
        </div>
      )}

      <div className="mt-6 flex gap-3">
        {onRetry && (
          <Button variant="outline" onClick={onRetry}>
            Try Again
          </Button>
        )}
        <Button asChild>
          <Link to="/dashboard">Back to Dashboard</Link>
        </Button>
      </div>
    </div>
  );
}
```

---

## Route Integration (`/forbidden`)

```tsx
// src/routes/forbidden.tsx
import { createFileRoute } from "@tanstack/react-router";
import { z } from "zod";
import { PermissionDenied } from "@/components/guards/PermissionDenied";

export const Route = createFileRoute("/forbidden")({
  validateSearch: z.object({
    permission: z.string().optional(),
  }),
  component: ForbiddenPage,
});

function ForbiddenPage() {
  const { permission } = Route.useSearch();
  return <PermissionDenied permission={permission} />;
}
```
