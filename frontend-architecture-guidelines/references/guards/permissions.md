# Permission Catalog (`permissions.ts`)

Located at: `src/config/permissions.ts`

## Purpose

The frontend mirror of the backend's seeded permission catalog (`*PermissionSeeder` classes).

Every permission codename is grouped by business domain (members, savings, charges, users, system). UI components reference these constants directly rather than using magic strings.

---

## Domain Permission Groups

```typescript
export const MEMBER_PERMISSIONS = {
  view: "view_member",
  list: "list_members",
  add: "add_member",
  change: "change_member",
  delete: "delete_member",
  activate: "activate_member",
  approve: "approve_member",
  reject: "reject_member",
  withdraw: "withdraw_member",
  close: "close_member",
  suspend: "suspend_member",
} as const;

export const USER_PERMISSIONS = {
  view: "view_user",
  list: "list_users",
  add: "add_user",
  change: "change_user",
  delete: "delete_user",
  grantPermission: "grant_permission",
} as const;

export const SAVINGS_PRODUCT_PERMISSIONS = {
  view: "view_savingsproduct",
  list: "list_savingsproduct",
  add: "add_savingsproduct",
  change: "change_savingsproduct",
  delete: "delete_savingsproduct",
} as const;

export const SAVINGS_ACCOUNT_PERMISSIONS = {
  view: "view_savingsaccount",
  list: "list_savingsaccount",
  add: "add_savingsaccount",
  change: "change_savingsaccount",
  delete: "delete_savingsaccount",
  approve: "approve_savingsaccount",
  activate: "activate_savingsaccount",
  reject: "reject_savingsaccount",
  withdraw: "withdraw_savingsaccount",
  close: "close_savingsaccount",
  deposit: "deposit_into_savingsaccount",
  withdrawFrom: "withdraw_from_savingsaccount",
  postInterest: "post_interest_savingsaccount",
  reverse: "reverse_savingsaccount",
  charge: "charge_savingsaccount",
  holdFunds: "hold_funds_savingsaccount",
  mature: "mature_savingsaccount",
  preclose: "preclose_savingsaccount",
  payInstallment: "pay_installment_savingsaccount",
} as const;

export const CHARGE_PERMISSIONS = {
  view: "view_charge",
  list: "list_charge",
  add: "add_charge",
  change: "change_charge",
  delete: "delete_charge",
} as const;

export const SYSTEM_PERMISSIONS = {
  viewPermission: "view_permission",
  managePermission: "manage_permission",
  viewGroup: "view_group",
  manageGroup: "manage_group",
  viewResourcePermission: "view_resourcepermission",
  grantPermission: "grant_permission",
} as const;
```

---

## Aggregated Catalogs & Dashboard Access

```typescript
export const ALL_PERMISSION_CODENAMES: readonly string[] = [
  ...new Set([
    ...ALL_MEMBER_PERMISSIONS,
    ...ALL_USER_PERMISSIONS,
    ...ALL_SAVINGS_PERMISSIONS,
    ...ALL_CHARGE_PERMISSIONS,
    ...ALL_SYSTEM_PERMISSIONS,
  ]),
];

export const DASHBOARD_ACCESS_PERMISSIONS: readonly string[] = [
  MEMBER_PERMISSIONS.list,
  MEMBER_PERMISSIONS.view,
  SAVINGS_PRODUCT_PERMISSIONS.list,
  SAVINGS_PRODUCT_PERMISSIONS.view,
  SAVINGS_ACCOUNT_PERMISSIONS.list,
  SAVINGS_ACCOUNT_PERMISSIONS.view,
  CHARGE_PERMISSIONS.list,
  CHARGE_PERMISSIONS.view,
  USER_PERMISSIONS.list,
  USER_PERMISSIONS.view,
  SYSTEM_PERMISSIONS.viewPermission,
  SYSTEM_PERMISSIONS.viewGroup,
];
```

---

## Architectural Rules

1. **Strict Sync with Backend**: Any change to backend permission seeders must be mirrored immediately in `src/config/permissions.ts`.
2. **Superuser Bypass**: Superusers always evaluate to `true` for all checks across frontend and backend.
