# Permission Element Gating (`Can.tsx`)

Located at: `src/components/guards/Can.tsx`

## Purpose

`<Can>` is a declarative UI element wrapper for Role-Based Access Control (RBAC). It evaluates whether the authenticated user holds specific permissions before rendering action buttons, links, or sections.

---

## Implementation

```tsx
import type { ReactNode } from "react";
import { useAuthStore } from "@/stores/auth";

interface CanProps {
  /** Single permission codename or an array of permissions (any-of semantics). */
  perform: string | readonly string[];
  /** Optional fallback to render when permission check fails. */
  fallback?: ReactNode;
  children: ReactNode;
}

export function Can({ perform, fallback = null, children }: CanProps) {
  const user = useAuthStore((s) => s.user);

  if (!user) return fallback;
  if (user.isSuperuser) return <>{children}</>;

  const granted = new Set(user.permissions ?? []);
  const required = Array.isArray(perform) ? perform : [perform];

  const hasAccess = required.some((perm) => granted.has(perm));
  return hasAccess ? <>{children}</> : <>{fallback}</>;
}
```

---

## Usage Examples

### 1. Gating Action Buttons
```tsx
<Can perform={MEMBER_PERMISSIONS.add}>
  <Button asChild>
    <Link to="/dashboard/members/new">Create Member</Link>
  </Button>
</Can>
```

### 2. Any-Of Permissions
```tsx
<Can perform={[MEMBER_PERMISSIONS.list, MEMBER_PERMISSIONS.view]}>
  <DataTable columns={columns} data={data} />
</Can>
```

### 3. Fallback Content
```tsx
<Can
  perform={USER_PERMISSIONS.change}
  fallback={<p className="text-muted-foreground">Contact admin to modify roles.</p>}
>
  <EditRoleButton />
</Can>
```

---

## Golden Rules for `<Can>`

1. **UX-Only Gating**: `<Can>` hides or disables UI elements for better user experience. Real security is ALWAYS enforced on the backend API and via router `beforeLoad` guards.
2. **Never Check Role Names**: Never check `role === 'admin'`. Always check granular permission codenames (`perform={MEMBER_PERMISSIONS.delete}`).
