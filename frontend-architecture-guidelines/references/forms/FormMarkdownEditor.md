# FormMarkdownEditor.tsx

**Why this pattern?**
> Encapsulates react-hook-form logic so feature components do not have to wire up Controllers manually.

## Implementation

```tsx
import { get, useFormContext, Controller } from "react-hook-form";
import { MarkdownEditor } from "@/components/ui/MarkdownEditor";

export interface FormMarkdownEditorProps {
  name: string;
  label?: string;
  placeholder?: string;
}

export function FormMarkdownEditor({
  name,
  label,
  placeholder,
}: Readonly<FormMarkdownEditorProps>) {
  const {
    control,
    formState: { errors },
  } = useFormContext();

  const error = get(errors, name) as { message?: string } | undefined;

  return (
    <Controller
      name={name}
      control={control}
      render={({ field }) => (
        <MarkdownEditor
          label={label}
          placeholder={placeholder}
          value={field.value || ""}
          onChange={field.onChange}
          error={error?.message}
        />
      )}
    />
  );
}

```
