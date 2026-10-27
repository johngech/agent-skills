# API Envelope Unwrapping (`unwrap.ts`)

Located at: `src/api/unwrap.ts`

## Purpose

The NestJS backend wraps every HTTP response in a standardized enterprise envelope:
```json
{
  "success": true,
  "timestamp": "2026-10-27T12:00:00.000Z",
  "path": "/api/v1/members",
  "message": "Resource retrieved successfully",
  "data": { ... }
}
```

In addition, authentication endpoints nest access tokens under `data.token.accessToken`.

The `unwrap.ts` module provides centralized extraction functions so call sites do not manually navigate or guess the payload nesting.

---

## Implementation

```typescript
/** The envelope every endpoint returns. */
interface Envelope<T> {
  data: T;
}

/** Unwrap the `data` field of a response envelope. */
export function unwrapData<T>(response: Envelope<T> | T): T {
  if (
    typeof response === "object" &&
    response !== null &&
    "data" in response &&
    (response as Envelope<T>).data !== undefined
  ) {
    return (response as Envelope<T>).data;
  }
  return response as T;
}

export interface TokenPair {
  accessToken: string;
  tokenType: string;
  expiresIn: number;
}

export interface AuthPayload {
  token: TokenPair;
  email: string;
  userId: string;
}

/**
 * Read the access token out of a login/refresh response.
 *
 * Throws when the token is missing or not a string, so a contract change
 * surfaces as a loud error instead of an `undefined` that silently becomes
 * `Bearer undefined` on the next request.
 */
export function extractAccessToken(response: unknown): string {
  const payload = unwrapData<AuthPayload>(response as Envelope<AuthPayload>);
  const accessToken = payload?.token?.accessToken;

  if (typeof accessToken !== "string" || accessToken.trim() === "") {
    throw new Error(
      "Auth response did not include data.token.accessToken. Check the API response shape.",
    );
  }
  return accessToken;
}
```

---

## Architectural Rules

1. **Assert Contract Loudly**: If the backend response shape changes unexpectedly, `extractAccessToken` fails immediately rather than generating corrupted `Bearer undefined` headers.
2. **Defensive Passthrough**: `unwrapData` tolerates both already-unwrapped objects and wrapped envelopes, preventing breaking changes when calling endpoints with different return types.
