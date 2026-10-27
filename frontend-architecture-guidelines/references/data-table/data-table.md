---
title: data-table.tsx
description: DataTable component reference
---

```typescript
import type { FetchStatus } from "@tanstack/react-query";
import {
  type ColumnDef,
  columnFilteringFeature,
  columnVisibilityFeature,
  createCoreRowModel,
  createPaginatedRowModel,
  flexRender,
  type PaginationState,
  type ReactTable,
  type RowData,
  rowPaginationFeature,
  rowSelectionFeature,
  rowSortingFeature,
  type SortingState,
  tableFeatures,
  useTable,
} from "@tanstack/react-table";
import { cn } from "cn";
import { useState } from "react";
import { EmptyState } from "#/components/ui/empty-state";
import { Skeleton } from "#/components/ui/skeleton";
import {
  Table,
  TableBody,
  TableCell,
  TableHead,
  TableHeader,
  TableRow,
} from "#/components/ui/table";
import {
  DataTableFilter,
  type DataTableFilterOption,
} from "./data-table-filter";
import { DataTablePagination } from "./data-table-pagination";
import { DataTableSearch } from "./data-table-search";

export const dataTableFeatures = tableFeatures({
  coreRowModel: createCoreRowModel(),
  rowSelectionFeature,
  columnVisibilityFeature,
  rowPaginationFeature,
  rowSortingFeature,
  columnFilteringFeature,
  paginatedRowModel: createPaginatedRowModel(),
});

export type DataTableFeatures = typeof dataTableFeatures;

export interface DataTableFilterConfig {
  id: string;
  label: string;
  options: DataTableFilterOption[];
}

export interface DataTableProps<TData extends RowData> {
  columns: ColumnDef<DataTableFeatures, TData>[];
  data: TData[];
  searchKey?: string;
  searchPlaceholder?: string;
  filters?: DataTableFilterConfig[];
  filterValues?: Record<string, string>;
  onFilterChange?: (filterId: string, value: string) => void;
  onSearch?: (value: string) => void;
  isLoading?: boolean;
  /**
   * Pass the query's `fetchStatus` so a paused (offline) request is not
   * mistaken for an in-flight one. Defaults to `"fetching"`, which preserves
   * the previous behaviour for callers that do not supply it.
   */
  fetchStatus?: FetchStatus;
  emptyMessage?: string;
  /** Shown instead of `emptyMessage` when offline with nothing cached. */
  offlineMessage?: string;
  pagination?: PaginationState;
  pageCount?: number;
  onPaginationChange?: (pagination: PaginationState) => void;
  enableSorting?: boolean;
  sorting?: SortingState;
  onSortingChange?: (sorting: SortingState) => void;
  manualPagination?: boolean;
  className?: string;
  borderless?: boolean;
}

interface DataTableBodyProps<TData extends RowData> {
  table: ReactTable<DataTableFeatures, TData>;
  columns: ColumnDef<DataTableFeatures, TData>[];
  isLoading: boolean;
  /**
   * TanStack Query's fetch status. `"paused"` means the request is waiting on
   * a network that is not there, which is not the same as "loading".
   */
  fetchStatus: FetchStatus;
  emptyMessage: string;
  offlineMessage: string;
  pageSize: number;
}

function DataTableBody<TData extends RowData>({
  table,
  columns,
  isLoading,
  fetchStatus,
  emptyMessage,
  offlineMessage,
  pageSize,
}: Readonly<DataTableBodyProps<TData>>) {
  const isPaused = fetchStatus === "paused";
  const hasRows = (table.getRowModel().rows?.length ?? 0) > 0;

  // A paused query must not render skeletons: it is not coming back until the
  // network returns, so the placeholder would sit there indefinitely instead
  // of showing the rows that are already cached.
  if (isLoading && !isPaused) {
    return Array.from({ length: pageSize }).map((_, rowIndex) => (
      // biome-ignore lint/suspicious/noArrayIndexKey: skeletons are static
      <TableRow key={`skeleton-${rowIndex}`}>
        {columns.map((_, colIndex) => (
          // biome-ignore lint/suspicious/noArrayIndexKey: skeletons are static
          <TableCell key={`skeleton-${rowIndex}-${colIndex}`}>
            <Skeleton className="h-5 w-full" />
          </TableCell>
        ))}
      </TableRow>
    ));
  }

  if (hasRows) {
    return table.getRowModel().rows.map((row) => (
      <TableRow key={row.id} data-state={row.getIsSelected() && "selected"}>
        {row.getVisibleCells().map((cell) => (
          <TableCell key={cell.id}>
            {flexRender(cell.column.columnDef.cell, cell.getContext())}
          </TableCell>
        ))}
      </TableRow>
    ));
  }

  return (
    <TableRow>
      <TableCell colSpan={columns.length} className="p-0">
        <EmptyState
          title={isPaused ? offlineMessage : emptyMessage}
          className="border-0 bg-transparent rounded-none my-8"
        />
      </TableCell>
    </TableRow>
  );
}

export function DataTable<TData extends RowData>({
  columns,
  data,
  searchPlaceholder,
  filters,
  filterValues,
  onFilterChange,
  onSearch,
  isLoading = false,
  fetchStatus = "fetching",
  emptyMessage = "No results found.",
  offlineMessage = "You are offline and this data has not been loaded yet.",
  pagination,
  pageCount,
  onPaginationChange,
  enableSorting = false,
  sorting,
  onSortingChange,
  manualPagination = false,
  className,
  borderless = false,
}: Readonly<DataTableProps<TData>>) {
  const [internalPagination, setInternalPagination] = useState<PaginationState>(
    {
      pageIndex: 0,
      pageSize: 20,
    },
  );
  const [internalSorting, setInternalSorting] = useState<SortingState>([]);

  const currentPagination = pagination ?? internalPagination;
  const currentSorting = sorting ?? internalSorting;

  const handlePaginationChange = (
    updater: PaginationState | ((old: PaginationState) => PaginationState),
  ) => {
    const newPagination =
      typeof updater === "function" ? updater(currentPagination) : updater;
    if (onPaginationChange) {
      onPaginationChange(newPagination);
    } else {
      setInternalPagination(newPagination);
    }
  };

  const handleSortingChange = (
    updater: SortingState | ((old: SortingState) => SortingState),
  ) => {
    const newSorting =
      typeof updater === "function" ? updater(currentSorting) : updater;
    if (onSortingChange) {
      onSortingChange(newSorting);
    } else {
      setInternalSorting(newSorting);
    }
  };

  const table = useTable({
    features: dataTableFeatures,
    data,
    columns,
    rowCount:
      pageCount && pageCount > 0
        ? pageCount * currentPagination.pageSize
        : undefined,
    state: {
      pagination: currentPagination,
      sorting: enableSorting ? currentSorting : undefined,
    },
    onPaginationChange: handlePaginationChange,
    onSortingChange: enableSorting ? handleSortingChange : undefined,
    manualPagination,
  });

  const hasActiveFilters = filters && filters.length > 0;
  const hasSearch = !!onSearch;

  return (
    <div className={cn("space-y-4", className)}>
      {(hasSearch || hasActiveFilters) && (
        <div className="flex flex-col gap-4 sm:flex-row sm:items-end">
          {hasSearch && (
            <DataTableSearch
              placeholder={searchPlaceholder}
              onSearch={onSearch}
              className="w-full sm:max-w-sm"
            />
          )}
          {hasActiveFilters && (
            <div className="flex flex-wrap gap-3">
              {filters.map((filter) => (
                <DataTableFilter
                  key={filter.id}
                  label={filter.label}
                  options={filter.options}
                  value={filterValues?.[filter.id] ?? ""}
                  onChange={(value) => onFilterChange?.(filter.id, value)}
                />
              ))}
            </div>
          )}
        </div>
      )}

      <div
        className={cn(
          "overflow-x-auto bg-transparent",
          borderless ? "border-0" : "rounded-md border border-border shadow-sm",
        )}
      >
        <Table>
          <TableHeader>
            {table.getHeaderGroups().map((headerGroup) => (
              <TableRow key={headerGroup.id}>
                {headerGroup.headers.map((header) => (
                  <TableHead key={header.id} colSpan={header.colSpan}>
                    {header.isPlaceholder
                      ? null
                      : flexRender(
                          header.column.columnDef.header,
                          header.getContext(),
                        )}
                  </TableHead>
                ))}
              </TableRow>
            ))}
          </TableHeader>
          <TableBody>
            <DataTableBody
              table={table}
              columns={columns}
              isLoading={isLoading}
              fetchStatus={fetchStatus}
              emptyMessage={emptyMessage}
              offlineMessage={offlineMessage}
              pageSize={currentPagination.pageSize}
            />
          </TableBody>
        </Table>
      </div>

      {pageCount !== 0 && <DataTablePagination table={table} />}
    </div>
  );
}

```
