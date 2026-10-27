# AppErrorBoundary.tsx

**Why this pattern?**
> The global application-level error boundary that catches fatal crashes outside of the router.

## Implementation

```tsx
import type { PropsWithChildren } from "react";
import { ErrorBoundary } from "./ErrorBoundary";

/**
 * Last-resort application boundary. Route-level `errorElement`s do the real
 * work; this only catches crashes that escape the router (e.g. errors thrown
 * by RouterProvider internals).
 */
export function AppErrorBoundary({ children }: PropsWithChildren) {
  return <ErrorBoundary>{children}</ErrorBoundary>;
}

```
