# SubmitButton.tsx

**Why this pattern?**
> Automatically tracks form submission state to show loading spinners and disable double-clicks.

## Implementation

```tsx
import { type ReactNode } from "react";
import { useFormContext } from "react-hook-form";
import { Button, type ButtonProps } from "@/components/ui";

export interface SubmitButtonProps extends Omit<
  ButtonProps,
  "type" | "isLoading" | "disabled" | "title"
> {
  /** Button label. Children override title when provided. */
  title?: ReactNode;
}

export function SubmitButton({
  title,
  children,
  ...props
}: Readonly<SubmitButtonProps>) {
  const {
    formState: { isSubmitting, isSubmitted, isValid },
  } = useFormContext();

  return (
    <Button
      type="submit"
      isLoading={isSubmitting}
      disabled={!isValid && isSubmitted}
      {...props}
    >
      {children ?? title}
    </Button>
  );
}

```
