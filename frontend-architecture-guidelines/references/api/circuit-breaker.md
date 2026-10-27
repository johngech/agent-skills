---
title: circuit-breaker
description: API implementation reference
---

```typescript
import { ApiError, isApiError } from "@/api/errors";

// ---------------------------------------------------------------------------
// Types
// ---------------------------------------------------------------------------

type CircuitState = "closed" | "open" | "half-open";

export interface CircuitBreakerConfig {
  /** Number of consecutive failures in "closed" state before tripping to "open". */
  failureThreshold: number;
  /** Number of consecutive successes in "half-open" state before resetting to "closed". */
  successThreshold: number;
  /** Milliseconds to wait in "open" state before allowing a test request (→ "half-open"). */
  cooldownMs: number;
}

// ---------------------------------------------------------------------------
// Defaults
// ---------------------------------------------------------------------------

const DEFAULT_CONFIG: CircuitBreakerConfig = {
  failureThreshold: 5,
  successThreshold: 2,
  cooldownMs: 30_000,
};

/**
 * Whether a thrown error counts as an upstream outage.
 *
 * The breaker exists to ride out backend trouble, so only server-side and
 * transport-level failures may feed it:
 * - `5xx` responses — the backend is failing to serve the request;
 * - `429` — the backend is refusing work;
 * - no status at all — network failure or timeout, reported by `toApiError`
 *   as an `ApiError` without `status`.
 *
 * Every other rejection is a caller problem: a `400` from an invalid form or
 * a `404` for a missing record both prove the service answered promptly and
 * correctly. Counting those would let five typos take a feature offline for
 * thirty seconds and would poison reads (list/get) through a failed write.
 */
function isOutage(error: unknown): boolean {
  if (!isApiError(error)) return true;
  if (error.status === undefined) return true;
  return error.status >= 500 || error.status === 429;
}

// ---------------------------------------------------------------------------
// CircuitBreaker
// ---------------------------------------------------------------------------

/**
 * Classic circuit-breaker state machine that wraps async operations.
 *
 * States
 * ------
 * - **closed** – requests pass through; consecutive *outages* are tracked
 *   (4xx validation errors do not count, see `isOutage`).
 * - **open** – requests are rejected immediately with a 503 `ApiError` until
 *   the cooldown elapses.
 * - **half-open** – a single test request is allowed through. Successes count
 *   toward `successThreshold`; any failure snaps back to "open".
 *
 * Usage:
 * ```ts
 * const breaker = new CircuitBreaker();
 * const result = await breaker.execute(() => axios.get("/data"));
 * ```
 */
export class CircuitBreaker {
  private readonly config: CircuitBreakerConfig;

  private state: CircuitState = "closed";
  private consecutiveFailures = 0;
  private consecutiveSuccesses = 0;
  private lastFailureAt = 0;
  /** In-flight half-open probe, so concurrent callers share one request. */
  private halfOpenProbe: Promise<unknown> | null = null;

  constructor(config?: Partial<CircuitBreakerConfig>) {
    this.config = { ...DEFAULT_CONFIG, ...config };
  }

  // -----------------------------------------------------------------------
  // Public API
  // -----------------------------------------------------------------------

  /**
   * Run `fn` through the circuit breaker.
   *
   * - In **closed** state the call is forwarded and the success/failure
   *   counters are updated accordingly.
   * - In **open** state the call is rejected without invoking `fn` unless
   *   the cooldown has elapsed, in which case the breaker transitions to
   *   **half-open** and lets the request through.
   * - In **half-open** state exactly one test request is forwarded; the
   *   outcome determines the next state.
   */
  async execute<T>(fn: () => Promise<T>): Promise<T> {
    this.transitionIfReady();

    switch (this.state) {
      case "open":
        throw new CircuitBreakerOpenError();

      case "half-open":
        // Exactly one probe may be in flight. Concurrent callers await the
        // same promise rather than each being forwarded through the breaker,
        // which would defeat the point of a half-open state.
        this.halfOpenProbe ??= this.handleHalfOpen(fn).finally(() => {
          this.halfOpenProbe = null;
        });
        return this.halfOpenProbe as Promise<T>;

      case "closed":
        return this.handleClosed(fn);
    }
  }

  /** Read-only view of the current state (useful for testing / logging). */
  get currentState(): CircuitState {
    return this.state;
  }

  /** Reset the breaker to its initial "closed" state. */
  reset(): void {
    this.state = "closed";
    this.consecutiveFailures = 0;
    this.consecutiveSuccesses = 0;
    this.lastFailureAt = 0;
    this.halfOpenProbe = null;
  }

  // -----------------------------------------------------------------------
  // Internal helpers
  // -----------------------------------------------------------------------

  /** Transition from "open" → "half-open" once the cooldown has elapsed. */
  private transitionIfReady(): void {
    if (this.state !== "open") return;

    const elapsed = Date.now() - this.lastFailureAt;
    if (elapsed >= this.config.cooldownMs) {
      this.state = "half-open";
      this.consecutiveSuccesses = 0;
    }
  }

  /** Forward the request in "closed" state and update counters. */
  private async handleClosed<T>(fn: () => Promise<T>): Promise<T> {
    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (error: unknown) {
      // A caller-side error still proves the backend is healthy, so it resets
      // the streak rather than extending it.
      if (isOutage(error)) this.onFailure();
      else this.onSuccess();
      throw error;
    }
  }

  /**
   * Forward the single test request in "half-open" state.
   *
   * `successThreshold` consecutive successes → "closed".
   * An outage → "open" (timer restarts). A caller-side error is *also*
   * evidence that the backend recovered — half-open is probing for a
   * responsive service, not for a 2xx — so it advances the success count
   * instead of reopening the circuit.
   */
  private async handleHalfOpen<T>(fn: () => Promise<T>): Promise<T> {
    try {
      const result = await fn();
      this.recordHalfOpenSuccess();
      return result;
    } catch (error: unknown) {
      if (isOutage(error)) {
        this.tripOpen();
        throw error;
      }
      this.recordHalfOpenSuccess();
      throw error;
    }
  }

  /** Count one non-outage outcome while half-open; close at the threshold. */
  private recordHalfOpenSuccess(): void {
    this.consecutiveSuccesses += 1;

    if (this.consecutiveSuccesses >= this.config.successThreshold) {
      this.state = "closed";
      this.consecutiveFailures = 0;
      this.consecutiveSuccesses = 0;
    }
  }

  /** Record a success while in "closed" state. */
  private onSuccess(): void {
    this.consecutiveFailures = 0;
  }

  /** Record a failure while in "closed" state; trip to "open" if threshold is reached. */
  private onFailure(): void {
    this.consecutiveFailures += 1;

    if (this.consecutiveFailures >= this.config.failureThreshold) {
      this.tripOpen();
    }
  }

  /** Transition to "open" and record the timestamp for cooldown tracking. */
  private tripOpen(): void {
    this.state = "open";
    this.lastFailureAt = Date.now();
    this.consecutiveFailures = 0;
    this.consecutiveSuccesses = 0;
  }
}

// ---------------------------------------------------------------------------
// CircuitBreakerOpenError
// ---------------------------------------------------------------------------

/**
 * Thrown when a request is rejected because the circuit breaker is open.
 * Extends `ApiError` so upstream error-handling (extractErrorMessage, etc.)
 * treats it consistently with other API failures.
 */
export class CircuitBreakerOpenError extends ApiError {
  constructor() {
    super("Service temporarily unavailable. Please try again.", {
      status: 503,
    });
    this.name = "CircuitBreakerOpenError";
  }
}

```
