# FormMultipleImagePicker.tsx

**Why this pattern?**
> Encapsulates react-hook-form logic so feature components do not have to wire up Controllers manually.

## Implementation

```tsx
import { useEffect, useState } from "react";
import { get, useFormContext } from "react-hook-form";
import {
  MultipleImageInput,
  type MultipleImageInputProps,
} from "@/components/ui";

/** Multiple image field value: newly picked Files and/or existing image URLs. */
export type FormMultipleImageValue = (File | string)[];

export interface FormMultipleImagePickerProps extends Omit<
  MultipleImageInputProps,
  "previews" | "onChange" | "onRemove" | "error"
> {
  /** Field name, matching a key in the form's Zod schema. */
  name: string;
}

export function FormMultipleImagePicker({
  name,
  ...props
}: Readonly<FormMultipleImagePickerProps>) {
  const {
    watch,
    setValue,
    formState: { errors },
  } = useFormContext();

  const value = watch(name) as FormMultipleImageValue | undefined;
  const items = value ?? [];

  const [previews, setPreviews] = useState<string[]>(() =>
    items.filter((item): item is string => typeof item === "string"),
  );

  useEffect(() => {
    const urls: string[] = [];

    const nextPreviews = items.map((item) => {
      if (typeof item === "string") {
        return item;
      }
      const url = URL.createObjectURL(item);
      urls.push(url);
      return url;
    });

    setPreviews(nextPreviews);

    return () => {
      urls.forEach((url) => URL.revokeObjectURL(url));
    };
  }, [value]);

  const error = get(errors, name) as { message?: string } | undefined;

  const handleAdd = (files: File[]) => {
    setValue(name, [...items, ...files], {
      shouldValidate: true,
      shouldDirty: true,
    });
  };

  const handleRemove = (index: number) => {
    setValue(
      name,
      items.filter((_, i) => i !== index),
      { shouldValidate: true, shouldDirty: true },
    );
  };

  return (
    <MultipleImageInput
      previews={previews}
      onChange={handleAdd}
      onRemove={handleRemove}
      error={error?.message}
      {...props}
    />
  );
}

```
