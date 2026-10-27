# SelectField (`SelectField.tsx`)

Located at: `src/components/forms/SelectField.tsx`

## Purpose

A form field wrapper connecting Radix UI / shadcn `<Select>` with React Hook Form's `Controller`. Automatically manages label rendering, option selection, and validation error messages via `<FormMessage>`.

---

## Implementation

```tsx
import { Controller, useFormContext } from "react-hook-form";
import {
  FormControl,
  FormItem,
  FormLabel,
  FormMessage,
} from "@/components/ui/form";
import {
  Select,
  SelectContent,
  SelectItem,
  SelectTrigger,
  SelectValue,
} from "@/components/ui/select";

export interface SelectFieldProps {
  name: string;
  label: string;
  options: { value: string; label: string }[];
  placeholder?: string;
}

export function SelectField({
  name,
  label,
  options,
  placeholder,
}: Readonly<SelectFieldProps>) {
  const { control } = useFormContext();

  return (
    <Controller
      control={control}
      name={name}
      render={({ field }) => (
        <FormItem>
          <FormLabel>{label}</FormLabel>
          <Select value={field.value} onValueChange={field.onChange}>
            <FormControl>
              <SelectTrigger className="w-full">
                <SelectValue placeholder={placeholder} />
              </SelectTrigger>
            </FormControl>
            <SelectContent>
              {options.map((option) => (
                <SelectItem key={option.value} value={option.value}>
                  {option.label}
                </SelectItem>
              ))}
            </SelectContent>
          </Select>
          <FormMessage />
        </FormItem>
      )}
    />
  );
}
```

---

## Usage Example

```tsx
<SelectField
  name="clientType"
  label="Client Type"
  options={[
    { label: "Individual", value: "INDIVIDUAL" },
    { label: "Joint", value: "JOINT" },
    { label: "Corporate", value: "CORPORATE" },
  ]}
/>
```
