---
title: data-table-filter.tsx
description: DataTable component reference
---

```typescript
import { cn } from "cn";
import {
  Select,
  SelectContent,
  SelectItem,
  SelectTrigger,
  SelectValue,
} from "#/components/ui/select";

export interface DataTableFilterOption {
  value: string;
  label: string;
}

export interface DataTableFilterProps {
  label: string;
  options: DataTableFilterOption[];
  value?: string;
  onChange: (value: string) => void;
  className?: string;
  placeholder?: string;
}

export function DataTableFilter({
  label,
  options,
  value = "",
  onChange,
  className,
  placeholder = `All ${label}s`,
}: Readonly<DataTableFilterProps>) {
  const handleChange = (selected: string) => {
    onChange(selected === "all" ? "" : selected);
  };

  // Convert empty value to "all" so Select UI handles it smoothly
  const displayValue = value === "" ? "all" : value;

  return (
    <div className={cn("min-w-37.5", className)}>
      <Select value={displayValue} onValueChange={handleChange}>
        <SelectTrigger>
          <SelectValue placeholder={placeholder} />
        </SelectTrigger>
        <SelectContent>
          <SelectItem value="all">{placeholder}</SelectItem>
          {options.map((option) => (
            <SelectItem key={option.value} value={option.value}>
              {option.label}
            </SelectItem>
          ))}
        </SelectContent>
      </Select>
    </div>
  );
}

```
