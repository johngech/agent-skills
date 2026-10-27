# Next.js App Router Architecture & Implementation Guide

This guide details the architecture for applications built with the **Next.js App Router**, covering React Server Components (RSC), Server Actions, leaf client boundaries, caching, and edge security.

## 1. Core Principles

1. **Server-First by Default**: Every component under `app/` is a React Server Component (RSC) unless marked with `'use client'`.
2. **Push Client Boundaries Down**: Keep `'use client'` at the leaves of the component tree. Pass Server Components as `children` or props to prevent bundle bloating.
3. **Colocation & Deletion-Safety**: Features are colocated under route segments (`app/(dashboard)/members/`). Deleting a feature folder cleanly removes it.
4. **Mutations via Server Actions**: Mutations are executed via `'use server'` functions with Zod validation and cache revalidation.
5. **Multi-Tier Caching**: Explicit cache management using `next: { tags: [...] }` and `revalidateTag(tag, 'max')` / `updateTag(tag)`.

---

## 2. Directory Structure

```text
src/
├── app/
│   ├── (auth)/
│   │   ├── login/page.tsx
│   │   └── register/page.tsx
│   ├── (dashboard)/
│   │   ├── layout.tsx                     # Authenticated app shell
│   │   └── members/
│   │       ├── page.tsx                   # Route page (Server Component)
│   │       ├── new/page.tsx               # Create page
│   │       ├── [id]/page.tsx              # Detail page
│   │       ├── api.ts                     # Server-side data fetching layer
│   │       ├── actions.ts                 # Server Actions ('use server')
│   │       ├── schemas.ts                 # Zod validation schemas
│   │       ├── types.ts                   # Domain types and interfaces
│   │       ├── queries.ts                 # TanStack Query options (if client state needed)
│   │       └── _components/               # Feature-private components
│   ├── layout.tsx                         # Root layout
│   └── globals.css
├── api/                                   # Shared API client & error utilities
├── components/                            # Shared UI kit (DataTable, Form, UI primitives)
│   ├── ui/
│   ├── forms/
│   ├── data-table/
│   └── error-boundary/
├── config/                                # Permissions, navigation config
├── lib/                                   # Shared helpers & validations
└── middleware.ts                          # Edge route protection
```

> [!NOTE]
> In Next.js, files prefixed with `_` (e.g. `_components/`) are ignored by the route compiler. Route groups like `(dashboard)` organize layouts without altering URL path segments.

---

## 3. Data Fetching in React Server Components

Server Components fetch data directly without `useEffect`, lifecycle hooks, or client state:

```tsx
// app/(dashboard)/members/page.tsx
import { Suspense } from "react";
import { getMembers } from "./api";
import { MembersTable } from "./_components/members-table";
import { TableSkeleton } from "@/components/ui/skeleton";

interface PageProps {
  searchParams: Promise<{ page?: string; search?: string }>;
}

export default async function MembersPage({ searchParams }: PageProps) {
  const { page = "1", search = "" } = await searchParams;
  const members = await getMembers({ page, search });

  return (
    <div className="space-y-6">
      <h1 className="text-2xl font-bold">Members Directory</h1>
      <Suspense fallback={<TableSkeleton />}>
        <MembersTable initialData={members} />
      </Suspense>
    </div>
  );
}
```

### Typed Data Access Layer (`api.ts`)
```typescript
// app/(dashboard)/members/api.ts
import { MemberEntity } from "./types";

const API_BASE = process.env.API_BASE_URL;

export async function getMembers(params?: Record<string, string>): Promise<MemberEntity[]> {
  const url = new URL("/members", API_BASE);
  if (params) {
    Object.entries(params).forEach(([k, v]) => {
      if (v) url.searchParams.set(k, v);
    });
  }

  const res = await fetch(url.toString(), {
    headers: {
      Authorization: `Bearer ${process.env.INTERNAL_SERVICE_TOKEN}`,
    },
    next: {
      tags: ["members"], // Tag for on-demand revalidation
    },
  });

  if (!res.ok) {
    throw new Error(`Failed to load members: ${res.statusText}`);
  }

  return res.json();
}
```

---

## 4. Server Actions & Mutations (`actions.ts`)

Mutations are executed using `'use server'`. Always validate with Zod and revalidate tags:

```typescript
// app/(dashboard)/members/actions.ts
'use server';

import { revalidateTag } from "next/cache";
import { redirect } from "next/navigation";
import { createMemberSchema } from "./schemas";
import { auth } from "@/lib/auth";

export interface ActionResult {
  success: boolean;
  error?: string;
  fieldErrors?: Record<string, string[]>;
}

export async function createMemberAction(
  _prevState: ActionResult | null,
  formData: FormData,
): Promise<ActionResult> {
  const session = await auth();
  if (!session?.user) {
    return { success: false, error: "Unauthorized" };
  }

  const raw = Object.fromEntries(formData.entries());
  const parsed = createMemberSchema.safeParse(raw);

  if (!parsed.success) {
    return {
      success: false,
      error: "Validation failed",
      fieldErrors: parsed.error.flatten().fieldErrors,
    };
  }

  const res = await fetch(`${process.env.API_BASE_URL}/members`, {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      Authorization: `Bearer ${session.accessToken}`,
    },
    body: JSON.stringify(parsed.data),
  });

  if (!res.ok) {
    return { success: false, error: "Failed to persist member" };
  }

  // Next.js recommended on-demand cache revalidation
  revalidateTag("members", "max");
  redirect("/dashboard/members");
}
```

---

## 5. Client Boundaries (`'use client'`)

Extract interactive controls into small leaf components:

```tsx
// app/(dashboard)/members/_components/create-member-form.tsx
'use client';

import { useActionState } from "react";
import { createMemberAction } from "../actions";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";

export function CreateMemberForm() {
  const [state, formAction, isPending] = useActionState(createMemberAction, null);

  return (
    <form action={formAction} className="space-y-4">
      {state?.error && <div className="text-red-500">{state.error}</div>}
      
      <div>
        <label htmlFor="firstName">First Name</label>
        <Input id="firstName" name="firstName" required />
        {state?.fieldErrors?.firstName && (
          <p className="text-xs text-red-500">{state.fieldErrors.firstName[0]}</p>
        )}
      </div>

      <Button type="submit" disabled={isPending}>
        {isPending ? "Saving..." : "Create Member"}
      </Button>
    </form>
  );
}
```

---

## 6. Edge Protection via Middleware

Route protection is handled at the network edge via `middleware.ts`:

```typescript
// middleware.ts
import { NextResponse } from "next/server";
import type { NextRequest } from "next/server";

export function middleware(request: NextRequest) {
  const token = request.cookies.get("session_token")?.value;
  const { pathname } = request.nextUrl;

  if (pathname.startsWith("/dashboard") && !token) {
    const loginUrl = new URL("/login", request.url);
    loginUrl.searchParams.set("returnTo", pathname);
    return NextResponse.redirect(loginUrl);
  }

  if (pathname.startsWith("/auth") && token) {
    return NextResponse.redirect(new URL("/dashboard", request.url));
  }

  return NextResponse.next();
}

export const config = {
  matcher: ["/dashboard/:path*", "/auth/:path*"],
};
```

---

## 7. Migration Mapping: TanStack Router SPA -> Next.js App Router

| Architectural Concept | TanStack Router SPA (`ethiomfi-frontend`) | Next.js App Router Target |
| :--- | :--- | :--- |
| Non-route co-location | `-api.ts`, `-queries.ts`, `-components/` | `api.ts`, `queries.ts`, `_components/` |
| Route protection | `beforeLoad` with `requireAccess` | `middleware.ts` + Server Component session checks |
| Data reading | `loader` via `queryClient.query` | Async Server Component `fetch` + `tags` |
| Mutations | `useMutation` hook calling `apiClient` | Server Actions (`'use server'`) |
| Cache invalidation | `queryClient.invalidateQueries(...)` | `revalidateTag('tag', 'max')` / `updateTag('tag')` |
| URL search params | `validateSearch: schema` with `.catch()` | `searchParams` prop in `page.tsx` + Zod |
