# Error Handling, Retry Logic & Rate Limiting

## 1. mudbase HTTP error codes

| Code | Meaning | Client action |
|------|---------|---------------|
| `400` | Bad Request, invalid body or params | Fix the request; don't retry |
| `401` | Unauthorized, missing or invalid token | Refresh token or redirect to login |
| `403` | Forbidden, insufficient permissions | Show permission error to user |
| `404` | Not Found, resource does not exist | Show not-found UI |
| `429` | Too Many Requests, rate limit hit | Respect `Retry-After` header; exponential backoff |
| `5xx` | Server error | Retry with backoff (3 attempts max) |

## 2. `MudbaseError`, typed error class

The client in `sdk-client.md` already throws `MudbaseError`. Use it for targeted handling:

```typescript
import { MudbaseError } from '@/lib/mudbase'

async function loadDashboard() {
  try {
    const data = await getMudbaseClient().getDocuments('orders')
    return data
  } catch (err) {
    if (!(err instanceof MudbaseError)) throw err

    switch (err.statusCode) {
      case 401:
        // Token expired, refresh session
        await getMudbaseClient().getSession()
        return loadDashboard() // retry once
      case 403:
        router.push('/unauthorized')
        return null
      case 404:
        return { data: [], pagination: { page: 1, limit: 20, total: 0, totalPages: 0, hasMore: false } }
      case 429:
        // Retry-After is in the response headers, handled by retryWithBackoff below
        throw err
      default:
        if (err.statusCode >= 500) throw err
        throw err
    }
  }
}
```

## 3. Exponential backoff with `Retry-After`

```typescript
// src/lib/retry.ts
import { getMudbaseClient, MudbaseError } from '@/lib/mudbase'

interface RetryOptions {
  maxAttempts?: number
  initialDelayMs?: number
  maxDelayMs?: number
}

export async function retryWithBackoff<T>(
  fn: () => Promise<T>,
  options: RetryOptions = {}
): Promise<T> {
  const { maxAttempts = 3, initialDelayMs = 500, maxDelayMs = 30_000 } = options
  let attempt = 0

  while (attempt < maxAttempts) {
    try {
      return await fn()
    } catch (err) {
      attempt++
      if (attempt >= maxAttempts) throw err

      if (!(err instanceof MudbaseError)) throw err
      const isRetryable = err.statusCode === 429 || err.statusCode >= 500
      if (!isRetryable) throw err

      // Respect Retry-After if present
      const retryAfter = (err.details as { retryAfter?: number } | undefined)?.retryAfter
      const delay = retryAfter
        ? retryAfter * 1000
        : Math.min(initialDelayMs * 2 ** (attempt - 1), maxDelayMs)

      await new Promise((resolve) => setTimeout(resolve, delay))
    }
  }

  throw new Error('Max retry attempts reached')
}

// Usage
async function loadOrdersWithRetry(): Promise<unknown> {
  return retryWithBackoff(() => getMudbaseClient().getDocuments('orders'))
}
```

## 4. React Query retry config for mudbase

```typescript
// src/lib/query-client.ts
import { QueryClient } from '@tanstack/react-query'
import { MudbaseError } from '@/lib/mudbase'

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      retry: (failureCount, error) => {
        if (error instanceof MudbaseError) {
          // Never retry 4xx errors (fix the request)
          if (error.statusCode >= 400 && error.statusCode < 500) return false
          // Retry up to 3 times on 5xx / network errors
          return failureCount < 3
        }
        return failureCount < 2
      },
      retryDelay: (attempt) => Math.min(500 * 2 ** attempt, 15_000),
      staleTime: 1000 * 30,
    },
    mutations: {
      retry: false, // never auto-retry mutations, the user should confirm
    },
  },
})
```

## 5. Rate Limiting: Client-Side Strategies

mudbase rate limiting is **windowed and per-IP**, applied by layered limiter middleware, not a per-second per-project token bucket. Several buckets stack: the general API limiter runs on everything under `/api`, and the more specific limiter for the route you hit runs as well, so an auth call is bounded by both.

Defaults (each is env-tunable per deployment, and the org-adjustable ones can be raised per organization from the console under `org.settings.rateLimits`):

| Bucket | Window | Default max | Org-adjustable |
|--------|--------|-------------|----------------|
| General API (`/api/*`) | 15 min | 2000 | yes |
| Auth (`/api/auth/*`) | 15 min | 20 | no |
| Login (`/api/auth/local/login`) | 15 min | 5 | no |
| Registration (`/api/auth/local/register`) | 60 min | 3 | no |
| Password-reset request | 15 min | per-project limiter, tight | no |
| Password-reset OTP confirm | 5 min | 3 | no |
| Token refresh | 15 min | 60 | no |
| Data mutations (create/update/delete document) | 1 min | 600 | yes |
| File upload | 60 min | 500 | yes |
| Collection creation | 15 min | 200 | yes |
| Project creation | 60 min | 30 | no |
| API-key creation | 60 min | 50 | no |
| Webhook trigger | 15 min | 120 | yes |
| Public endpoints | 15 min | 200 | no |

Two consequences for client design:

- The binding constraint on a bulk import is the **data-mutation** bucket, 600 writes/minute, roughly 10/second sustained, not the general 2000/15min figure. Size batches against that.
- Auth limits are per-IP. Users behind a shared NAT (an office, a mobile carrier gateway) share the budget, so a login form that retries automatically on failure can lock out an entire building. Never auto-retry a `401`.

### Detecting and handling 429

```typescript
// In MudbaseClient.request(), add Retry-After header handling
if (res.status === 429) {
  const retryAfter = parseInt(res.headers.get('Retry-After') ?? '1', 10)
  const error = (await res.json().catch(() => ({}))) as Record<string, unknown>
  throw new MudbaseError('Rate limit exceeded', 429, { retryAfter, ...error })
}
```

### Client-side request queue

For bulk operations (e.g. importing 500 documents), batch requests and respect the rate limit:

```typescript
// src/lib/request-queue.ts
import { getMudbaseClient } from '@/lib/mudbase'

export async function batchWithRateLimit<T>(
  items: T[],
  fn: (item: T) => Promise<unknown>,
  options: { batchSize?: number; delayMs?: number } = {}
): Promise<void> {
  const { batchSize = 10, delayMs = 200 } = options

  for (let i = 0; i < items.length; i += batchSize) {
    const batch = items.slice(i, i + batchSize)
    await Promise.all(batch.map(fn))
    if (i + batchSize < items.length) {
      await new Promise((resolve) => setTimeout(resolve, delayMs))
    }
  }
}

// Usage: import 500 documents inside the data-mutation budget (600 writes / 60s).
// 8 writes per 1000ms = 480/min, comfortably under the limit with headroom for
// whatever else the app is doing on the same IP.
async function importProducts(documents: Record<string, unknown>[]): Promise<void> {
  await batchWithRateLimit(
    documents,
    (doc) => getMudbaseClient().createDocument('products', doc),
    { batchSize: 8, delayMs: 1000 }
  )
}
```

### Debounce on search / filter inputs (corrected)

The data endpoint has no `search` param, and (see `collections-and-data.md`) `$regex` is on the query sanitizer's blocklist and always returns `400`, so escaping the input does not make it usable. Do not build a debounced search against `filter.$regex`. Instead, fetch a bounded, real-filtered batch and match the substring client-side:

```typescript
// src/hooks/useDebouncedSearch.ts
import { useState, useEffect } from 'react'

interface DebouncedSearch {
  query: string
  setQuery: (next: string) => void
  debouncedQuery: string
}

export function useDebouncedSearch(delay = 300): DebouncedSearch {
  const [query, setQuery] = useState('')
  const [debouncedQuery, setDebouncedQuery] = useState('')

  useEffect(() => {
    const timer = setTimeout(() => setDebouncedQuery(query), delay)
    return () => clearTimeout(timer)
  }, [query, delay])

  return { query, setQuery, debouncedQuery }
}
```

```typescript
// src/components/SearchBar.tsx
'use client'

import { useMemo } from 'react'
import { useDebouncedSearch } from '@/hooks/useDebouncedSearch'
import { useDocuments } from '@/hooks/useCollection'
import type { Document } from '@/lib/mudbase'

interface Searchable extends Document {
  title: string
}

export function SearchBar({ collectionId }: { collectionId: string }) {
  const { query, setQuery, debouncedQuery } = useDebouncedSearch(300)

  // No server-side $regex: fetch a bounded, real-filtered batch and match client-side.
  // Widen the real filters (status, category, owner) as far as your use case allows
  // before relying on this; 100 is the endpoint's hard per-request cap.
  const { data } = useDocuments<Searchable>(collectionId, {
    filter: {},
    limit: 100,
  })

  const results = useMemo(() => {
    const needle = debouncedQuery.trim().toLowerCase()
    const all = data?.data ?? []
    if (!needle) return all
    return all.filter((doc) => doc.title.toLowerCase().includes(needle))
  }, [data, debouncedQuery])

  return (
    <div>
      <input
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder="Search"
        className="w-full rounded-md border px-3 py-2"
      />
      <p className="mt-1 text-sm text-gray-500">{results.length} results</p>
    </div>
  )
}
```

For search over more than one bounded page, or across collections, use `/api/search` (see `collections-and-data.md`) instead of trying to widen this pattern.

### Socket.IO connection limit

mudbase enforces a plan-based socket connection limit. Handle the `error` event:

```typescript
// In MudbaseSocket.connect(), already handled:
this.socket.on('error', (data: { message: string; limit?: number }) => {
  if (data.message.includes('connection limit')) {
    // Notify the user, they cannot add more sockets on this plan
    showToast(`Realtime connection limit reached (${data.limit} max on your plan)`, 'warning')
  }
  this.socket?.disconnect()
})
```

## See also

- `collections-and-data.md`, why `$regex` is blocked and the bounded-batch pattern used above
- `sdk-client.md`, the `MudbaseError` class and base `request()` method
