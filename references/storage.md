# File Storage

## 1. The two storage surfaces

mudbase exposes storage through two distinct route trees. Pick one per use case; they are not interchangeable.

| Surface | Mount | Addressing | Use for |
|---------|-------|-----------|---------|
| Buckets | `/api/bucket` | `projects/:projectId/buckets/:bucketId/...`, `bucketId` is an ObjectId | Managed buckets with per-bucket size caps, allowed-type lists, ACLs, image transforms |
| Flat files | `/api/files` | `projectId` + a bucket **name** string in the body | Direct-to-storage presigned uploads, download URLs |

Key contract details that differ from most BaaS platforms:

- The multipart field name for a bucket upload is **`files`** (plural, array), max 10 per request, not `file`.
- The presigned flow returns an **S3-style presigned POST**, not a PUT URL. You must send a `multipart/form-data` POST with every entry of the returned `fields` object appended *before* the file part. A raw `PUT` to that URL fails. (This is wire-protocol detail you need to integrate correctly, not something to expose in product copy.)
- A signed download URL is obtained with a **POST** carrying `{ expiresIn }`, and the response field is `signedUrl` (plus `expiresAt`), not `url`.
- Uploading into a public bucket does not force the object public, pass `isPublic` per upload to store a private object inside a public bucket.
- mudbase refuses to sign a public object: for `isPublic: true` files the download endpoint returns the permanent public URL plus a `warning`, because signing a world-readable object buys nothing.

## 2. Direct Upload (simple, for small files)

```typescript
// src/hooks/useFileUpload.ts
import { useState, useCallback } from 'react'
import { useMudbase } from '@/lib/mudbase-provider'
import type { FileObject } from '@/lib/mudbase'

interface UploadState {
  uploading: boolean
  progress: number
  error: string | null
  file: FileObject | null
}

interface UseFileUploadResult extends UploadState {
  upload: (file: File, isPublic?: boolean) => Promise<FileObject | null>
  reset: () => void
}

export function useFileUpload(bucketId: string): UseFileUploadResult {
  const { client } = useMudbase()
  const [state, setState] = useState<UploadState>({
    uploading: false,
    progress: 0,
    error: null,
    file: null,
  })

  const upload = useCallback(
    async (file: File, isPublic = false): Promise<FileObject | null> => {
      const token = client.getToken()
      if (!token) {
        setState({ uploading: false, progress: 0, error: 'Not authenticated', file: null })
        return null
      }

      setState({ uploading: true, progress: 0, error: null, file: null })
      try {
        // fetch() has no upload-progress event, so bucket uploads go through XHR.
        const uploaded = await uploadWithProgress(
          { token, projectId: client.getProjectId(), baseUrl: client.getBaseUrl() },
          bucketId,
          file,
          isPublic,
          (p) => setState((prev) => ({ ...prev, progress: p }))
        )
        setState({ uploading: false, progress: 100, error: null, file: uploaded })
        return uploaded
      } catch (err: unknown) {
        const msg = err instanceof Error ? err.message : 'Upload failed'
        setState({ uploading: false, progress: 0, error: msg, file: null })
        return null
      }
    },
    [client, bucketId]
  )

  const reset = useCallback((): void => {
    setState({ uploading: false, progress: 0, error: null, file: null })
  }, [])

  return { ...state, upload, reset }
}

interface UploadContext {
  token: string
  projectId: string
  baseUrl: string
}

interface BucketUploadResponse {
  success: boolean
  files: FileObject[]
}

function uploadWithProgress(
  ctx: UploadContext,
  bucketId: string,
  file: File,
  isPublic: boolean,
  onProgress: (percent: number) => void
): Promise<FileObject> {
  return new Promise<FileObject>((resolve, reject) => {
    const formData = new FormData()
    // Field name is `files`, multer is configured as .array("files", 10).
    formData.append('files', file)
    formData.append('isPublic', String(isPublic))

    const xhr = new XMLHttpRequest()
    xhr.open(
      'POST',
      `${ctx.baseUrl}/api/bucket/projects/${ctx.projectId}/buckets/${bucketId}/files`
    )
    xhr.setRequestHeader('Authorization', `Bearer ${ctx.token}`)

    xhr.upload.onprogress = (e: ProgressEvent): void => {
      if (e.lengthComputable) {
        onProgress(Math.round((e.loaded / e.total) * 100))
      }
    }

    xhr.onload = (): void => {
      let parsed: unknown
      try {
        parsed = JSON.parse(xhr.responseText || '{}')
      } catch {
        reject(new Error(`Upload failed: unreadable response (status ${xhr.status})`))
        return
      }

      if (xhr.status >= 200 && xhr.status < 300) {
        const body = parsed as BucketUploadResponse
        const uploaded = body.files?.[0]
        if (!uploaded) {
          reject(new Error('Upload succeeded but returned no file record'))
          return
        }
        resolve(uploaded)
        return
      }

      // 413 carries maxFileUploadBytes; 400 carries a MIME-policy `code`.
      const body = parsed as { error?: string; maxFileUploadBytes?: number }
      const detail = body.maxFileUploadBytes
        ? ` (max ${body.maxFileUploadBytes} bytes)`
        : ''
      reject(new Error(`${body.error ?? `Upload failed with status ${xhr.status}`}${detail}`))
    }

    xhr.onerror = (): void => reject(new Error('Network error during upload'))
    xhr.send(formData)
  })
}
```

The `getProjectId()` / `getBaseUrl()` accessors above are two one-line additions to `MudbaseClient`:

```typescript
// src/lib/mudbase.ts, inside class MudbaseClient
  getProjectId(): string {
    return this.projectId
  }

  getBaseUrl(): string {
    return this.baseUrl
  }
```

## 3. Presigned Upload (production pattern for large files)

Three steps: ask mudbase for a presigned POST, POST the file straight to object storage, then tell mudbase the object landed so it can virus-scan it and create the metadata record.

```typescript
// src/lib/presigned-upload.ts
import { getMudbaseClient } from '@/lib/mudbase'

export interface PresignedUploadResult {
  fileId: string
  key: string
}

export async function uploadWithPresignedPost(
  file: File,
  options: { bucket?: string; isPublic?: boolean } = {}
): Promise<PresignedUploadResult> {
  const client = getMudbaseClient()

  // 1. Ask mudbase for the presigned POST. mudbase derives the object key itself,
  //    the client never picks it, which is what stops key-collision and path traversal.
  const presigned = await client.getPresignedUploadUrl({
    bucket: options.bucket ?? 'default',
    originalName: file.name,
    contentType: file.type,
    isPublic: options.isPublic ?? false,
  })

  if (file.size > presigned.maxFileUploadBytes) {
    throw new Error(
      `File is ${file.size} bytes; this plan allows ${presigned.maxFileUploadBytes}.`
    )
  }

  // 2. POST directly to object storage. The `fields` entries are the signed policy and
  //    MUST be appended before the file part, the storage backend ignores anything after `file`.
  const form = new FormData()
  for (const [name, value] of Object.entries(presigned.fields)) {
    form.append(name, value)
  }
  form.append('file', file)

  const uploadRes = await fetch(presigned.url, { method: 'POST', body: form })
  if (!uploadRes.ok) {
    throw new Error(`Presigned upload failed: ${uploadRes.status} ${uploadRes.statusText}`)
  }

  // 3. Confirm, keyed by the object key, not a client-generated file id. This is the
  //    step that scans the object and writes the File record; skip it and the upload
  //    is an orphaned blob mudbase knows nothing about.
  const confirmed = await client.confirmUpload({
    key: presigned.key,
    originalName: file.name,
    contentType: file.type,
    size: file.size,
    bucket: options.bucket ?? 'default',
    isPublic: options.isPublic ?? false,
  })

  return { fileId: confirmed.fileId, key: presigned.key }
}
```

A quarantined file (virus scan hit) comes back as `400` with `{ message: "File quarantined", details }`, surface that distinctly from a generic upload failure, because retrying will not help.

## 4. File Upload Component

```typescript
// src/components/FileUpload.tsx
'use client'

import { useRef, useState } from 'react'
import { useFileUpload } from '@/hooks/useFileUpload'
import { useMudbase } from '@/lib/mudbase-provider'

interface FileUploadProps {
  bucketId: string
  accept?: string
  maxSizeMB?: number
  isPublic?: boolean
  onUploaded?: (url: string, fileId: string) => void
}

export function FileUpload({
  bucketId,
  accept = '*/*',
  maxSizeMB = 10,
  isPublic = false,
  onUploaded,
}: FileUploadProps) {
  const inputRef = useRef<HTMLInputElement>(null)
  const { client } = useMudbase()
  const { upload, uploading, progress, error, file, reset } = useFileUpload(bucketId)
  const [localError, setLocalError] = useState<string | null>(null)

  const handleFileChange = async (e: React.ChangeEvent<HTMLInputElement>): Promise<void> => {
    const selected = e.target.files?.[0]
    if (!selected) return

    setLocalError(null)
    if (selected.size > maxSizeMB * 1024 * 1024) {
      setLocalError(`File must be under ${maxSizeMB}MB`)
      return
    }

    const uploaded = await upload(selected, isPublic)
    if (!uploaded || !onUploaded) return

    if (uploaded.isPublic) {
      // Public objects already carry a permanent URL; mudbase declines to sign them.
      onUploaded(uploaded.url, uploaded.id)
      return
    }

    try {
      const signedUrl = await client.getSignedUrl(bucketId, uploaded.id)
      onUploaded(signedUrl, uploaded.id)
    } catch (err: unknown) {
      setLocalError(err instanceof Error ? err.message : 'Could not sign the uploaded file')
    }
  }

  const message = error ?? localError

  return (
    <div className="space-y-2">
      <input
        ref={inputRef}
        type="file"
        accept={accept}
        onChange={handleFileChange}
        disabled={uploading}
        className="hidden"
      />
      <button
        type="button"
        onClick={() => {
          reset()
          setLocalError(null)
          inputRef.current?.click()
        }}
        disabled={uploading}
        className="w-full rounded-md border border-dashed px-4 py-8 text-sm text-gray-500 hover:bg-gray-50 disabled:opacity-50"
      >
        {uploading
          ? `Uploading... ${progress}%`
          : file
            ? `Uploaded: ${file.originalName}`
            : 'Click to upload file'}
      </button>
      {uploading && (
        <div className="h-2 w-full rounded bg-gray-200">
          <div
            className="h-full rounded bg-blue-500 transition-all"
            style={{ width: `${progress}%` }}
          />
        </div>
      )}
      {message && <p className="text-sm text-red-600">{message}</p>}
    </div>
  )
}
```

## See also

- `sdk-client.md`, the `getPresignedUploadUrl`/`confirmUpload`/`getSignedUrl` client methods and their type signatures
- `mcp-and-ai-agents.md`, the standalone MCP server's file tools cover the flat-files surface read path (list/get/upload/delete/download-url) for agent use
