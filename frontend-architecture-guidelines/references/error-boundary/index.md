# index.ts

**Why this pattern?**
> Barrel export for the error boundary module.

## Implementation

```typescript
export { AppErrorBoundary } from "./AppErrorBoundary";

export { ErrorBoundary } from "./ErrorBoundary";
export type { ErrorBoundaryProps } from "./ErrorBoundary";

export { ErrorFallback } from "./ErrorFallback";
export type { ErrorFallbackProps } from "./ErrorFallback";

export { MinimalFallback } from "./MinimalFallback";
export type { MinimalFallbackProps } from "./MinimalFallback";

export { RouteErrorFallback } from "./RouteErrorFallback";

export { reportError } from "./ErrorReporter";
export type { ErrorReportInfo } from "./ErrorReporter";

```
