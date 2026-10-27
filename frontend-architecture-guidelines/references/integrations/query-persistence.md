# Query Cache Persistence (`persistence.ts` & `indexed-db-storage.ts`)

Located at: `src/integrations/tanstack-query/persistence.ts` and `src/integrations/tanstack-query/indexed-db-storage.ts`

## Purpose

Enables offline read resilience by serializing the TanStack Query cache to browser IndexedDB storage using `@tanstack/query-async-storage-persister`.

On application boot, cached queries are restored before route components render. On logout, the persisted cache is purged to protect sensitive financial records.

---

## IndexedDB Storage Layer

`indexed-db-storage.ts` provides an async key-value interface over native browser IndexedDB:

```typescript
import type { AsyncStorage } from "@tanstack/query-async-storage-persister";

export function createIndexedDbStorage(dbName = "ethiomfi_query_cache"): AsyncStorage {
  // Uses IndexedDB object store with get, set, and remove methods
  return {
    getItem: async (key: string) => { ... },
    setItem: async (key: string, value: string) => { ... },
    removeItem: async (key: string) => { ... },
  };
}
```

---

## Persistence Lifecycle (`persistence.ts`)

```typescript
import { persistQueryClient } from "@tanstack/react-query-persist-client";
import { createAsyncStoragePersister } from "@tanstack/query-async-storage-persister";
import type { QueryClient } from "@tanstack/react-query";
import { createIndexedDbStorage } from "./indexed-db-storage";

const storage = createIndexedDbStorage();
const persister = createAsyncStoragePersister({
  storage,
  key: "REACT_QUERY_OFFLINE_CACHE",
});

export function startQueryPersistence(queryClient: QueryClient) {
  persistQueryClient({
    queryClient,
    persister,
    maxAge: 1000 * 60 * 60 * 24, // 24 hours
    buster: "v1", // Cache invalidation key on breaking schema changes
  });
}

/** Purge cached data on logout */
export async function purgePersistedCache(): Promise<void> {
  await storage.removeItem("REACT_QUERY_OFFLINE_CACHE");
}
```

---

## Architectural Rules

1. **Purge on Sign-Out**: Always call `purgePersistedCache()` when the user logs out so personal and financial data is completely erased from the local device.
2. **Never Persist Sensitive Tokens**: The query persister only serializes query results, never JWT access tokens.
