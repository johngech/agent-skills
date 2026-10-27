# FormContainer.tsx

**Why this pattern?**
> Encapsulates react-hook-form logic so feature components do not have to wire up Controllers manually.

## Implementation

```tsx
import { cn } from "@/libs/utils";
import type { ReactNode } from "react";

export interface FormContainerProps {
  children: ReactNode;
  className?: string;
}

/**
 * A centered, max-width container with a thin light border and subtle
 * background. Wraps feature form components to keep forms visually
 * contained and consistent across all admin pages.
 */
export function FormContainer({
  children,
  className,
}: Readonly<FormContainerProps>) {
  return (
    <div
      className={cn(
        "w-full max-w-2xl rounded-[8px] border border-gray-100 bg-white p-8",
        className,
      )}
    >
      {children}
    </div>
  );
}

```
