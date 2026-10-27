# FormField.tsx

**Why this pattern?**
> Bridges standard UI inputs to the react-hook-form context automatically handling registration and error states.

## Implementation

```tsx
import { get, useFormContext } from "react-hook-form";
import { Input, type InputProps } from "@/components/ui";

export interface FormFieldProps extends Omit<
  InputProps,
  "name" | "errorText" | "onChange" | "onBlur" | "ref"
> {
  /** Field name, matching a key in the form's Zod schema. */
  name: string;
}

export function FormField({ name, type, ...props }: Readonly<FormFieldProps>) {
  const {
    register,
    formState: { errors },
  } = useFormContext();

  const error = get(errors, name) as { message?: string } | undefined;

  return (
    <Input
      {...register(name, type === "number" ? { valueAsNumber: true } : {})}
      type={type}
      errorText={error?.message}
      {...props}
    />
  );
}

```
