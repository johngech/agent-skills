# FormTextarea.tsx

**Why this pattern?**
> Encapsulates react-hook-form logic so feature components do not have to wire up Controllers manually.

## Implementation

```tsx
import { get, useFormContext } from "react-hook-form";
import { Textarea, type TextareaProps } from "@/components/ui/Textarea";

export interface FormTextareaProps extends Omit<
  TextareaProps,
  "name" | "error" | "onChange" | "onBlur" | "ref"
> {
  /** Field name, matching a key in the form's Zod schema. */
  name: string;
}

export function FormTextarea({ name, ...props }: Readonly<FormTextareaProps>) {
  const {
    register,
    formState: { errors },
  } = useFormContext();

  const error = get(errors, name) as { message?: string } | undefined;

  return <Textarea {...register(name)} error={error?.message} {...props} />;
}

```
