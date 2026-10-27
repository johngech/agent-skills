# PaginatedPermissionTable (`PaginatedPermissionTable.tsx`)

## Purpose

A presentation table component for browsing and toggling permissions. Connects to `DataTable` and server-side pagination while leaving state management to the parent container.

---

## Implementation

```tsx
import { useState, useMemo } from "react";
import { useQuery } from "@tanstack/react-query";
import { Switch } from "@/components/ui/switch";
import { permissionsQueryOptions } from "@/routes/settings/access/-queries";
import { DataTable } from "@/components/data-table/data-table";
import type { Permission } from "@/routes/settings/access/-types";
import type { PaginationState, ColumnDef } from "@tanstack/react-table";

interface PaginatedPermissionTableProps {
  selectedIds: number[];
  onToggle: (id: number, checked: boolean) => void;
}

export function PaginatedPermissionTable({
  selectedIds,
  onToggle,
}: Readonly<PaginatedPermissionTableProps>) {
  const [search, setSearch] = useState("");
  const [pagination, setPagination] = useState<PaginationState>({
    pageIndex: 0,
    pageSize: 10,
  });

  const params = useMemo(
    () => ({
      page: pagination.pageIndex + 1,
      limit: pagination.pageSize,
      search: search || undefined,
    }),
    [search, pagination],
  );

  const { data: permissionsData, isLoading } = useQuery(
    permissionsQueryOptions(params),
  );

  const selectedSet = useMemo(() => new Set(selectedIds), [selectedIds]);

  const columns: ColumnDef<Permission, unknown>[] = useMemo(
    () => [
      {
        accessorKey: "name",
        header: "Name",
        cell: ({ row }) => (
          <div className="font-medium">{row.original.name}</div>
        ),
      },
      {
        accessorKey: "codename",
        header: "Code Name",
        cell: ({ row }) => (
          <div className="text-xs text-muted-foreground font-mono">
            {row.original.codename}
          </div>
        ),
      },
      {
        id: "actions",
        header: "Granted",
        cell: ({ row }) => (
          <Switch
            checked={selectedSet.has(row.original.id)}
            onCheckedChange={(checked) => onToggle(row.original.id, checked)}
          />
        ),
      },
    ],
    [selectedSet, onToggle],
  );

  return (
    <DataTable
      columns={columns}
      data={permissionsData?.results ?? []}
      searchKey="name"
      searchPlaceholder="Search permissions..."
      onSearch={setSearch}
      isLoading={isLoading}
      pageCount={permissionsData?.totalPages ?? 0}
      manualPagination
      emptyMessage="No permissions found."
    />
  );
}
```
