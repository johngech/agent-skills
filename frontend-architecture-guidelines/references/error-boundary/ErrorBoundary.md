# ErrorBoundary.tsx

**Why this pattern?**
> The core class-based error boundary component that implements getDerivedStateFromError and componentDidCatch.

## Implementation

```tsx
import {
  Component,
  type ErrorInfo,
  type PropsWithChildren,
  type ReactNode,
} from "react";
import { ErrorFallback } from "./ErrorFallback";
import { reportError } from "./ErrorReporter";

/**
 * Reusable error boundary — the ONLY place a class-based boundary may exist
 * in this project (see project-error-boundary-architecture skill).
 *
 * Catches render / lifecycle / constructor errors in the subtree and swaps in
 * a fallback. Recovery happens via `resetErrorBoundary` (retry button) or
 * automatically when `resetKeys` change (e.g. navigation).
 */
export interface ErrorBoundaryProps extends PropsWithChildren {
  /** Custom fallback. Prefer the default ErrorFallback. */
  fallback?:
    ReactNode | ((error: Error, resetErrorBoundary: () => void) => ReactNode);
  /** When any of these values change after an error, the boundary resets. */
  resetKeys?: unknown[];
  /** Called after a manual reset (e.g. to reset failed queries). */
  onReset?: () => void;
  /** Called when an error is caught (centralize reporting here). */
  onError?: (error: Error, info: ErrorInfo) => void;
}

interface ErrorBoundaryState {
  error: Error | null;
  resetKey: number;
}

function normalizeError(error: unknown): Error {
  if (error instanceof Error) return error;
  return new Error(
    typeof error === "string" ? error : "An unexpected error occurred",
  );
}

function hasArrayChanged(a: unknown[] = [], b: unknown[] = []): boolean {
  return a.length !== b.length || a.some((item, index) => item !== b[index]);
}

export class ErrorBoundary extends Component<
  ErrorBoundaryProps,
  ErrorBoundaryState
> {
  state: ErrorBoundaryState = { error: null, resetKey: 0 };

  static getDerivedStateFromError(error: unknown): Partial<ErrorBoundaryState> {
    return { error: normalizeError(error) };
  }

  componentDidCatch(error: Error, info: ErrorInfo): void {
    this.props.onError?.(error, info);
    reportError(error, { componentStack: info.componentStack });
  }

  componentDidUpdate(prevProps: ErrorBoundaryProps): void {
    if (
      this.state.error !== null &&
      hasArrayChanged(prevProps.resetKeys, this.props.resetKeys)
    ) {
      this.resetErrorBoundary();
    }
  }

  resetErrorBoundary = (): void => {
    this.setState((state) => ({ error: null, resetKey: state.resetKey + 1 }));
    this.props.onReset?.();
  };

  render(): ReactNode {
    const { error } = this.state;

    if (error !== null) {
      const { fallback } = this.props;
      if (typeof fallback === "function") {
        return fallback(error, this.resetErrorBoundary);
      }
      if (fallback !== undefined) {
        return fallback;
      }
      return (
        <ErrorFallback
          error={error}
          resetErrorBoundary={this.resetErrorBoundary}
        />
      );
    }

    return this.props.children;
  }
}

```
