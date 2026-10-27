# FormSwitchGroup.tsx

**Why this pattern?**
> Encapsulates react-hook-form logic so feature components do not have to wire up Controllers manually.

## Implementation

```tsx
import { get, useFormContext } from "react-hook-form";
import { Switch } from "@/components/ui/Switch";

export interface SwitchOption {
  id: number | string;
  label: string;
  description?: string;
}

export interface FormSwitchGroupProps {
  name: string;
  label?: string;
  options: SwitchOption[];
}

export function FormSwitchGroup({
  name,
  label,
  options,
}: Readonly<FormSwitchGroupProps>) {
  const {
    watch,
    setValue,
    formState: { errors },
  } = useFormContext();

  const value = watch(name) as (number | string)[] | undefined;
  const selectedSet = new Set(value ?? []);

  const handleToggle = (id: number | string, checked: boolean) => {
    const next = new Set(selectedSet);
    if (checked) next.add(id);
    else next.delete(id);
    setValue(name, Array.from(next), {
      shouldValidate: true,
      shouldDirty: true,
    });
  };

  const error = get(errors, name) as { message?: string } | undefined;

  return (
    <div className="space-y-3">
      {label && <label className="block text-sm font-medium">{label}</label>}
      <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
        {options.map((opt) => (
          <div
            key={opt.id}
            className="flex items-center justify-between py-3 border-b border-gray-100 last:border-0"
          >
            <div>
              <div className="font-medium">{opt.label}</div>
              {opt.description && (
                <div className="text-xs text-muted-foreground font-mono mt-1">
                  {opt.description}
                </div>
              )}
            </div>
            <Switch
              checked={selectedSet.has(opt.id)}
              onCheckedChange={(checked) => handleToggle(opt.id, checked)}
            />
          </div>
        ))}
      </div>
      {error && <p className="text-xs text-red-500">{error.message}</p>}
    </div>
  );
}

```
