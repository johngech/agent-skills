# FormSwitch.tsx

**Why this pattern?**
> Encapsulates react-hook-form logic so feature components do not have to wire up Controllers manually.

## Implementation

```tsx
import { get, useFormContext, Controller } from "react-hook-form";
import { Switch } from "@/components/ui/Switch";

export interface FormSwitchProps {
  name: string;
  label: string;
  description?: string;
}

export function FormSwitch({
  name,
  label,
  description,
}: Readonly<FormSwitchProps>) {
  const {
    control,
    formState: { errors },
  } = useFormContext();

  const error = get(errors, name) as { message?: string } | undefined;

  return (
    <div className="flex items-center justify-between py-3 border-b border-gray-100 last:border-0">
      <div>
        <div className="font-medium">{label}</div>
        {description && (
          <div className="text-sm text-muted-foreground mt-1">
            {description}
          </div>
        )}
        {error && <p className="text-xs text-red-500 mt-1">{error.message}</p>}
      </div>
      <Controller
        control={control}
        name={name}
        render={({ field: { value, onChange } }) => (
          <Switch checked={!!value} onCheckedChange={onChange} />
        )}
      />
    </div>
  );
}

```
