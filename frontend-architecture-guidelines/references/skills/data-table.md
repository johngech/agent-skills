---
name: project-data-table
description: How data grids and tables are built in this project. Use when creating or modifying any list or admin view that displays tabular data. Enforces the reuse of the DataTable abstraction and TanStack Table column definitions.
---

# Project Data Table Architecture

This project uses **TanStack Table v8** via a heavily abstracted `DataTable` component. 

Agents **MUST NEVER** build raw HTML `<table>` elements in feature code. Agents **MUST NEVER** manually invoke `useReactTable` directly in feature code unless modifying the `DataTable` wrapper itself.

## 1. The `DataTable` Component

The `DataTable` abstraction located at `@/components/data-table/DataTable` handles:

- Rendering the table UI (`TableHeader`, `TableBody`, etc.)
- Pagination (Manual/Server-Side and Client-Side)
- Search and Filtering UI
- Loading states via Skeletons
- Empty states

### Basic Usage

To render a table, pass your memoized `columns` and your `data` array to the `DataTable` component:

```tsx
import { DataTable } from "@/components/data-table/DataTable";
import { columns } from "./columns";

export function FeatureList({ data, isLoading }) {
  return (
    <DataTable
      columns={columns}
      data={data}
      isLoading={isLoading}
      emptyMessage="No items found."
    />
  );
}
```

## 2. Defining Columns (`ColumnDef`)

Columns must be defined using TanStack Table's `ColumnDef`. 
**Golden Rule:** Always define columns outside the component render cycle or memoize them with `useMemo` to prevent the table from needlessly re-rendering rows on unrelated state changes.

```tsx
// app/(dashboard)/{featureName}/_components/columns.tsx
import { ColumnDef } from "@tanstack/react-table";
import { FeatureEntity } from "../types";
import { Badge } from "@/components/ui/Badge";
import { Button } from "@/components/ui/Button";

export const columns: ColumnDef<FeatureEntity>[] = [
  {
    accessorKey: "title",
    header: "Title",
    cell: ({ row }) => <span className="font-medium">{row.original.title}</span>,
  },
  {
    accessorKey: "status",
    header: "Status",
    cell: ({ row }) => (
      <Badge variant={row.original.isActive ? "success" : "secondary"}>
        {row.original.isActive ? "Active" : "Inactive"}
      </Badge>
    ),
  },
  {
    id: "actions",
    cell: ({ row }) => (
      <Button variant="outline" size="sm" onClick={() => edit(row.original.id)}>
        Edit
      </Button>
    ),
  },
];
```

## 3. Server-Side Pagination and Filtering

For large datasets (which is the default in the admin panel), connect the `DataTable` state changes to your URL search params, which then drive your TanStack Query loaders.

```tsx
<DataTable
  columns={columns}
  data={queryData.results}
  pageCount={queryData.total_pages}
  
  // Search
  searchPlaceholder="Search features..."
  searchKey="search"
  onSearch={(value) => updateSearchParams({ search: value })}
  
  // Filtering
  filters={[
    {
      id: "status",
      label: "Status",
      options: [
        { label: "Active", value: "active" },
        { label: "Inactive", value: "inactive" }
      ]
    }
  ]}
  filterValues={{ status: currentStatusFilter }}
  onFilterChange={(id, value) => updateSearchParams({ [id]: value })}
  
  // Pagination
  manualPagination={true}
  pagination={{
    pageIndex: currentPage - 1, // TanStack Table is 0-indexed
    pageSize: 20
  }}
  onPaginationChange={(updater) => {
    // Handle pagination state update and sync to URL
  }}
  
  isLoading={isLoading}
/>
```

## Golden Rules for Data Tables

1. **Never use `<table>`, `<tr>`, `<td>`**: Always use `DataTable`.
2. **Memoize Columns**: Keep `ColumnDef` definitions outside of React components or wrap them in `useMemo`.
3. **Drive State from URL**: For admin CMS lists, table pagination and filtering state should be kept in sync with URL Search Parameters.

## Core Implementation References

Review the underlying components that power this abstraction to understand how they encapsulate TanStack Table:

**Data Table Abstraction:**

- [DataTable.tsx](../data-table/data-table.md) (The primary wrapper component)
- [DataTablePagination.tsx](../data-table/data-table-pagination.md) (Pagination controls)
- [DataTableSearch.tsx](../data-table/data-table-search.md) (Search input integration)
- [DataTableFilter.tsx](../data-table/data-table-filter.md) (Dropdown filter integration)

**Base UI Table Components:**

- [Table.tsx](../data-table/Table.md) (The raw styled table primitives)
