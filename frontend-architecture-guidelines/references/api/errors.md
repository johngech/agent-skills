---
title: errors
description: API implementation reference
---

```typescript
import { isAxiosError } from "axios";

/**
 * Normalized client-side API error.
 *
 * Every rejection surfaced by `apiClient` is an `ApiError`, so consumers can
 * rely on `instanceof` and on a stable shape. The originating axios error is
 * preserved on `cause` for debugging (status, config, response) without leaking
 * the axios type into feature code.
 */
export class ApiError extends Error {
  /** HTTP status when known. */
  readonly status?: number;
  /** Per-field validation messages, flattened to `string[]` per field. */
  readonly fieldErrors: Record<string, string[]>;

  constructor(
    message: string,
    options: {
      status?: number;
      fieldErrors?: Record<string, string[]>;
      cause?: unknown;
    } = {},
  ) {
    super(message, { cause: options.cause });
    this.name = "ApiError";
    this.status = options.status;
    this.fieldErrors = options.fieldErrors ?? {};
  }
}

/** Type guard for `ApiError`. Narrows `unknown` without a cast. */
export function isApiError(error: unknown): error is ApiError {
  return error instanceof ApiError;
}

/**
 * Narrow a runtime value to a plain object (record) — the common shape of
 * NestJS error payloads. Arrays are excluded because a response body we read
 * here is never a top-level array.
 */
function isRecord(value: unknown): value is Record<string, unknown> {
  return typeof value === "object" && value !== null && !Array.isArray(value);
}

/** Read a non-blank string from an unknown value, or null. */
function readString(value: unknown): string | null {
  if (typeof value !== "string") return null;
  return value.trim() === "" ? null : value;
}

/** Pull a human-readable message out of a response body, or null. */
function readMessage(data: unknown): string | null {
  if (!isRecord(data)) return null;

  const message = readString(data.message);
  if (message) return message;

  const detail = readString(data.detail);
  if (detail) return detail;

  if (isRecord(data.error)) {
    return readString(data.error.message);
  }
  const error = readString(data.error);
  if (error) return error;

  return null;
}

/** Flatten the body-level `errors` record into "field: a, b; other: c". */
function readErrors(data: unknown): string | null {
  if (!isRecord(data) || !isRecord(data.errors)) return null;

  const parts: string[] = [];
  for (const [field, messages] of Object.entries(data.errors)) {
    if (!Array.isArray(messages)) continue;

    const text = messages.filter(
      (item): item is string => typeof item === "string",
    );
    if (text.length > 0) {
      parts.push(`${field}: ${text.join(", ")}`);
    }
  }

  return parts.length > 0 ? parts.join("; ") : null;
}

/** Extract per-field validation errors from a response body. */
function extractFieldErrors(data: unknown): Record<string, string[]> {
  if (!isRecord(data) || !isRecord(data.errors)) return {};

  const result: Record<string, string[]> = {};
  for (const [field, messages] of Object.entries(data.errors)) {
    if (Array.isArray(messages)) {
      result[field] = messages.filter(
        (item): item is string => typeof item === "string",
      );
    }
  }
  return result;
}

/**
 * Convert any thrown value into an `ApiError`.
 *
 * The original error is always attached as `cause`, so nothing is lost while
 * callers get a single predictable type to branch on.
 */
export function toApiError(
  error: unknown,
  fallback = "Request failed",
): ApiError {
  if (isApiError(error)) return error;

  if (isAxiosError(error)) {
    const data = error.response?.data;
    const message = readMessage(data) ?? readErrors(data) ?? error.message;
    return new ApiError(message || fallback, {
      status: error.response?.status,
      fieldErrors: extractFieldErrors(data),
      cause: error,
    });
  }

  const message = error instanceof Error ? error.message : "";
  return new ApiError(message || fallback, { cause: error });
}

/** Readable message from any thrown value. */
export function extractErrorMessage(
  error: unknown,
  fallback = "Request failed",
): string {
  return toApiError(error, fallback).message;
}

```
