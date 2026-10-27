# ErrorFallback.tsx

**Why this pattern?**
> The default fallback UI displayed when a page or section crashes.

## Implementation

```tsx
import { AlertTriangle, Home, RefreshCcw } from "lucide-react";
import { Button } from "@/components/ui";

/**
 * Full presentational crash fallback used by page/route/application
 * boundaries. Purely presentational — no logging, no API calls, no business
 * logic. Error details are reported by the boundary, never shown to users.
 */
export interface ErrorFallbackProps {
  error: Error;
  resetErrorBoundary: () => void;
  title?: string;
  description?: string;
  /** When provided, renders a "Go Home" action. */
  onGoHome?: () => void;
}

export function ErrorFallback({
  title = "Something went wrong!",
  description = "An unexpected error occurred. Please try again!",
  resetErrorBoundary,
  onGoHome,
}: ErrorFallbackProps) {
  return (
    <div
      role="alert"
      className="flex min-h-full flex-col items-center justify-center gap-6 bg-background px-6 py-16 text-center"
    >
      <AlertTriangle className="h-12 w-12 text-red-600" aria-hidden="true" />

      <div className="space-y-2">
        <h1 className="text-2xl font-semibold  text-red-300">{title}</h1>
        <p className="mx-auto max-w-md text-sm leading-relaxed text-red-600">
          {description}
        </p>
      </div>

      <div className="flex flex-wrap items-center justify-center gap-3">
        <Button variant="primary" onClick={resetErrorBoundary}>
          <RefreshCcw className="h-4 w-4" aria-hidden="true" />
          Try Again
        </Button>

        {onGoHome && (
          <Button variant="outline" onClick={onGoHome}>
            <Home className="h-4 w-4" aria-hidden="true" />
            Go Home
          </Button>
        )}

        <Button variant="outline" onClick={() => window.location.reload()}>
          Reload
        </Button>
      </div>
    </div>
  );
}

```
