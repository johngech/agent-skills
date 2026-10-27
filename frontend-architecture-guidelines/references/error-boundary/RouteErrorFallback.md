# RouteErrorFallback.tsx

**Why this pattern?**
> The specific fallback used by React Router errorElement to catch routing, loader, and action errors.

## Implementation

```tsx
import { useQueryErrorResetBoundary } from "@tanstack/react-query";
import {
  isRouteErrorResponse,
  useLocation,
  useNavigate,
  useRouteError,
} from "react-router";
import { ErrorFallback } from "./ErrorFallback";

interface RouteErrorCopy {
  title: string;
  description: string;
}

function getRouteErrorCopy(error: unknown): RouteErrorCopy {
  if (isRouteErrorResponse(error)) {
    if (error.status === 404) {
      return {
        title: "Page not found",
        description:
          "The page you are looking for doesn't exist or has been moved.",
      };
    }
    return {
      title: `${error.status} ${error.statusText}`.trim(),
      description: "The page could not be loaded. Please try again.",
    };
  }
  return {
    title: "Something went wrong",
    description: "An unexpected error occurred. Please try again.",
  };
}

function toError(error: unknown): Error {
  if (error instanceof Error) return error;
  if (isRouteErrorResponse(error)) {
    return new Error(`${error.status} ${error.statusText}`.trim());
  }
  return new Error("Route error");
}

/**
 * Fallback wired via react-router's `errorElement`. Handles both loader/action
 * rejections (404, 403, ...) and render crashes for every route beneath it.
 * Retry resets failed queries and re-navigates to re-run loaders.
 */
export function RouteErrorFallback() {
  const error = useRouteError();
  const navigate = useNavigate();
  const location = useLocation();
  const { reset: resetQueryErrors } = useQueryErrorResetBoundary();

  const { title, description } = getRouteErrorCopy(error);

  const handleRetry = () => {
    resetQueryErrors();
    navigate(`${location.pathname}${location.search}`, { replace: true });
  };

  return (
    <ErrorFallback
      error={toError(error)}
      resetErrorBoundary={handleRetry}
      title={title}
      description={description}
      onGoHome={() => navigate("/")}
    />
  );
}

```
