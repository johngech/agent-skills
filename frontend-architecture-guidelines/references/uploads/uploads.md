# File Upload Service (`uploads.ts`)

Located at: `src/api/uploads.ts` or `src/lib/uploads.ts`

## Purpose

Centralizes `multipart/form-data` uploads for images and documents to the backend storage service using the typed `apiClient` factory.

---

## Implementation

```typescript
import { apiClient } from "@/api/client";

/** Which bucket folder the files belong to. */
export type EntityType =
  | "members"
  | "users"
  | "documents"
  | "misc";

export interface UploadedFile {
  key: string;
  url: string;
}

interface UploadImagesResponse {
  images: UploadedFile[];
}

/**
 * Uploads image files to the backend and returns their public URLs.
 */
export async function uploadImages(
  files: File[],
  entityType: EntityType = "misc",
): Promise<UploadedFile[]> {
  if (files.length === 0) return [];

  const formData = new FormData();
  files.forEach((file) => formData.append("images", file));
  formData.append("entityType", entityType);

  const client = apiClient<UploadImagesResponse>("/uploads/images");

  const response = await client.create(formData, {
    headers: { "Content-Type": "multipart/form-data" },
  });

  return response.data?.images || [];
}
```
