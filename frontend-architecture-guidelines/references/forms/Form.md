# Form.tsx

**Why this pattern?**
> The root provider component. Encapsulates react-hook-form setup and zod validation resolving.

## Implementation

```tsx
import { type ReactNode } from "react";
import { zodResolver } from "@hookform/resolvers/zod";
import {
  FormProvider,
  useForm,
  type DefaultValues,
  type FieldValues,
  type Mode,
  type SubmitHandler,
} from "react-hook-form";
import type { z } from "zod";

export interface FormProps<
  TSchema extends z.ZodType<FieldValues, FieldValues>,
> {
  /** Initial values, typed as the schema's input type. */
  initialValues: z.input<TSchema>;
  /** Zod schema used as the single source of truth for validation. */
  validationSchema: TSchema;
  /** Called with the parsed (output) values when the form is valid. */
  onSubmit: SubmitHandler<z.output<TSchema>>;
  children: ReactNode;
  className?: string;
  id?: string;
  /** React Hook Form validation mode. Defaults to "onSubmit". */
  mode?: Mode;
}

export function Form<TSchema extends z.ZodType<FieldValues, FieldValues>>({
  initialValues,
  validationSchema,
  onSubmit,
  children,
  className,
  id,
  mode = "onSubmit",
}: Readonly<FormProps<TSchema>>) {
  const methods = useForm<z.input<TSchema>, unknown, z.output<TSchema>>({
    resolver: zodResolver(validationSchema),
    defaultValues: initialValues as DefaultValues<z.input<TSchema>>,
    mode,
  });

  return (
    <FormProvider {...methods}>
      <form
        id={id}
        className={className}
        noValidate
        onSubmit={methods.handleSubmit(onSubmit)}
      >
        {children}
      </form>
    </FormProvider>
  );
}

```
