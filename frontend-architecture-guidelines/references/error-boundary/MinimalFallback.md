# MinimalFallback.tsx

**Why this pattern?**
> A smaller fallback UI intended for widgets or minor components so they do not take up the whole page when crashing.

## Implementation

```tsx
import { RefreshCcw } from "lucide-react";
import { Button } from "@/components/ui";

/**
 * Compact crash fallback for widget/section-level boundaries. Keeps the
 * surrounding layout intact while isolating the failed region.
 */
export interface MinimalFallbackProps {
  error: Error;
  resetErrorBoundary: () => void;
  title?: string;
}

export function MinimalFallback({
  title = "This section is unavailable",
  resetErrorBoundary,
}: MinimalFallbackProps) {
  return (
    <div
      role="alert"
      className="flex flex-col items-center justify-center gap-3 rounded-[8px] border border-border bg-background p-6 text-center"
    >
      <p className="text-sm font-semibold text-foreground">{title}</p>
      <Button variant="outline" size="sm" onClick={resetErrorBoundary}>
        <RefreshCcw className="h-3.5 w-3.5" aria-hidden="true" />
        Retry
      </Button>
    </div>
  );
}

```
