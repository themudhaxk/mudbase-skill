# Multi-Role System & API Keys

## 1. Multi-Role System

### 1.1 Role Selector on Signup

**The role config endpoint is not public.** `GET /api/projects/:projectId/multi-role` runs behind `authOrApiKey` + `validateProjectAccess` + `rbacCheck("project", "read")`, an anonymous browser on your signup page cannot call it. Calling `client.getMultiRoleConfig()` from an unauthenticated page returns `401`.

Two workable patterns:

1. **Bake the role list in at build time** (preferred for a signup page). The roles a project offers change on the order of months, not requests. Fetch them in a server component / build step using a server-side API key and render a static list.
2. **Proxy it** through your own backend route that holds the API key, if you need it live.

Below is pattern 1, with the fetch on the server and the selector as a client component.

```typescript
// src/app/auth/signup/page.tsx, server component, no 'use client'
import { RoleSelector } from '@/components/auth/RoleSelector'
import type { MultiRoleConfig, MultiRoleRole } from '@/lib/mudbase'

const MUDBASE_URL = process.env.MUDBASE_URL ?? 'https://cloud.mudbase.dev'
const PROJECT_ID = process.env.MUDBASE_PROJECT_ID
const API_KEY = process.env.MUDBASE_API_KEY // server-side only, never NEXT_PUBLIC_

async function loadSignupRoles(): Promise<MultiRoleRole[]> {
  if (!PROJECT_ID || !API_KEY) {
    throw new Error('MUDBASE_PROJECT_ID and MUDBASE_API_KEY must be set')
  }

  const res = await fetch(`${MUDBASE_URL}/api/projects/${PROJECT_ID}/multi-role`, {
    headers: { 'X-API-Key': API_KEY },
    // Roles change rarely; revalidate hourly rather than per-request.
    next: { revalidate: 3600 },
  })

  if (!res.ok) {
    throw new Error(`Failed to load multi-role config: ${res.status}`)
  }

  const body = (await res.json()) as { success: boolean; data: MultiRoleConfig }
  // Disabled roles exist in the config but must not be offered at signup.
  return body.data.roles.filter((role) => role.isEnabled)
}

export default async function SignupPage() {
  const roles = await loadSignupRoles()
  return <RoleSelector roles={roles} />
}
```

```typescript
// src/components/auth/RoleSelector.tsx
'use client'

import { useState } from 'react'
import type { ReactNode } from 'react'
import type { MultiRoleRole } from '@/lib/mudbase'

interface RoleSelectorProps {
  roles: MultiRoleRole[]
  /**
   * Renders the actual signup form for the chosen role, bring your own component here
   * rather than hardcoding one; the picker's only job is choosing the role.
   */
  renderForm?: (role: string, onChangeRole: (() => void) | undefined) => ReactNode
}

export function RoleSelector({ roles, renderForm }: RoleSelectorProps) {
  // A single-role project skips the picker entirely.
  const [selectedRole, setSelectedRole] = useState<string | null>(
    roles.length === 1 ? roles[0].slug : null
  )

  if (selectedRole !== null) {
    const onChangeRole = roles.length === 1 ? undefined : () => setSelectedRole(null)
    if (renderForm) {
      return <>{renderForm(selectedRole, onChangeRole)}</>
    }
    return (
      <p className="text-center text-sm text-gray-600">
        Selected role: <strong>{selectedRole}</strong>. Pass a <code>renderForm</code> prop to
        render your own signup form for this role.
      </p>
    )
  }

  return (
    <div className="flex min-h-screen items-center justify-center">
      <div className="w-full max-w-md space-y-4">
        <h1 className="text-2xl font-bold">Create an account</h1>
        <p className="text-gray-600">How will you use this platform?</p>
        <div className="grid gap-3">
          {roles.map((role) => (
            <button
              key={role.slug}
              type="button"
              onClick={() => setSelectedRole(role.slug)}
              className="rounded-lg border p-4 text-left transition-colors hover:border-blue-500 hover:bg-blue-50"
            >
              <p className="font-semibold">{role.name}</p>
              {role.description && (
                <p className="mt-1 text-sm text-gray-600">{role.description}</p>
              )}
              <div className="mt-2 flex flex-wrap gap-1">
                {role.requiresApproval && (
                  <span className="rounded bg-yellow-100 px-2 py-0.5 text-xs text-yellow-800">
                    Requires approval
                  </span>
                )}
                {role.requiresKYC && (
                  <span className="rounded bg-blue-100 px-2 py-0.5 text-xs text-blue-800">
                    Requires identity verification
                  </span>
                )}
                {role.requiresPayment && (
                  <span className="rounded bg-purple-100 px-2 py-0.5 text-xs text-purple-800">
                    Paid plan
                  </span>
                )}
              </div>
            </button>
          ))}
        </div>
      </div>
    </div>
  )
}
```

Roles are identified by `slug` (not `id`), and each carries a `signupEndpoint`, the role-scoped signup path the `register()` client method targets.

### 1.2 Role-Based UI Guards

```typescript
// src/components/RoleGuard.tsx
'use client'

import { useRouter } from 'next/navigation'
import { useEffect } from 'react'
import { useMudbase } from '@/lib/mudbase-provider'

interface RoleGuardProps {
  children: React.ReactNode
  allowedRoles: string[]
  fallback?: React.ReactNode
  redirectTo?: string
}

export function RoleGuard({ children, allowedRoles, fallback, redirectTo }: RoleGuardProps) {
  const { session, loading } = useMudbase()
  const router = useRouter()
  const userRole = session?.user?.role

  useEffect(() => {
    if (!loading && session && userRole && !allowedRoles.includes(userRole) && redirectTo) {
      router.replace(redirectTo)
    }
  }, [loading, session, userRole, allowedRoles, redirectTo, router])

  if (loading) return null
  if (!session) return null
  if (!userRole || !allowedRoles.includes(userRole)) return <>{fallback ?? null}</>
  return <>{children}</>
}

// Convenience wrappers
export function AdminOnly({ children }: { children: React.ReactNode }) {
  return <RoleGuard allowedRoles={['admin']}>{children}</RoleGuard>
}

export function StaffOnly({ children }: { children: React.ReactNode }) {
  return <RoleGuard allowedRoles={['admin', 'staff']}>{children}</RoleGuard>
}
```

```typescript
// src/hooks/useRole.ts
import { useMudbase } from '@/lib/mudbase-provider'

export function useRole() {
  const { session } = useMudbase()
  const role = session?.user?.role ?? null

  return {
    role,
    isAdmin: role === 'admin',
    isStaff: role === 'staff' || role === 'admin',
    hasRole: (r: string | string[]) => {
      if (!role) return false
      return Array.isArray(r) ? r.includes(role) : role === r
    },
  }
}
```

## 2. API Keys Management

API keys are for server-to-server communication. Never expose them to the browser.

### 2.1 The REST contract

The routes are **flat and org-scoped**, mounted at `/api/api-keys` (and aliased at `/api/tokens`). There is no `/api/projects/:id/api-keys`. The project is named in the request body on create, and as a path segment on the per-project list.

| Operation | Method + path |
|-----------|---------------|
| List (whole org) | `GET /api/api-keys` |
| List (one project) | `GET /api/api-keys/project/:projectId` |
| Create | `POST /api/api-keys`, body carries `projectId` |
| Update / revoke | `PATCH /api/api-keys/:id`, `{ isActive: false }` to revoke |
| Delete | `DELETE /api/api-keys/:id` |
| Usage stats | `GET /api/api-keys/:id/usage` |
| Rotate | `POST /api/api-keys/:id/regenerate` |

### 2.2 Permissions are structured, and required

`permissions` is **not** a string array. It is `{ resource, actions }[]`, validated against a fixed enum. An empty or missing array is rejected with `400`, mudbase deliberately has no "grant everything" default, so you must state the grant explicitly.

| Resources | Actions |
|-----------|---------|
| `auth`, `database`, `storage`, `functions`, `realtime`, `messaging`, `wallet`, `transactions`, `addons`, `kyc`, `payments` | `create`, `read`, `update`, `delete` |

Two further server-side rules:

- Only `owner` and `admin` may grant `delete` on any resource. A `developer` creating a key with any `delete` action gets `403`.
- The raw key is returned exactly once, on create (`apiKey.key`) and on rotate (`key`). Every later read exposes only `keyPrefix` / `keyPreview`. There is no recovery path, losing it means rotating.

Creation is also rate-limited (50/hour by default) and honours an `X-Idempotency-Key` header.

```typescript
// src/app/api/admin/api-keys/route.ts
// Server-side only, Next.js Route Handler.
import { NextRequest, NextResponse } from 'next/server'
import { z } from 'zod'

const MUDBASE_URL = process.env.MUDBASE_URL ?? 'https://cloud.mudbase.dev'
const MUDBASE_PROJECT_ID = process.env.MUDBASE_PROJECT_ID
const MUDBASE_ADMIN_TOKEN = process.env.MUDBASE_ADMIN_TOKEN // org owner/admin bearer token

const apiKeyResourceSchema = z.enum([
  'auth',
  'database',
  'storage',
  'functions',
  'realtime',
  'messaging',
  'wallet',
  'transactions',
  'addons',
  'kyc',
  'payments',
])

const apiKeyActionSchema = z.enum(['create', 'read', 'update', 'delete'])

const createKeySchema = z.object({
  name: z.string().min(1).max(120),
  permissions: z
    .array(
      z.object({
        resource: apiKeyResourceSchema,
        actions: z.array(apiKeyActionSchema).min(1),
      })
    )
    .min(1),
  expiresInDays: z.number().int().positive().max(3650).optional(),
})

interface CreateApiKeyResponse {
  message: string
  apiKey: {
    _id: string
    name: string
    keyPrefix: string
    key: string
  }
}

export async function POST(req: NextRequest): Promise<NextResponse> {
  if (!MUDBASE_PROJECT_ID || !MUDBASE_ADMIN_TOKEN) {
    return NextResponse.json({ error: 'mudbase credentials not configured' }, { status: 500 })
  }

  const parsed = createKeySchema.safeParse(await req.json())
  if (!parsed.success) {
    return NextResponse.json(
      { error: 'Invalid request', issues: parsed.error.issues },
      { status: 400 }
    )
  }

  const { name, permissions, expiresInDays } = parsed.data
  const expiresAt = expiresInDays
    ? new Date(Date.now() + expiresInDays * 24 * 60 * 60 * 1000).toISOString()
    : undefined

  const res = await fetch(`${MUDBASE_URL}/api/api-keys`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      Authorization: `Bearer ${MUDBASE_ADMIN_TOKEN}`,
      // Retrying a create without this can mint a second live key.
      'X-Idempotency-Key': crypto.randomUUID(),
    },
    body: JSON.stringify({
      name,
      projectId: MUDBASE_PROJECT_ID,
      permissions,
      expiresAt,
    }),
  })

  if (!res.ok) {
    const body = (await res.json().catch(() => ({}))) as { error?: string; limit?: number }
    return NextResponse.json(
      { error: body.error ?? 'Failed to create API key', limit: body.limit },
      { status: res.status }
    )
  }

  const body = (await res.json()) as CreateApiKeyResponse
  // `key` is present on this response and nowhere else, ever. Hand it to the operator
  // or write it to your secret store here, do not log it, do not persist it in your DB.
  return NextResponse.json({
    id: body.apiKey._id,
    key: body.apiKey.key,
    name: body.apiKey.name,
    keyPrefix: body.apiKey.keyPrefix,
  })
}
```

### 2.3 Least-privilege examples

```typescript
// src/lib/api-key-grants.ts
import type { ApiKeyPermission } from '@/lib/mudbase'

/** A read-only reporting job: pull data, touch nothing. */
export const REPORTING_GRANT: ApiKeyPermission[] = [
  { resource: 'database', actions: ['read'] },
  { resource: 'storage', actions: ['read'] },
]

/** A content ingestion worker: writes documents and files, never deletes. */
export const INGEST_GRANT: ApiKeyPermission[] = [
  { resource: 'database', actions: ['create', 'read', 'update'] },
  { resource: 'storage', actions: ['create', 'read'] },
]

/** An MCP key for an AI coding agent, scope to what it is actually building against. */
export const AGENT_GRANT: ApiKeyPermission[] = [
  { resource: 'database', actions: ['create', 'read', 'update'] },
  { resource: 'functions', actions: ['create', 'read', 'update'] },
  { resource: 'storage', actions: ['read'] },
]
```

The MCP server (see `mcp-and-ai-agents.md`) authenticates with exactly these keys and checks each write tool against the grant, so a narrow key is a real containment boundary for an agent, not just a label.

**When to use API keys vs JWTs:**

| API Key | JWT (User Token) |
|---------|-----------------|
| Server-to-server calls | User-facing client requests |
| CI/CD pipelines, cron jobs | Browser, mobile app sessions |
| Stored in env vars / secrets manager | Stored in memory / httpOnly cookie |
| Long-lived (months/years) | Short-lived (hours/days) |
| Scoped to service permissions | Scoped to user's own data |

## See also

- `auth.md`, session/JWT handling for the user-facing side of this table
- `mcp-and-ai-agents.md`, how an AI agent authenticates with a scoped API key
