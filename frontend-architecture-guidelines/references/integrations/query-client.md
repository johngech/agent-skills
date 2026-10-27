# TanStack Query Client Configuration (`root-provider.tsx`)

Located at: `src/integrations/tanstack-query/root-provider.tsx`

## Purpose

Configures the central `QueryClient` instance with defaults optimized for enterprise reliability, offline-first resilience, and zero-flash navigation.

---

## Configuration & Architecture

```typescript
import { QueryClient } from "@tanstack/react-query";
import { isOnline } from "@/lib/utils/online";
import { startQueryPersistence } from "./persistence";

/** Twenty-four hours. */
const DAY_MS = 1000 * 60 * 60 * 24;

export function getContext() {
  const queryClient = new QueryClient({
    defaultOptions: {
      queries: {
        // 1. Instant navigation without repeated network requests
        staleTime: 30_000,

        // 2. Cache lifetime exceeds longest expected offline session
        gcTime: DAY_MS,

        // 3. Serve cached data immediately, then revalidate in background
        networkMode: "offlineFirst",

        // 4. Intelligent retry predicate
        retry: (failureCount, error) => {
          // Never retry when offline
          if (!isOnline()) return false;
          
          // Never retry 4xx client errors (400, 401, 403, 404, 422)
          const status = (error as { status?: number } | null)?.status;
          if (status !== undefined && status < 500) return false;

          // Retry 5xx server errors and network drops up to 2 times
          return failureCount < 2;
        },
      },
      mutations: {
        // Mutations pause when offline rather than failing immediately
        networkMode: "online",
        retry: false,
      },
    },
  });

  // Early persistence setup before router renders
  startQueryPersistence(queryClient);

  return { queryClient };
}
```

---

## Architectural Rules

1. **`networkMode: "offlineFirst"`**: Never let offline state throw an uncached query into an error state.
2. **Never Retry 4xx Errors**: Retrying a 400 Bad Request or 404 Not Found hammers the server without benefit. Only retry network errors and 5xx server errors.
3. **No Retries on Mutations**: Mutations should fail fast to avoid duplicate operations (e.g. creating two loan applications).
