# Authentication Store (`auth.ts`)

Located at: `src/stores/auth.ts`

## Purpose

Manages the authenticated user profile and session state using `@tanstack/store` and `@tanstack/react-store`.

### Critical Security Rule
The access token is **deliberately absent** from this store. Access tokens are stored exclusively in memory via `tokenManager` so they are never duplicated or serialized to disk (e.g., localStorage or IndexedDB). Only the user profile (`CurrentUser`) is persisted locally for offline read access.

---

## Architecture & Implementation

```typescript
import { useSelector } from "@tanstack/react-store";
import { Store } from "@tanstack/store";
import { tokenManager } from "@/api/token-manager";
import { extractAccessToken } from "@/api/unwrap";
import { purgePersistedCache } from "@/integrations/tanstack-query/persistence";
import { isOnline } from "@/lib/utils/online";
import { authApi } from "@/routes/auth/-api";
import type { CurrentUser, LoginInput } from "@/routes/auth/-types";

export interface AuthState {
  user: CurrentUser | null;
  isAuthenticated: boolean;
}

const USER_STORAGE_KEY = "currentUser";

export const authStore = new Store<AuthState>({
  user: readCachedUser(),
  isAuthenticated: Boolean(readCachedUser() && tokenManager.hasAccessToken()),
});
```

---

## Cross-Tab Synchronization via BroadcastChannel

When a user signs out in one tab, `tokenManager` broadcasts `null` across all tabs via `BroadcastChannel`. The store listens to this event to ensure sibling tabs do not remain in an authenticated state or read stale sensitive data from the offline IndexedDB cache:

```typescript
tokenManager.subscribe((token) => {
  if (token === null && authStore.state.isAuthenticated) {
    clearSession();
  }
});

export function clearSession(): void {
  writeCachedUser(null);
  authStore.setState(() => ({ user: null, isAuthenticated: false }));
}
```

---

## Session Restoration (`ensureSession`)

Called by router `beforeLoad` guards before evaluating protected routes. When a user refreshes the page, the in-memory token is empty; `ensureSession` invokes the refresh endpoint using the secure `httpOnly` refresh cookie to rebuild the memory token:

```typescript
async function ensureSession(): Promise<boolean> {
  if (authStore.state.isAuthenticated) return true;

  try {
    const response = await authApi.refreshToken();
    const token = extractAccessToken(response);
    tokenManager.setAccessToken(token);

    if (!authStore.state.user) {
      await fetchUser();
    }
    return true;
  } catch {
    clearSession();
    return false;
  }
}
```

---

## React Hook Usage

```typescript
export function useAuthStore<T>(selector: (state: AuthState) => T): T {
  return useSelector(authStore, selector);
}

// In components:
const user = useAuthStore((s) => s.user);
const isAuthenticated = useAuthStore((s) => s.isAuthenticated);
```
