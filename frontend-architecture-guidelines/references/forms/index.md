# index.ts

**Why this pattern?**
> Encapsulates react-hook-form logic so feature components do not have to wire up Controllers manually.

## Implementation

```typescript
export { Form } from "./Form";
export type { FormProps } from "./Form";

export { FormField } from "./FormField";
export type { FormFieldProps } from "./FormField";

export { FormImagePicker } from "./FormImagePicker";
export type { FormImagePickerProps, FormImageValue } from "./FormImagePicker";

export { FormMultipleImagePicker } from "./FormMultipleImagePicker";
export type {
  FormMultipleImagePickerProps,
  FormMultipleImageValue,
} from "./FormMultipleImagePicker";

export { SubmitButton } from "./SubmitButton";
export type { SubmitButtonProps } from "./SubmitButton";

export { FormTextarea } from "./FormTextarea";
export type { FormTextareaProps } from "./FormTextarea";

export * from "./FormVideoPicker";
export * from "./FormMultipleVideoPicker";
export * from "./FormMarkdownEditor";
export { FormSelect } from "./FormSelect";
export type { FormSelectProps } from "./FormSelect";
export { FormSwitch } from "./FormSwitch";
export type { FormSwitchProps } from "./FormSwitch";
export * from "./FormContainer";

```
