# Collections & Data

## 1. The REST contract

All document CRUD lives under one path family. There is no `/api/collections/...` route.

| Operation | Method + path |
|-----------|---------------|
| List | `GET /api/data/projects/:projectId/collections/:collectionId/data` |
| Read one | `GET /api/data/projects/:projectId/collections/:collectionId/data/:documentId` |
| Create | `POST /api/data/projects/:projectId/collections/:collectionId/data` |
| Update | `PATCH /api/data/projects/:projectId/collections/:collectionId/data/:documentId` |
| Delete | `DELETE /api/data/projects/:projectId/collections/:collectionId/data/:documentId` |

Query parameters on the list endpoint:

| Param | Type | Notes |
|-------|------|-------|
| `filter` | JSON string | A raw Mongo-style query object, server-sanitized before execution. Not an operator array. |
| `sort` | string | Mongoose sort syntax, e.g. `-createdAt`. Defaults to `-createdAt`. |
| `page` | number | 1-indexed. |
| `limit` | number | Default 20, clamped to a maximum of 100. |
| `fields` | CSV | Projection allow-list, e.g. `title,slug,publishedAt`. |

There is **no** `search` parameter here, full-text search is a separate surface (`/api/search`). See "Substring matching" below before reaching for `$regex`, it will not do what you expect.

Response shapes:

```jsonc
// GET .../data
{ "data": [ /* documents */ ], "pagination": { "page": 1, "limit": 20, "total": 57, "totalPages": 3, "hasMore": true } }

// GET .../data/:documentId
{ "data": { "_id": "...", "...": "..." } }

// POST / PATCH .../data
{ "message": "Data created successfully", "data": { "_id": "...", "...": "..." } }

// DELETE .../data/:documentId
{ "message": "Data deleted successfully" }
```

Create and update take the document body **flat**, do not nest it under a `data` key and do not send `projectId` in the body; the project is already in the path. Collection-level permission conditions (`$userId`, `$orgId`, `$userCustomRole`) are auto-populated server-side on create and rejected if the client tries to assign a record to somebody else.

## 2. Substring matching: `$regex` is blocked, do not use it

The `filter` object passed to the list endpoint goes through a query sanitizer before it reaches the database, and that sanitizer keeps a hard blocklist of Mongo operators it considers dangerous for a client-supplied filter. **`$regex` (and `$text`'s `$search`) are on that blocklist and always return `400`, regardless of whether the pattern is anchored.** Anchoring a pattern with `^` does not help, does not "unblock" the operator, and does not change the request outcome; the sanitizer rejects the operator itself, not the pattern shape.

Practical consequence: there is no server-side substring or prefix search on collection data through this endpoint. For a typeahead, filter box, or "find as you type" feature, do one of:

1. **Bounded fetch + client-side match.** Use real, allowed filters (`status`, `tags`, ranges, an exact-match field) to narrow the result set, cap it with `limit: 100`, and substring-match the returned page in the browser or in your own backend. This is the right approach for anything scoped to a single user, org, or small collection.
2. **The dedicated search surface**, `/api/search`, for anything that needs to search across a whole collection or across collections. That is a separate indexed surface, not a `filter` trick.

Do not write an `escapeRegex()` helper and pass the escaped pattern through `filter.$regex` expecting it to work once "safely escaped", the operator is rejected outright, escaping does not change that.

## 3. Schema Design Patterns

mudbase collections use JSON-schema style definitions. Common patterns:

```
Collection: user_profiles
Fields: userId (string, unique ref to auth user), avatar (string), bio (string),
        location (string), socialLinks (object), onboardingComplete (boolean)

Collection: posts
Fields: authorId (string), title (string), slug (string, unique), content (string),
        status (enum: draft|published|archived), tags (array<string>),
        publishedAt (datetime), viewCount (number)

Collection: orders
Fields: userId (string), items (array<OrderItem>), subtotal (number),
        currency (string), status (enum: pending|paid|shipped|delivered|cancelled),
        paymentRef (string), shippingAddress (object), metadata (object)
```

## 4. React Query Hooks, `src/hooks/useCollection.ts`

```typescript
// src/hooks/useCollection.ts
import {
  useQuery,
  useMutation,
  useQueryClient,
  type UseQueryOptions,
} from '@tanstack/react-query'
import { useMudbase } from '@/lib/mudbase-provider'
import type { Document, ListResponse, QueryParams } from '@/lib/mudbase'

// useDocuments

export function useDocuments<T extends Document = Document>(
  collectionId: string,
  query?: QueryParams,
  options?: Omit<UseQueryOptions<ListResponse<T>>, 'queryKey' | 'queryFn'>
) {
  const { client } = useMudbase()
  return useQuery<ListResponse<T>>({
    queryKey: ['collection', collectionId, query],
    queryFn: () => client.getDocuments<T>(collectionId, query),
    staleTime: 1000 * 30,
    ...options,
  })
}

// useDocument

export function useDocument<T extends Document = Document>(
  collectionId: string,
  documentId: string | null | undefined,
  options?: Omit<UseQueryOptions<T>, 'queryKey' | 'queryFn'>
) {
  const { client } = useMudbase()
  return useQuery<T>({
    queryKey: ['collection', collectionId, 'doc', documentId],
    queryFn: () => {
      if (!documentId) throw new Error('useDocument called without a documentId')
      return client.getDocument<T>(collectionId, documentId)
    },
    enabled: !!documentId,
    staleTime: 1000 * 30,
    ...options,
  })
}

// useCreateDocument

export function useCreateDocument<T extends Document = Document>(collectionId: string) {
  const { client } = useMudbase()
  const queryClient = useQueryClient()

  return useMutation<T, Error, Record<string, unknown>>({
    mutationFn: (data) => client.createDocument<T>(collectionId, data),
    onSuccess: (created) => {
      queryClient.invalidateQueries({ queryKey: ['collection', collectionId] })
      queryClient.setQueryData(['collection', collectionId, 'doc', created._id], created)
    },
  })
}

// useUpdateDocument

export function useUpdateDocument<T extends Document = Document>(collectionId: string) {
  const { client } = useMudbase()
  const queryClient = useQueryClient()

  return useMutation<T, Error, { documentId: string; data: Record<string, unknown> }>({
    mutationFn: ({ documentId, data }) =>
      client.updateDocument<T>(collectionId, documentId, data),
    onMutate: async ({ documentId, data }) => {
      await queryClient.cancelQueries({
        queryKey: ['collection', collectionId, 'doc', documentId],
      })
      const previous = queryClient.getQueryData<T>([
        'collection',
        collectionId,
        'doc',
        documentId,
      ])
      if (previous) {
        queryClient.setQueryData(['collection', collectionId, 'doc', documentId], {
          ...previous,
          ...data,
        })
      }
      return { previous, documentId }
    },
    onError: (_err, { documentId }, context) => {
      const ctx = context as { previous: T | undefined; documentId: string }
      if (ctx?.previous) {
        queryClient.setQueryData(
          ['collection', collectionId, 'doc', documentId],
          ctx.previous
        )
      }
    },
    onSettled: (_data, _err, { documentId }) => {
      queryClient.invalidateQueries({ queryKey: ['collection', collectionId, 'doc', documentId] })
      queryClient.invalidateQueries({ queryKey: ['collection', collectionId] })
    },
  })
}

// useDeleteDocument

export function useDeleteDocument(collectionId: string) {
  const { client } = useMudbase()
  const queryClient = useQueryClient()

  return useMutation<void, Error, string>({
    mutationFn: (documentId) => client.deleteDocument(collectionId, documentId),
    onSuccess: (_data, documentId) => {
      queryClient.removeQueries({
        queryKey: ['collection', collectionId, 'doc', documentId],
      })
      queryClient.invalidateQueries({ queryKey: ['collection', collectionId] })
    },
  })
}
```

## 5. Query Parameters Example (corrected)

```typescript
// src/app/blog/page.tsx
import { useDocuments } from '@/hooks/useCollection'
import type { Document } from '@/lib/mudbase'

interface Post extends Document {
  title: string
  slug: string
  status: 'draft' | 'published' | 'archived'
  tags: string[]
  publishedAt: string
  viewCount: number
}

// Published posts tagged "typescript", newest first, page 2 of 10-per-page.
// `filter` is a plain Mongo query object, array membership is a bare equality match.
const { data: page2, isLoading } = useDocuments<Post>('posts', {
  filter: { status: 'published', tags: 'typescript' },
  sort: '-publishedAt',
  page: 2,
  limit: 10,
})

// Operators are written the Mongo way, not as a DSL.
const { data: popular } = useDocuments<Post>('posts', {
  filter: { status: 'published', viewCount: { $gte: 1000 } },
  sort: '-viewCount',
  limit: 20,
})

// There is no server-side substring search: $regex is blocked by the query sanitizer
// and always returns 400, anchored or not. Narrow with real filters, cap the batch,
// and match the substring client-side.
function useTitleSearch(term: string) {
  const { data: batch } = useDocuments<Post>('posts', {
    filter: { status: 'published' },
    sort: '-publishedAt',
    limit: 100,
  })
  const needle = term.trim().toLowerCase()
  const matches = needle
    ? (batch?.data ?? []).filter((post) => post.title.toLowerCase().includes(needle))
    : (batch?.data ?? [])
  return matches
}

// Trim the payload to the columns a list view actually renders.
const { data: index } = useDocuments<Post>('posts', {
  filter: { status: 'published' },
  fields: ['title', 'slug', 'publishedAt'],
  sort: '-publishedAt',
  limit: 50,
})

const total: number = page2?.pagination.total ?? 0
const hasNextPage: boolean = page2?.pagination.hasMore ?? false
const posts: Post[] = page2?.data ?? []
```

## See also

- `sdk-client.md`, the `getDocuments`/`createDocument`/etc. methods these hooks call
- `graphql.md`, for read-only querying across relationships instead of client-side joins
- `error-handling-and-limits.md`, rate limits and the debounced-search pattern (also affected by the `$regex` block)
