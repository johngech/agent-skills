---
title: data-table-search.tsx
description: DataTable component reference
---

```typescript
import { cn } from "cn";
import { Search } from "lucide-react";
import { useEffect, useState } from "react";
import { Input } from "#/components/ui/input";
import { useDebounce } from "#/hooks/use-debounce";

export interface DataTableSearchProps {
  placeholder?: string;
  onSearch: (value: string) => void;
  debounceMs?: number;
  className?: string;
  value?: string;
}

export function DataTableSearch({
  placeholder = "Search...",
  onSearch,
  debounceMs = 300,
  className,
  value: controlledValue,
}: Readonly<DataTableSearchProps>) {
  const [searchValue, setSearchValue] = useState(controlledValue ?? "");
  const debouncedValue = useDebounce(searchValue, debounceMs);

  useEffect(() => {
    onSearch(debouncedValue);
  }, [debouncedValue, onSearch]);

  return (
    <div className={cn("relative", className)}>
      <Search className="absolute left-3 top-1/2 h-4 w-4 -translate-y-1/2 text-muted-foreground" />
      <Input
        placeholder={placeholder}
        value={searchValue}
        onChange={(e) => setSearchValue(e.target.value)}
        className="pl-9"
      />
    </div>
  );
}

```
