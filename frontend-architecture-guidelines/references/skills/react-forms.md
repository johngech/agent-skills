---
name: project-form-architecture
description: How forms are built in this project. Use when creating or modifying any form, form field, validation schema, or submit flow. The form structure is built on React Hook Form + Zod.
version: 1.2.0
author: Yohannes Getachew
type: skill
category: development
tags: [forms, react-hook-form, zod, validation, ui]
---

# Project Form Architecture

**Purpose**: Teach the agent how forms are built in this project so every new form matches the existing architecture instead of inventing a new one.

This project already contains a complete form abstraction built on top of **React Hook Form** and **Zod**.

## Core Golden Rules

1. **Always reuse existing components**: You MUST use `Form`, `FormField`, `SubmitButton`, `ImageInput`, `Input`, and `Button`.
2. **Never build raw HTML forms**: Never generate `<form>` tags. Use `<Form>`.
3. **Never create raw HTML inputs**: Never generate `<input />`. Use `<FormField />` (inside forms) or `<Input />` (outside forms).
4. **Never manually display validation errors**: The `FormField` handles this automatically via the Zod resolver.
5. **Never create custom loading buttons**: Do not use `<button disabled={{loading}}>`. Use `<SubmitButton />` which binds to the form state.
6. **Never use React Hook Form primitives directly**: Do not use `useForm()`, `Controller`, or `register()` directly in feature forms.

## Usage Examples

### Standard Form Setup

```tsx
import { Form, FormField, SubmitButton } from "@/components/forms";
import { z } from "zod";

const loginSchema = z.object({
  email: z.string().email(),
  password: z.string().min(6),
});

export function LoginForm() {
  const handleSubmit = async (data: z.infer<typeof loginSchema>) => {
    // API call here
  };

  return (
    <Form
      initialValues={{ email: "", password: "" }}
      validationSchema={loginSchema}
      onSubmit={handleSubmit}
    >
      <FormField name="email" label="Email" type="email" />
      <FormField name="password" label="Password" type="password" />
      <SubmitButton title="Login" />
    </Form>
  );
}
```

### Form Component Hierarchy

```text
Feature Form
      |
      v
    Form  (Handles useForm, Zod resolver, context)
      |
      +----------------+
      |                |
      v                v
  FormField       FormImagePicker
      |
      v
 SubmitButton  (Auto-handles isSubmitting)
```

## Core Implementation References

Review the underlying infrastructure that powers this form architecture to understand how `react-hook-form` and `zod` are integrated:

**Forms Core:**

- [Form.md](../forms/Form.md) (The root provider handling form context and submit lifecycle)
- [FormField.md](../forms/FormField.md) (Connects standard inputs to the form context)
- [SubmitButton.md](../forms/SubmitButton.md) (Automatically manages loading and validation states)
- [FormImagePicker.md](../forms/FormImagePicker.md) (Connects single image upload with form state)
- [index.md](../forms/index.md) (Barrel exports)

**UI Base Components:**

- [Input.md](../ui/input.md) (The raw styled input component)
- [Button.md](../ui/button.md) (The raw styled button component)
- [ImageInput.md](../ui/ImageInput.md) (Handles file previews and clear actions)
