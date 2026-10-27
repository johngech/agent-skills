---
name: project-rbac-ui
description: How reusable roles, groups, and permissions assignment UI is built in this project. Use when creating or modifying UIs that require paginated search and selection (e.g., assigning permissions to roles, or roles to users). Details the split between base presentation tables, form wrappers, and dialog wrappers.
---

# Project RBAC UI Architecture

This project implements a highly reusable pattern for managing many-to-many relationships (like Roles, Groups, and Permissions) that require pagination and search.

Because datasets like Permissions can be massive, we cannot simply load all of them into a standard HTML `<select multiple>`. Instead, we use a paginated data table.

The architecture is split into three layers:

1. **The Base Presentation Table** (e.g., `PaginatedPermissionTable`)
2. **The Form Integration Wrapper** (e.g., `FormPermissionTable`)
3. **The Dialog/Modal Wrapper** (e.g., `RolePermissionsDialog`)

Agents **MUST** follow this split pattern when building similar assignment interfaces.

## 1. The Base Presentation Table

The base component handles data fetching, pagination state, and search state internally, but it **inverts control** of the selection state to the parent via `selectedIds` and `onToggle` props.

```tsx
// app/(dashboard)/permissions/_components/paginated-permission-table.tsx
import { useState, useMemo } from "react";
import { useQuery } from "@tanstack/react-query";
import { DataTable } from "@/components/data-table/DataTable";
import { Switch } from "@/components/ui/Switch";
import { permissionsQueryOptions } from "../queries";

interface Props {
  selectedIds: number[];
  onToggle: (id: number, checked: boolean) => void;
}

export function PaginatedPermissionTable({ selectedIds, onToggle }: Props) {
  const [search, setSearch] = useState("");
  const [pagination, setPagination] = useState({ pageIndex: 0, pageSize: 10 });
  
  // 1. Internal state drives the query parameters
  const params = useMemo(() => ({
    page: pagination.pageIndex + 1,
    page_size: pagination.pageSize,
    search: search || undefined,
  }), [search, pagination]);

  const { data, isLoading } = useQuery(permissionsQueryOptions(params));
  const selectedSet = useMemo(() => new Set(selectedIds), [selectedIds]);

  // 2. Parent controls the Switch state via selectedSet
  const columns = [
    { accessorKey: "name", header: "Name" },
    {
      id: "actions",
      cell: ({ row }) => (
        <Switch
          checked={selectedSet.has(row.original.id)}
          onCheckedChange={(checked) => onToggle(row.original.id, checked)}
        />
      ),
    },
  ];

  // 3. Delegate rendering to the DataTable abstraction
  return (
    <DataTable
      columns={columns}
      data={data?.data ?? []}
      searchKey="name"
      onSearch={setSearch}
      pagination={pagination}
      pageCount={data?.pagination?.total_pages ?? 0}
      onPaginationChange={setPagination}
      manualPagination
      isLoading={isLoading}
    />
  );
}
```

## 2. The Form Integration Wrapper

When this paginated table needs to be used inside a `react-hook-form` (e.g., when creating a new Role), we build a wrapper that uses `Controller` to bridge React Hook Form's `field.value` and `field.onChange` to the base table's props.

```tsx
// app/(dashboard)/permissions/_components/form-permission-table.tsx
import { useFormContext, Controller } from "react-hook-form";
import { PaginatedPermissionTable } from "./PaginatedPermissionTable";

export function FormPermissionTable({ name }: { name: string }) {
  const { control } = useFormContext();

  return (
    <Controller
      name={name}
      control={control}
      render={({ field }) => {
        const value: number[] = field.value || [];

        const handleToggle = (id: number, checked: boolean) => {
          const set = new Set(value);
          if (checked) set.add(id);
          else set.delete(id);
          field.onChange(Array.from(set));
        };

        return (
          <PaginatedPermissionTable
            selectedIds={value}
            onToggle={handleToggle}
          />
        );
      }}
    />
  );
}
```

## 3. The Dialog/Modal Wrapper

When this paginated table is used standalone inside a Modal (e.g., clicking "Manage Permissions" on an existing role), we build a wrapper that holds local state and triggers an API mutation on save.

```tsx
// app/(dashboard)/roles/_components/role-permissions-dialog.tsx
import { useState, useEffect } from "react";
import { Dialog, DialogContent } from "@/components/ui/Dialog";
import { Button } from "@/components/ui/Button";
import { PaginatedPermissionTable } from "@/features/permissions/components/PaginatedPermissionTable";
import { useAssignPermissions } from "../queries";

export function RolePermissionsDialog({ role, isOpen, onClose }) {
  const mutation = useAssignPermissions(role.id);
  
  // 1. Local state to track selections before saving
  const [selected, setSelected] = useState<Set<number>>(new Set());

  // 2. Hydrate state when dialog opens
  useEffect(() => {
    if (isOpen) setSelected(new Set(role.permissions || []));
  }, [isOpen, role]);

  const handleToggle = (id: number, checked: boolean) => {
    const next = new Set(selected);
    if (checked) next.add(id);
    else next.delete(id);
    setSelected(next);
  };

  const handleSave = async () => {
    await mutation.mutateAsync(Array.from(selected));
    onClose();
  };

  return (
    <Dialog open={isOpen} onClose={onClose}>
      <DialogContent>
        {/* 3. Render base table with local state */}
        <PaginatedPermissionTable
          selectedIds={Array.from(selected)}
          onToggle={handleToggle}
        />
        <Button onClick={handleSave} isLoading={mutation.isPending}>Save</Button>
      </DialogContent>
    </Dialog>
  );
}
```

## Golden Rules

1. **Never Fetch Inside Form or Dialog Wrappers**: Data fetching and pagination logic MUST stay inside the Base Presentation Table.
2. **Never Manage State Inside the Base Table**: Selection state (`selectedIds`) MUST be controlled by the parent. The Base Table is "dumb" regarding what is selected.
3. **Always Use `Set` for Selections**: When tracking arrays of IDs, convert them to a `Set` for O(1) lookups during render, especially for large paginated datasets.

## Core Implementation References

- [PaginatedPermissionTable.tsx](../guards/PaginatedPermissionTable.md)
- [FormPermissionTable.tsx](../guards/FormPermissionTable.md)
- [RolePermissionsDialog.tsx](../guards/RolePermissionsDialog.md)
