# GraphQL (auto-generated, read-only)

Every project gets a GraphQL schema for free, generated live from its collections and declared relationships. There is nothing to write or deploy, the schema exists the moment you have collections.

**This phase is read-only.** There is a `Query` type only, no `Mutation` and no `Subscription`. Use the REST data routes in `collections-and-data.md` for writes, and Socket.IO (`realtime.md`) for live updates. Do not build a client that expects GraphQL mutations to exist yet.

## 1. Endpoint

| | |
|---|---|
| Route | `POST /api/projects/:projectId/graphql` |
| Body | `{ query: string, variables?: object, operationName?: string }` |
| Response | Standard GraphQL `{ data, errors }` shape |
| Auth | Bearer token or API key, `rbacCheck("project", "read")`, plus project access and any app-role feature gate mapped on the project |
| Rate limit | 100 requests / 60s, a dedicated bucket separate from the general API limiter (see `error-handling-and-limits.md`) |

```typescript
// src/lib/graphql.ts
export async function runGraphQL<T>(
  baseUrl: string,
  projectId: string,
  bearerToken: string,
  query: string,
  variables?: Record<string, unknown>
): Promise<{ data: T | null; errors?: Array<{ message: string }> }> {
  const res = await fetch(`${baseUrl}/api/projects/${projectId}/graphql`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      Authorization: `Bearer ${bearerToken}`,
    },
    body: JSON.stringify({ query, variables }),
  })
  if (!res.ok && res.status !== 200) {
    throw new Error(`GraphQL request failed: ${res.status}`)
  }
  return res.json()
}
```

A malformed body (missing or non-string `query`) is a plain `400`, not a GraphQL-shaped error, check for that before parsing the response as `{ data, errors }`.

## 2. How the schema is derived

For a collection with slug `blog_posts`, the generator derives:

- An object type, PascalCase singular (e.g. `BlogPost`), one field per schema field, plus a field per relationship declared **from** that collection.
- A list query field, camelCase plural (`blogPosts`), args `filter` (a JSON-encoded string, parsed and sanitized through the **same** filter sanitizer REST's `?filter=` uses, so the `$regex`/`$text` blocklist in `collections-and-data.md` applies here too), `sort` (mirrors REST's `?sort=-createdAt` syntax), `limit`, and `offset`.
- A single-document query field, camelCase singular (`blogPost`), arg `id` (required).

Singularization is a narrow, English-only heuristic. If a collection slug singularizes to the same string as its plural form (already-singular slugs, some irregular plurals), the single-document field is suffixed `ById` instead (`blogById` rather than colliding with `blog`) to keep both fields addressable. Don't assume `<slug minus trailing s>` always matches the generated field name, check `GET /mcp/config`-style introspection or the console's GraphQL explorer if a collection's derived name is not obvious.

A relationship declared on a collection becomes a field on its GraphQL type resolving to the related type (or a list of it, for a one-to-many relationship), the same relationship metadata the REST `?populate=` param and the relationship explorer read, so a relationship visible there is queryable here too.

A brand-new project with zero collections still returns a valid schema, a single placeholder `_empty` field keeps introspection and tooling from breaking before the first collection exists.

## 3. Pagination and filtering

```graphql
query ListPublishedPosts($filter: String, $limit: Int, $offset: Int) {
  blogPosts(filter: $filter, sort: "-createdAt", limit: $limit, offset: $offset) {
    _id
    title
    status
    author {
      _id
      name
    }
  }
}
```

```typescript
const { data } = await runGraphQL<{ blogPosts: Array<{ _id: string; title: string; status: string; author: { _id: string; name: string } | null }> }>(
  baseUrl,
  projectId,
  bearerToken,
  LIST_PUBLISHED_POSTS_QUERY,
  {
    filter: JSON.stringify({ status: 'published' }),
    limit: 20,
    offset: 0,
  }
)
```

`limit` is clamped server-side the same way REST is, effectively capped at 100 per request regardless of what you pass, and `offset` floors at 0. Do not rely on requesting a larger page than that, page through with `offset` instead.

## 4. Hardening and limits (why a query can get rejected before it runs)

The endpoint is budgeted against abusive queries before execution, all four are hard, non-configurable ceilings on this route:

| Budget | Limit | What happens over it |
|---|---|---|
| Query depth | 8 nested selection levels | `GraphQLError`, "Query exceeds maximum depth of 8." |
| Alias / field count | 30 distinct aliases | `GraphQLError`, "Query exceeds maximum field/alias count of 30." |
| Parse token count | 10,000 tokens | Parse-time rejection before validation runs |
| Execution timeout | 10,000 ms | Execution aborted, partial results are not returned |

A real application query is typically 2 to 3 selection levels deep, so 8 leaves real headroom, if you are hitting the depth or alias ceiling, the query is either pathological or should be split into multiple requests rather than one deeply-nested one.

## 5. What this is not

- Not a way to write data. There is no `Mutation` type; `create`/`update`/`delete` still go through the REST routes in `collections-and-data.md`.
- Not a live subscription source. There is no `Subscription` type; for push-based updates use Socket.IO (`realtime.md`) or Server-Sent Events.
- Not a bypass of row-level scoping. The same role/permission-scoped filter REST queries build is applied here before your `filter` argument is layered on top, a caller cannot see documents their role could not already see over REST.

## See also

- `collections-and-data.md`, the REST data routes for writes, and the `$regex`/`$text` blocklist this endpoint's `filter` argument shares
- `realtime.md`, Socket.IO and SSE for the live-update capability GraphQL does not provide in this phase
- `error-handling-and-limits.md`, the general rate-limit bucket table this endpoint's 100/60s limiter sits alongside
