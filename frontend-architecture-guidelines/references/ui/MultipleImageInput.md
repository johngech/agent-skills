# MultipleImageInput.tsx

**Why this pattern?**
> Encapsulates react-hook-form logic so feature components do not have to wire up Controllers manually.

## Implementation

```tsx
import { type ChangeEvent, useEffect, useId, useRef, useState } from "react";
import { ImagePlus, X } from "lucide-react";
import { cn } from "@/libs/utils";

export interface MultipleImageInputProps {
  /** Field label rendered above the picker. */
  label?: string;
  /** Controlled preview URLs. When provided the component is controlled. */
  previews?: string[];
  /** Called with newly selected files (appended to the collection). */
  onChange?: (files: File[]) => void;
  /** Called when the image at the given index is removed. */
  onRemove?: (index: number) => void;
  error?: string;
  hint?: string;
  disabled?: boolean;
  accept?: string;
  /** Maximum number of images allowed. */
  max?: number;
  className?: string;
  id?: string;
}

export function MultipleImageInput({
  label,
  previews,
  onChange,
  onRemove,
  error,
  hint,
  disabled,
  accept = "image/*",
  max,
  className,
  id,
}: Readonly<MultipleImageInputProps>) {
  const generatedId = useId();
  const inputId = id ?? generatedId;
  const inputRef = useRef<HTMLInputElement>(null);
  const objectUrlsRef = useRef<string[]>([]);

  const isControlled = previews !== undefined;
  const [internalPreviews, setInternalPreviews] = useState<string[]>([]);
  const [internalFiles, setInternalFiles] = useState<File[]>([]);

  const visiblePreviews = isControlled ? previews : internalPreviews;
  const atLimit = max !== undefined && visiblePreviews.length >= max;

  useEffect(() => {
    return () => {
      objectUrlsRef.current.forEach((url) => URL.revokeObjectURL(url));
      objectUrlsRef.current = [];
    };
  }, []);

  const handleSelect = (event: ChangeEvent<HTMLInputElement>) => {
    const selected = Array.from(event.target.files ?? []);
    if (selected.length === 0) {
      return;
    }

    if (!isControlled) {
      const urls = selected.map((file) => URL.createObjectURL(file));
      objectUrlsRef.current.push(...urls);
      const nextFiles = [...internalFiles, ...selected];
      setInternalFiles(nextFiles);
      setInternalPreviews((prev) => [...prev, ...urls]);
      onChange?.(nextFiles);
    } else {
      onChange?.(selected);
    }

    if (inputRef.current) {
      inputRef.current.value = "";
    }
  };

  const handleRemove = (index: number) => {
    if (!isControlled) {
      const url = objectUrlsRef.current[index];
      if (url) {
        URL.revokeObjectURL(url);
      }
      objectUrlsRef.current = objectUrlsRef.current.filter(
        (_, i) => i !== index,
      );
      setInternalFiles((prev) => prev.filter((_, i) => i !== index));
      setInternalPreviews((prev) => prev.filter((_, i) => i !== index));
    }

    onRemove?.(index);
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

      <div className="grid grid-cols-3 gap-3">
        {visiblePreviews.map((preview, index) => (
          <div
            key={`${preview}-${index}`}
            className="relative aspect-square overflow-hidden rounded-[8px] border border-gray-200"
          >
            <img
              src={preview}
              alt={`${label ?? "Image"} ${index + 1}`}
              className="h-full w-full object-cover"
            />
            {!disabled && (
              <button
                type="button"
                onClick={() => handleRemove(index)}
                aria-label={`Remove image ${index + 1}`}
                className="absolute right-1.5 top-1.5 inline-flex h-6 w-6 cursor-pointer items-center justify-center rounded-full bg-foreground/70 text-white transition-colors hover:bg-foreground"
              >
                <X className="h-3.5 w-3.5" />
              </button>
            )}
          </div>
        ))}

        {!disabled && !atLimit && (
          <label
            htmlFor={inputId}
            className={cn(
              "flex aspect-square cursor-pointer flex-col items-center justify-center gap-1 rounded-[8px] border border-dashed text-xs text-gray-500 transition-all duration-200",
              error
                ? "border-red-400 text-red-500 hover:border-red-500 hover:bg-red-50"
                : "border-gray-300 hover:border-primary hover:bg-muted",
            )}
          >
            <ImagePlus className="h-5 w-5" />
            <span>Add</span>
          </label>
        )}
      </div>

      <input
        ref={inputRef}
        id={inputId}
        type="file"
        accept={accept}
        multiple
        disabled={disabled}
        className="hidden"
        onChange={handleSelect}
      />

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
