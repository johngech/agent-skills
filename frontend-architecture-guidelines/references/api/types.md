---
title: types
description: API implementation reference
---

```typescript
export interface BaseApiResponse {
  success: boolean;
  timestamp: string;
  path: string;
  message?: string;
  errors?: Record<string, string[]>;
}

export interface ApiResponse<T> extends BaseApiResponse {
  data: T;
}

export interface PaginatedApiResponse<T> extends BaseApiResponse {
  page: number;
  limit: number;
  totalItems: number;
  totalPages: number;
  /**
   * Absolute URL of the next/previous page, or `null` at either boundary.
   * Both keys are always present in the response but carry `null` rather than
   * being omitted, so they are typed as nullable.
   */
  next?: string | null;
  prev?: string | null;
  results: T[];
}

```
