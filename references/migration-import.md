# Project Sharing, Cloning, Forking & Vendor Migration Import

Everything in this file is **management-plane, not data-plane**. These are console/admin operations, an app built *on* a mudbase project never calls them. Reach for this file only when you are building tooling against mudbase's own management API (an internal admin surface, a template gallery, a provisioning or import wizard).

## 1. Project Sharing, Cloning & Forking

Everything here requires a bearer token with `owner` or `admin` on the org that owns the project.

| Operation | Method + path | Notes |
|-----------|---------------|-------|
| Publish / unpublish | `PATCH /api/projects/:id/visibility` | `{ visibility: 'private' \| 'public', communityMeta }` |
| Share with a person | `POST /api/projects/:id/share` | `{ email, permission }`, default permission `clone` |
| List shares | `GET /api/projects/:id/shares` | |
| Revoke a share | `DELETE /api/projects/:id/shares/:shareId` | |
| Accept a share invite | `POST /api/projects/share/:token/accept` | invitee's own token |
| Browse public projects | `GET /api/projects/community` | unauthenticated |
| Preview a public project | `GET /api/projects/community/:id/preview` | unauthenticated |
| Fork a public/shared project | `POST /api/projects/community/:id/fork` | `{ targetOrgId, name, slug, includeData, includeFiles }` |
| Rate a public project | `POST` / `DELETE /api/projects/community/:id/rate` | |
| Duplicate your own project | `POST /api/projects/:id/clone` | same body as fork |
| Clone job status | `GET /api/projects/clone-jobs/:jobId` | |
| Accept an ownership transfer | `POST /api/projects/transfer/:token/accept` | |

Three behaviors worth knowing before you build UI around this:

- **Publishing requires metadata.** A project cannot go `public` without a `communityMeta.title` and `communityMeta.summary`. Collect them in the same form as the visibility toggle, or the request `400`s.
- **Publishing production data requires a second, explicit confirmation.** If `communityMeta.dataProfile` is `production`, the request is rejected with `code: "PRODUCTION_DATA_PUBLIC_CONFIRMATION_REQUIRED"` unless you also send `confirmProductionDataPublic: true`. Surface that as a distinct, deliberately awkward confirmation step, it exists because a mislabelled dataset becomes copyable by anyone the instant it is published.
- **Clone and fork are partly asynchronous.** Without `includeFiles`, the response is `201` and the project already exists. With `includeFiles`, file copying is queued: the response carries a `jobId` and `status`, and you poll `GET /api/projects/clone-jobs/:jobId`. The message text differs too (`"Clone started"` vs `"Project cloned successfully"`), branch on the presence of `jobId`, not on the message.
- **Forking versus cloning** differ in the access check, not the mechanics: `/clone` requires org access to the *source* project, `/community/:id/fork` requires only that the source is public or shared with you. Both require membership of the target org.

## 2. Vendor Migration Importer

mudbase can pull an existing project in from another vendor, or from a raw database, so a customer does not have to hand-write a migration. You would call this from a console/import wizard.

| Operation | Method + path | Auth |
|-----------|---------------|------|
| Test credentials | `POST /api/migration-import/test-connection` | bearer |
| Discover what's importable | `POST /api/migration-import/discover` | bearer |
| Start the import | `POST /api/migration-import/start/:projectId` | bearer, owner/admin, project access |
| Job status | `GET /api/migration-import/:jobId` | bearer |

`test-connection` and `discover` both take `{ vendor, credentials }`; `start` additionally takes `{ selection, estimatedBytes }` and returns `201 { message, jobId, status }`. The import itself runs on a queue, so `jobId` is the only handle you get, poll it.

The credentials are the customer's own vendor keys. They are used for the duration of the request (test/discover) or held encrypted at rest on the job (start), and never logged. Build the wizard so those credentials are entered once, submitted directly, and not retained in your own client state or storage after the job is created.

### Supported sources

| `vendor` value | What it connects to |
|---|---|
| `postgres` | Any reachable Postgres database, direct SQL connection |
| `mysql` | Any reachable MySQL database, direct SQL connection |
| `supabase` | A Supabase project (Postgres underneath, connected through Supabase's own API surface) |
| `mongodb` | Any reachable MongoDB deployment |
| `firebase` | A Firebase project (Firestore) |
| `appwrite` | An Appwrite project, including self-hosted instances on a custom domain |
| `clerk` | A Clerk project (user/auth data) |

### Relationship (foreign key) import: real for SQL and Supabase sources only

This is a common point of overclaiming, so be precise: automatic conversion of a source foreign key into a real, queryable mudbase **relationship field** (the kind the relationship explorer and the fluent query builder's `?populate=` traverse) is implemented for `postgres`, `mysql`, and `supabase` today. Those three connectors read the source's real FK/constraint catalog, so the importer knows which column points at which table before a single row is copied.

For those three sources:

- Each detected FK column is marked as a mudbase `reference` field pointing at the target collection, not flattened into a plain string or number.
- Row import remaps the source database's raw FK scalar into the corresponding mudbase document `_id` as data is copied in, using an internal id map built while importing.
- After every table in the job has finished, the importer creates the actual `Relationship` documents (the same records the relationship explorer and populate API read) for each detected FK, skipping only where the target table was not part of this import job (visible as a `relationship_target_not_imported` warning on the job) or where a relationship already exists.
- Default `onDelete` policy on an auto-created relationship is conservative (restrict-style), adjust it by hand afterward via the relationship API or UI if a different policy is needed.

For every other source, `mongodb`, `firebase`, `clerk`, and `appwrite`, relationship import does **not** currently create mudbase relationship fields. What each of those actually does with a reference-shaped value:

- `mongodb` is schemaless; there is no constraint catalog to read, so referenced ObjectIds come across as plain values, not typed relationships.
- `firebase` preserves a Firestore `DocumentReference` as its path string, so the pointer is human-relinkable after import, but it is not converted into a mudbase relationship field.
- `clerk` and `appwrite` import user/schema data without FK detection.

Do not tell a user migrating from Appwrite (or Mongo, Firebase, or Clerk) that their foreign keys become live mudbase relationships automatically, they do not yet. If your wizard copy says "relationships are preserved," scope that sentence to the Postgres/MySQL/Supabase path, or say plainly that other sources import as flat data and relationships can be declared by hand afterward.

```typescript
// src/lib/migration-import.ts
export type MigrationJobStatus = 'queued' | 'running' | 'completed' | 'failed'

export interface MigrationImportJob {
  _id: string
  vendor: string
  status: MigrationJobStatus
  progress?: number
  error?: string | null
}

export async function startMigrationImport(
  baseUrl: string,
  bearerToken: string,
  projectId: string,
  body: {
    vendor: string
    credentials: Record<string, string>
    selection: Record<string, unknown>
    estimatedBytes?: number
  }
): Promise<{ jobId: string; status: MigrationJobStatus }> {
  const res = await fetch(`${baseUrl}/api/migration-import/start/${projectId}`, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      Authorization: `Bearer ${bearerToken}`,
    },
    body: JSON.stringify(body),
  })

  const parsed = (await res.json()) as {
    jobId?: string
    status?: MigrationJobStatus
    error?: string
  }
  if (!res.ok || !parsed.jobId || !parsed.status) {
    throw new Error(parsed.error ?? `Import failed to start (${res.status})`)
  }
  return { jobId: parsed.jobId, status: parsed.status }
}
```

## See also

- `collections-and-data.md`, the reference field type and `?populate=` query pattern that a successfully-imported relationship becomes queryable through
- `access-control.md`, the owner/admin role checks these management-plane routes require
