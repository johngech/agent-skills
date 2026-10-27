---
name: project-error-boundary-architecture
description: How error boundaries are built in this project. Use when creating or modifying any error boundary, crash fallback UI, or route protection — always reuse the existing ErrorBoundary, ErrorFallback, and RouteErrorFallback components. The boundary layer is built on React class components + react-router (errorElement, useRouteError); styling follows the project's design system. Never write raw class-based error boundaries in feature code.
version: 1.0.0
author: Yohannes Getachew
type: skill
category: development
tags:
  - error-boundary
  - error-handling
  - react
  - react-router
  - resilience
  - structure
---

# Project Error Boundary Architecture

**Purpose**: Teach the agent how error boundaries are built in this project so every new page, section, or route gets the same crash protection instead of inventing a new boundary per feature.

This project already contains a complete error boundary layer built on top of:

- React class components (`getDerivedStateFromError` + `componentDidCatch`)
- react-router (`errorElement`, `useRouteError`)

This skill focuses on **boundary structure and placement**, not styling. Styling follows whatever design system the project uses. Adapt the examples to the project's own components and design tokens.

The agent **MUST ALWAYS** use these abstractions. The agent must **never** generate its own error boundary implementation when an existing project component already solves the problem.

---

### What I Do

- Enforce reuse of the existing boundary components (`ErrorBoundary`, `ErrorFallback`, `RouteErrorFallback`)
- Prevent raw class-based error boundaries in feature code
- Standardize crash fallback UI and recovery (reset / retry) semantics
- Provide canonical patterns for app-root, layout, section, and individual-route protection

### When to Use

Use this skill whenever you are:

- Adding an error boundary to a page, section, or route
- Building or editing crash fallback UI
- Protecting routes with `errorElement` (data-router syntax) or a component wrapper
- Handling render errors, loader/action errors, or async/query errors near a boundary
- Wiring recovery behavior (retry buttons, `resetKeys`, `onReset`, `onError`)

---

## Core Rules

### Rule #1 — Always reuse existing components

If the project already provides `ErrorBoundary`, `ErrorFallback`, or `RouteErrorFallback`, they **must** be used. Never replace them.

### Rule #2 — Never write raw class-based boundaries in feature code

Never generate a class component like `class MyPageBoundary extends Component` in feature code. 
Instead, always generate a functional wrapper:

```tsx
<ErrorBoundary resetKeys={[id]} onReset={handleReset}>
  <PageContent />
</ErrorBoundary>
```

### Rule #3 — Know what a boundary catches

An error boundary catches errors thrown while **rendering**, in **lifecycle methods**, and in **constructors**.
It does **NOT** catch:

- Event handlers (`onClick`, `onSubmit`, ...)
- Async code: promises, `setTimeout`, `await` inside event handlers
- Errors thrown inside the boundary itself
- Server-side rendering
*(Handle async errors locally with try/catch)*

### Rule #4 — Never use boundaries for expected errors

Boundaries are for **unexpected crashes**, not for validation errors, 404s, or business logic failures. Handle expected errors locally.

---

## Architecture Patterns

### 1. Application Boundary

Protects the entire app from fatal crashes outside the router.
**Implementation:** *See `references/AppErrorBoundary.md`.*

### 2. Route / Page Boundary

Protects an individual page. Prevents a single page failure from crashing the application and allows navigation to continue.
**Implementation:** *See `references/RouteErrorFallback.md` (for `errorElement`) or wrap the page in an `<ErrorBoundary>`.*

### 3. Widget Boundary

Protects reusable dashboard widgets. Keeps the surrounding layout intact while isolating the failed region.
**Implementation:** *See `references/MinimalFallback.md`.*

---

## Core Components & References

Instead of generating boundaries from scratch, always use the existing implementations. For internal code details and usage examples, refer to these references:

- **[AppErrorBoundary](../error-boundary/AppErrorBoundary.md)**: Global app boundary.
- **[ErrorBoundary](../error-boundary/ErrorBoundary.md)**: Core class-based boundary (`getDerivedStateFromError` + `componentDidCatch`).
- **[ErrorFallback](../error-boundary/ErrorFallback.md)**: Default full-page fallback UI.
- **[MinimalFallback](../error-boundary/MinimalFallback.md)**: Compact fallback UI for widgets.
- **[RouteErrorFallback](../error-boundary/RouteErrorFallback.md)**: Fallback wired via react-router's `errorElement`.
- **[ErrorReporter](../error-boundary/ErrorReporter.md)**: Centralized error logging utility.

---

## Guidelines & Best Practices

- **TanStack Query Integration**: Use `<QueryErrorResetBoundary>` to reset failed queries when the boundary resets.
- **Navigation Recovery**: Automatically clear errors after route changes using `resetKeys={[location.pathname]}`.
- **Logging Flow**: Component -> Error Boundary -> `reportError()` -> External Service. Never report errors directly from UI components.
- **Fallback Independence**: Keep fallback components purely presentational (no API calls, no business logic).
