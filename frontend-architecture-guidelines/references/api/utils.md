# API Table Utilities (`utils.ts`)

Located at: `src/api/utils.ts`

## Purpose

Provides resilient collection unpacking and pagination metadata extraction for TanStack Table components and query options.

List endpoints in the enterprise backend return paginated responses:
```json
{
  "total": 120,
  "totalPages": 12,
  "currentPage": 1,
  "limit": 10,
  "results": [ ... ]
}
```

Certain non-paginated endpoints return flat arrays wrapped in `{ data: [ ... ] }`. The utility functions normalize both formats.

---

## Implementation

```typescript
import type { ApiResponse, PaginatedApiResponse } from "./types";

/** Any response shape that may carry a list of records. */
type ListResponse<T> = ApiResponse<T[]> | PaginatedApiResponse<T>;

/**
 * Extract the records from a list response.
 *
 * List endpoints return a paginated envelope with a `results` array. Detail
 * endpoints return a `data` field instead, so a non-paginated body is passed
 * through defensively rather than being coerced.
 */
export function getListData<T>(
  response: ListResponse<T> | undefined | null,
): T[] {
  if (!response) return [];
  if ("results" in response) return response.results;
  return Array.isArray(response.data) ? response.data : [];
}

/**
 * Number of pages in a paginated response.
 *
 * Defaults to `1` so a table that receives a non-paginated body still renders
 * its rows, instead of computing a page count of zero and hiding the data.
 */
export function getPageCount(
  response: ListResponse<unknown> | undefined | null,
  defaultCount = 1,
): number {
  if (!response) return defaultCount;
  if ("totalPages" in response) return response.totalPages;
  return defaultCount;
}
```

---

## Usage with DataTable

```tsx
function MemberListPage() {
  const { data } = useSuspenseQuery(memberListOptions(search));

  return (
    <DataTable
      columns={columns}
      data={getListData(data)}
      pageCount={getPageCount(data, 1)}
    />
  );
}
```
