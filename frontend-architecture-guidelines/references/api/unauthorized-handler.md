# Unauthorized Handler (`unauthorized-handler.ts`)

Located at: `src/api/unauthorized-handler.ts`

## Purpose

The API layer must not import the auth store, router, or feature modules, but it still has to notify the application when a session has expired (e.g., when a 401 response cannot be refreshed).

This registry pattern implements **Dependency Inversion** from Clean Architecture: the low-level HTTP transport invokes a registered callback without taking a hard dependency on high-level routing or state management.

---

## Implementation

```typescript
type UnauthorizedHandler = () => void;

let handler: UnauthorizedHandler | null = null;

export function setUnauthorizedHandler(fn: UnauthorizedHandler | null): void {
  handler = fn;
}

/** Invoke the registered handler, if any. Never throws. */
export function notifyUnauthorized(): void {
  try {
    handler?.();
  } catch (error) {
    console.error("Unauthorized handler failed", error);
  }
}
```

---

## Registration in Application Root

The application shell (e.g. `src/router.tsx` or `src/routes/__root.tsx`) wires this handler at startup:

```typescript
import { setUnauthorizedHandler } from "@/api/unauthorized-handler";
import { clearSession } from "@/stores/auth";
import { router } from "@/router";

setUnauthorizedHandler(() => {
  clearSession();
  const currentPath = window.location.pathname;
  router.navigate({
    to: "/auth/login",
    search: { returnTo: currentPath },
    replace: true,
  });
});
```

---

## Architectural Rules

1. **One-Way Dependency**: `src/api/` must NEVER import `src/stores/` or `src/routes/`.
2. **Never Throw**: `notifyUnauthorized()` catches all errors internally so network interceptors can finish rejecting cleanly.
