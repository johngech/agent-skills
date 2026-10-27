# FormImagePicker.tsx

**Why this pattern?**
> Connects file inputs and image previews seamlessly into the react-hook-form state.

## Implementation

```tsx
import { useEffect, useState } from "react";
import { get, useFormContext } from "react-hook-form";
import { ImageInput, type ImageInputProps } from "@/components/ui";

/** A single image field value: a newly picked File or an existing image URL. */
export type FormImageValue = File | string | null;

export interface FormImagePickerProps extends Omit<
  ImageInputProps,
  "previewUrl" | "onChange" | "onClear" | "error"
> {
  /** Field name, matching a key in the form's Zod schema. */
  name: string;
}

export function FormImagePicker({
  name,
  ...props
}: Readonly<FormImagePickerProps>) {
  const {
    watch,
    setValue,
    formState: { errors },
  } = useFormContext();

  const value = watch(name) as FormImageValue | undefined;
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
    <ImageInput
      previewUrl={previewUrl}
      onChange={handleChange}
      onClear={handleClear}
      error={error?.message}
      {...props}
    />
  );
}

```
