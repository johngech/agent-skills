---
name: project-file-uploads
description: How image and video file uploads are handled in this project. Use when building forms that include image or video upload fields. Enforces the two-step upload pattern (upload media first, then submit JSON payload).
version: 1.0.0
author: Yohannes Getachew
type: skill
category: development
tags: [uploads, forms, media, structure]
---

# Project File Upload Architecture

This project handles file uploads (images and videos) via a **two-step upload pattern**.

The main feature endpoints (like `/api/news` or `/api/competitions`) **only accept JSON**. They do not accept `multipart/form-data`. Therefore, you cannot send raw `File` objects directly to these endpoints.

Instead, you must first upload the files to a dedicated media endpoint using the `uploadImages` or `uploadVideos` utilities, receive the public URLs back, and then include those URLs in your final JSON payload.

## Core Golden Rules

1. **Never send `File` objects in JSON payloads**: Your main CRUD mutations (e.g., `useCreateNews`) should only receive arrays of strings (`image_urls: string[]`).
2. **Use the `entityType` correctly**: When calling `uploadImages(files, entityType)`, the `entityType` must match the feature you are building (e.g., `"news"`, `"competitions"`, `"users"`). This acts as a bucket folder parameter for the backend.
3. **Preserve existing URLs on edit**: When editing an entity, the form will contain a mix of existing string URLs and new `File` objects. Ensure you map them correctly so existing images are not lost.

## Usage Example (Two-Step Upload Pattern)

Inside your React Hook Form's `onSubmit` handler, intercept the payload to handle the uploads first:

```tsx
import { uploadImages } from "@/api/uploads";
import { useCreateNews } from "../queries";

// Inside your form component...
const onSubmit = async (data: NewsFormData) => {
  try {
    const entries = data.images || [];
    
    // 1. Separate new files from existing URLs
    const newFiles = entries.filter((entry): entry is File => entry instanceof File);

    // 2. Upload the new files to the dedicated endpoint
    const uploadedImages = await uploadImages(newFiles, "news");

    // 3. Reconstruct the array, preserving order
    let uploadIndex = 0;
    const images = entries.map((entry) => {
      if (typeof entry === "string") {
        return entry.length > 0 ? entry : null; // Existing image
      }
      return uploadedImages[uploadIndex++]?.url ?? null; // Newly uploaded image
    }).filter((url): url is string => url !== null);

    // 4. Send the final JSON payload
    const payload = {
      title: data.title,
      image_urls: images.length > 0 ? images : undefined,
    };

    await createMutation.mutateAsync(payload);
  } catch (err) {
    // Handle error...
  }
};
```

## Core Implementation References

Review the underlying utility that powers this pattern:

- [uploads.md](../uploads/uploads.md)
