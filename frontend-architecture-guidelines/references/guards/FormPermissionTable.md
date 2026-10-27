# FormPermissionTable (`FormPermissionTable.tsx`)

## Purpose

A form field wrapper bridging React Hook Form state with `PaginatedPermissionTable`, allowing permissions to be bound and validated directly inside a form schema.

---

## Implementation

```tsx
import { useFormContext, Controller } from "react-hook-form";
import { PaginatedPermissionTable } from "./PaginatedPermissionTable";

interface FormPermissionTableProps {
  name: string;
}

export function FormPermissionTable({
  name,
}: Readonly<FormPermissionTableProps>) {
  const { control } = useFormContext();

  return (
    <Controller
      name={name}
      control={control}
      render={({ field }) => {
        const value: number[] = field.value || [];

        const handleToggle = (id: number, checked: boolean) => {
          const set = new Set(value);
          if (checked) set.add(id);
          else set.delete(id);
          field.onChange(Array.from(set));
        };

        return (
          <PaginatedPermissionTable
            selectedIds={value}
            onToggle={handleToggle}
          />
        );
      }}
    />
  );
}
```
