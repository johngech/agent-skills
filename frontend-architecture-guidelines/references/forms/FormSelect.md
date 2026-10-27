# FormSelect.tsx

**Why this pattern?**
> Encapsulates react-hook-form logic so feature components do not have to wire up Controllers manually.

## Implementation

```tsx
import { get, useFormContext } from "react-hook-form";
import { Select, type SelectProps } from "@/components/ui/Select";

export interface FormSelectProps extends Omit<
  SelectProps,
  "name" | "error" | "value" | "onChange"
> {
  /** Field name, matching a key in the form's Zod schema. */
  name: string;
}

export function FormSelect({ name, ...props }: Readonly<FormSelectProps>) {
  const {
    watch,
    setValue,
    formState: { errors },
  } = useFormContext();

  const error = get(errors, name) as { message?: string } | undefined;
  const value = watch(name);

  return (
    <Select
      value={value}
      onChange={(val) => setValue(name, val, { shouldValidate: true })}
      error={error?.message}
      {...props}
    />
  );
}

```
