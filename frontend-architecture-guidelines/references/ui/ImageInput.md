---
title: ImageInput
description: Component reference
---

# ImageInput.tsx

**Why this pattern?**
> Encapsulates react-hook-form logic so feature components do not have to wire up Controllers manually.

## Implementation

```tsx
import { type ChangeEvent, useEffect, useId, useRef, useState } from "react";
import { ImagePlus, X } from "lucide-react";
import { cn } from "@/libs/utils";

export interface ImageInputProps {
  /** Field label rendered above the picker. */
  label?: string;
  /** Controlled preview URL. When provided the component is controlled. */
  previewUrl?: string | null;
  /** Called with the selected file (or null on clear). */
  onChange?: (file: File | null) => void;
  /** Called when the user removes the current image. */
  onClear?: () => void;
  error?: string;
  hint?: string;
  disabled?: boolean;
  accept?: string;
  className?: string;
  id?: string;
}

export function ImageInput({
  label,
  previewUrl,
  onChange,
  onClear,
  error,
  hint,
  disabled,
  accept = "image/*",
  className,
  id,
}: ImageInputProps) {
  const generatedId = useId();
  const inputId = id ?? generatedId;
  const inputRef = useRef<HTMLInputElement>(null);
  const objectUrlRef = useRef<string | null>(null);

  const isControlled = previewUrl !== undefined;
  const [internalPreview, setInternalPreview] = useState<string | null>(null);

  const preview = isControlled ? previewUrl : internalPreview;

  useEffect(() => {
    return () => {
      if (objectUrlRef.current) {
        URL.revokeObjectURL(objectUrlRef.current);
      }
    };
  }, []);

  const handleSelect = (event: ChangeEvent<HTMLInputElement>) => {
    const file = event.target.files?.[0] ?? null;
    if (!file) {
      return;
    }

    if (!isControlled) {
      if (objectUrlRef.current) {
        URL.revokeObjectURL(objectUrlRef.current);
      }
      const url = URL.createObjectURL(file);
      objectUrlRef.current = url;
      setInternalPreview(url);
    }

    if (inputRef.current) {
      inputRef.current.value = "";
    }

    onChange?.(file);
  };

  const handleClear = () => {
    if (objectUrlRef.current) {
      URL.revokeObjectURL(objectUrlRef.current);
      objectUrlRef.current = null;
    }
    setInternalPreview(null);
    if (inputRef.current) {
      inputRef.current.value = "";
    }
    onClear?.();
  };

  return (
    <div className={cn("space-y-1.5", className)}>
      {label && (
        <label
          htmlFor={inputId}
          className="block text-sm font-medium text-foreground"
        >
          {label}
        </label>
      )}

      <input
        ref={inputRef}
        id={inputId}
        type="file"
        accept={accept}
        disabled={disabled}
        className="hidden"
        onChange={handleSelect}
      />

      {preview ? (
        <div className="relative overflow-hidden rounded-[8px] border border-gray-200">
          <img
            src={preview}
            alt={label ?? "Selected image"}
            className="h-40 w-full object-cover"
          />
          {!disabled && (
            <button
              type="button"
              onClick={handleClear}
              aria-label="Remove image"
              className="absolute right-2 top-2 inline-flex h-7 w-7 cursor-pointer items-center justify-center rounded-full bg-foreground/70 text-white transition-colors hover:bg-foreground"
            >
              <X className="h-4 w-4" />
            </button>
          )}
        </div>
      ) : (
        <label
          htmlFor={inputId}
          className={cn(
            "flex h-32 cursor-pointer flex-col items-center justify-center gap-1.5 rounded-[8px] border border-dashed text-sm transition-all duration-200",
            error
              ? "border-red-400 text-red-500 hover:border-red-500 hover:bg-red-50"
              : "border-gray-300 text-gray-500 hover:border-primary hover:bg-muted",
            disabled && "cursor-not-allowed opacity-50",
          )}
        >
          <ImagePlus className="h-6 w-6" />
          <span>Choose image</span>
        </label>
      )}

      {error && (
        <p className="flex items-center gap-1 text-xs font-medium text-red-600">
          {error}
        </p>
      )}
      {hint && !error && <p className="text-xs text-gray-500">{hint}</p>}
    </div>
  );
}

```
