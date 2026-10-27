---
title: token-manager
description: API implementation reference
---

```typescript
import axios from "axios";
import { ApiError } from "@/api/errors";
import { extractAccessToken } from "@/api/unwrap";
import { env } from "@/lib/config/env";

export interface TokenManagerOptions {
  refreshEndpoint?: string;
  /** Channel name for cross-tab rotation. Pass null to disable. */
  channelName?: string | null;
}

type TokenListener = (token: string | null) => void;

type AuthChannelMessage =
  | { type: "token"; accessToken: string }
  | { type: "logout" };

/** Storage key for the access token in localStorage. */
const STORAGE_KEY = "accessToken";

/**
 * Holds the access token in memory and persists it to localStorage so sessions
 * survive page reloads.
 *
 * `BroadcastChannel` and `storage` events keep sibling tabs in step so a login,
 * rotation, or logout in one tab is immediately reflected across all tabs.
 */
export class TokenManager {
  private readonly refreshEndpoint: string;
  private readonly channelName: string | null;
  private accessToken: string | null = null;
  private refreshPromise: Promise<string> | null = null;
  private channel: BroadcastChannel | null = null;
  private storageListener: ((event: StorageEvent) => void) | null = null;
  /** Guards against echoing a message we just received from another tab. */
  private applyingRemote = false;
  private readonly listeners = new Set<TokenListener>();

  constructor(options: TokenManagerOptions = {}) {
    this.refreshEndpoint = options.refreshEndpoint ?? "/auth/refresh";
    this.channelName =
      options.channelName === undefined ? "ethiomfi-auth" : options.channelName;

    this.readStorageToken();
    this.openChannel();
    this.listenToStorage();
  }

  // -------------------------------------------------------------------------
  // Token access
  // -------------------------------------------------------------------------

  getAccessToken(): string | null {
    return this.accessToken;
  }

  hasAccessToken(): boolean {
    return this.accessToken !== null;
  }

  /** Subscribe to token changes. Returns an unsubscribe function. */
  subscribe(listener: TokenListener): () => void {
    this.listeners.add(listener);
    return () => {
      this.listeners.delete(listener);
    };
  }

  setAccessToken(token: string): void {
    this.accessToken = token;
    this.writeStorageToken(token);
    this.notify(token);
    this.broadcast({ type: "token", accessToken: token });
  }

  clearAccessToken(): void {
    this.accessToken = null;
    this.writeStorageToken(null);
    this.notify(null);
    this.broadcast({ type: "logout" });
  }

  // -------------------------------------------------------------------------
  // Refresh
  // -------------------------------------------------------------------------

  /**
   * Refresh the access token. Race-condition-safe: concurrent callers share a
   * single in-flight refresh promise, and the promise resets after completion
   * (success OR failure) so subsequent 401s trigger a fresh refresh.
   */
  async refreshAccessToken(): Promise<string> {
    this.refreshPromise ??= this.requestRefresh().finally(() => {
      this.refreshPromise = null;
    });
    return this.refreshPromise;
  }

  /**
   * POST the refresh cookie using a bare axios instance, so the 401 interceptor
   * in `client.ts` cannot recurse into this call.
   */
  private async requestRefresh(): Promise<string> {
    let data: unknown;
    try {
      const response = await axios.post<unknown>(
        `${env.apiBaseUrl}${this.refreshEndpoint}`,
        {},
        {
          withCredentials: true,
          headers: { "Content-Type": "application/json" },
        },
      );
      data = response.data;
    } catch (error) {
      throw toRefreshError(error);
    }

    // The backend wraps the token at `data.token.accessToken`; anything else
    // means the contract changed, which must fail loudly rather than install
    // `undefined` as a bearer token.
    let accessToken: string;
    try {
      accessToken = extractAccessToken(data);
    } catch (error) {
      throw new ApiError("Refresh response did not include an access token", {
        cause: error,
      });
    }

    this.setAccessToken(accessToken);
    return accessToken;
  }

  // -------------------------------------------------------------------------
  // Teardown
  // -------------------------------------------------------------------------

  /** Clear every piece of token state and close the channel (used by logout). */
  clear(): void {
    this.accessToken = null;
    this.refreshPromise = null;
    this.writeStorageToken(null);
    this.applyRemote(() => {
      this.notify(null);
    });
    this.broadcast({ type: "logout" });
  }

  /** Release the BroadcastChannel and storage listeners. Safe to call more than once. */
  destroy(): void {
    this.channel?.close();
    this.channel = null;
    if (typeof window !== "undefined" && this.storageListener) {
      window.removeEventListener("storage", this.storageListener);
      this.storageListener = null;
    }
  }

  // -------------------------------------------------------------------------
  // Internals
  // -------------------------------------------------------------------------

  private readStorageToken(): void {
    if (typeof window === "undefined") return;
    try {
      this.accessToken = window.localStorage.getItem(STORAGE_KEY) || null;
    } catch {
      this.accessToken = null;
    }
  }

  private writeStorageToken(token: string | null): void {
    if (typeof window === "undefined") return;
    try {
      if (token) {
        window.localStorage.setItem(STORAGE_KEY, token);
      } else {
        window.localStorage.removeItem(STORAGE_KEY);
      }
    } catch {
      // Storage can be unavailable (private mode, full quota).
    }
  }

  private listenToStorage(): void {
    if (typeof window === "undefined") return;
    this.storageListener = (event: StorageEvent) => {
      if (event.key !== STORAGE_KEY) return;
      const newToken = event.newValue;
      if (newToken !== this.accessToken) {
        this.applyRemote(() => {
          this.accessToken = newToken;
          this.notify(newToken);
        });
      }
    };
    window.addEventListener("storage", this.storageListener);
  }

  private openChannel(): void {
    if (!this.channelName) return;
    if (typeof window === "undefined" || !("BroadcastChannel" in window)) {
      return;
    }

    try {
      this.channel = new BroadcastChannel(this.channelName);
      this.channel.onmessage = (event: MessageEvent<AuthChannelMessage>) => {
        this.receive(event.data);
      };
    } catch {
      this.channel = null;
    }
  }

  private broadcast(message: AuthChannelMessage): void {
    if (this.applyingRemote) return;
    try {
      this.channel?.postMessage(message);
    } catch {
      // A closed channel must never break the auth flow.
    }
  }

  private receive(message: AuthChannelMessage | undefined): void {
    if (!message || typeof message !== "object") return;

    if (message.type === "token" && typeof message.accessToken === "string") {
      this.applyRemote(() => {
        this.accessToken = message.accessToken;
        this.writeStorageToken(message.accessToken);
        this.notify(message.accessToken);
      });
      return;
    }

    if (message.type === "logout") {
      this.applyRemote(() => {
        this.accessToken = null;
        this.refreshPromise = null;
        this.writeStorageToken(null);
        this.notify(null);
      });
    }
  }

  private applyRemote(mutate: () => void): void {
    this.applyingRemote = true;
    try {
      mutate();
    } finally {
      this.applyingRemote = false;
    }
  }

  private notify(token: string | null): void {
    for (const listener of this.listeners) {
      try {
        listener(token);
      } catch (error) {
        console.error("Token listener failed", error);
      }
    }
  }
}

/** Normalize a refresh failure into an ApiError. */
function toRefreshError(error: unknown): ApiError {
  if (error instanceof ApiError) return error;
  return new ApiError(
    error instanceof Error && error.message
      ? error.message
      : "Could not refresh the session",
    { cause: error },
  );
}

/** Singleton used across the app. */
export const tokenManager = new TokenManager();

```
