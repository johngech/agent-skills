# ErrorReporter.ts

**Why this pattern?**
> Centralized error reporting utility for logging to external services (e.g., Sentry).

## Implementation

```typescript
/**
 * Centralized runtime error reporting.
 *
 * All error boundaries report through this module so reporting can be
 * swapped (Sentry, Azure Monitor, ...) without touching UI code.
 * UI components never call this directly — boundaries do.
 */
export interface ErrorReportInfo {
  componentStack?: string | null;
}

export function reportError(error: unknown, info?: ErrorReportInfo): void {
  console.error("[error-boundary]", error);

  if (info?.componentStack) {
    console.error("[error-boundary] component stack:", info.componentStack);
  }
}

```
