# FormVideoPicker.tsx

**Why this pattern?**
> Encapsulates react-hook-form logic so feature components do not have to wire up Controllers manually.

## Implementation

```tsx
import { useEffect, useState } from "react";
import { get, useFormContext } from "react-hook-form";
import { VideoInput, type VideoInputProps } from "@/components/ui";

/** A single video field value: a newly picked File or an existing video URL. */
export type FormVideoValue = File | string | null;

export interface FormVideoPickerProps extends Omit<
  VideoInputProps,
  "previewUrl" | "onChange" | "onClear" | "error"
> {
  /** Field name, matching a key in the form's Zod schema. */
  name: string;
}

export function FormVideoPicker({
  name,
  ...props
}: Readonly<FormVideoPickerProps>) {
  const {
    watch,
    setValue,
    formState: { errors },
  } = useFormContext();

  const value = watch(name) as FormVideoValue | undefined;
  const [previewUrl, setPreviewUrl] = useState<string | null>(
    typeof value === "string" ? value : null,
  );

  useEffect(() => {
    if (value instanceof File) {
      const url = URL.createObjectURL(value);
      setPreviewUrl(url);
      return () => URL.revokeObjectURL(url);
    }
    setPreviewUrl(typeof value === "string" ? value : null);
  }, [value]);

  const error = get(errors, name) as { message?: string } | undefined;

  const handleChange = (file: File | null) => {
    setValue(name, file, { shouldValidate: true, shouldDirty: true });
  };

  const handleClear = () => {
    setValue(name, null, { shouldValidate: true, shouldDirty: true });
  };

  return (
    <VideoInput
      previewUrl={previewUrl}
      onChange={handleChange}
      onClear={handleClear}
      error={error?.message}
      {...props}
    />
  );
}

```
