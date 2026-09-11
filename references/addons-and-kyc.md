# Add-ons Marketplace & Identity Verification (KYC/KYB)

## 1. Add-ons Marketplace

Add-ons are small metered utilities the platform runs for you: QR/PDF/CSV/ICS/vCard generation, GeoIP, currency and crypto price lookup, UUID/hash/slug/password generation, Markdown rendering. Each successful invocation is billed per call against the org's in-app credit balance (see `wallets-and-payments.md`, section 3).

| Operation | Method + path | Auth |
|-----------|---------------|------|
| Catalog | `GET /api/addons` | bearer or API key, `addon:read` |
| Invoke | `POST /api/projects/:projectId/addons/:addon/invoke` | bearer or API key, `addon:create`, project ownership |
| Job status | `GET /api/projects/:projectId/addons/jobs/:id` | bearer or API key, `addon:read` |

Invoke returns **`200` when the job finished inline** and **`202` when it is still processing**, both with the same `{ job }` envelope. Write the client against `job.status`, not the HTTP status, and poll when it is not yet terminal. Pass `X-Idempotency-Key` (or `idempotencyKey` in the body) so a retried invocation reuses the existing job rather than billing twice.

```typescript
// src/lib/mudbase.ts, inside class MudbaseClient
  async listAddons(): Promise<AddonCatalogEntry[]> {
    const res = await this.request<{ addons: AddonCatalogEntry[] }>('GET', '/api/addons')
    return res.addons
  }

  async invokeAddon(
    addon: string,
    input: Record<string, unknown>,
    idempotencyKey?: string
  ): Promise<AddonJob> {
    const res = await this.request<{ job: AddonJob }>(
      'POST',
      `/api/projects/${this.projectId}/addons/${addon}/invoke`,
      { input, idempotencyKey }
    )
    return res.job
  }

  async getAddonJob(jobId: string): Promise<AddonJob> {
    const res = await this.request<{ job: AddonJob }>(
      'GET',
      `/api/projects/${this.projectId}/addons/jobs/${jobId}`
    )
    return res.job
  }
```

```typescript
// src/lib/mudbase.ts, types
export interface AddonCatalogEntry {
  key: string
  name: string
  description: string
  billingResource: string
  resultMode: 'file' | 'inline' | 'data'
  priceUsd: number
}

export type AddonJobStatus = 'queued' | 'processing' | 'completed' | 'failed'

export interface AddonJobResult {
  mode: 'file' | 'inline' | 'data' | null
  fileId: string | null
  url: string | null
  mimeType: string | null
  filename: string | null
  /** base64, only for small binary results returned inline. */
  inline: string | null
  data: unknown
  expiresAt: string | null
}

export interface AddonJob {
  _id: string
  addon: string
  status: AddonJobStatus
  result: AddonJobResult
  error: string | null
  /** Charged on success; refunded automatically if the job ends up failing. */
  billedCents: number
  startedAt: string | null
  completedAt: string | null
  createdAt: string
}
```

```typescript
// src/hooks/useAddon.ts
import { useMutation, useQuery } from '@tanstack/react-query'
import { useMudbase } from '@/lib/mudbase-provider'
import type { AddonCatalogEntry, AddonJob } from '@/lib/mudbase'

export function useAddonCatalog() {
  const { client } = useMudbase()
  return useQuery<AddonCatalogEntry[]>({
    queryKey: ['addons', 'catalog'],
    queryFn: () => client.listAddons(),
    // The catalog changes on deploys, not on user actions.
    staleTime: 1000 * 60 * 60,
  })
}

export function useInvokeAddon(addon: string) {
  const { client } = useMudbase()
  return useMutation<AddonJob, Error, Record<string, unknown>>({
    mutationFn: (input) => client.invokeAddon(addon, input, crypto.randomUUID()),
  })
}

/** Polls a pending job to a terminal state, then stops. Skip entirely if invoke already returned `completed`. */
export function useAddonJob(jobId: string | null, pollIntervalMs = 1_500) {
  const { client } = useMudbase()
  return useQuery<AddonJob>({
    queryKey: ['addons', 'job', jobId],
    queryFn: () => {
      if (jobId === null) throw new Error('useAddonJob called without a jobId')
      return client.getAddonJob(jobId)
    },
    enabled: jobId !== null,
    refetchInterval: (query) => {
      const status = query.state.data?.status
      return status === 'completed' || status === 'failed' ? false : pollIntervalMs
    },
    staleTime: 0,
  })
}
```

A `file`-mode result carries a `url` with an `expiresAt`, download or re-host it before it lapses rather than storing the URL as a permanent reference.

## 2. Identity Verification (KYC/KYB)

mudbase gates financial-egress actions behind organization-level identity verification. Two distinct things share the name:

- **Platform KYC**, verifying *your* organization, so mudbase will let you move money. This is what unblocks the gated actions below.
- **White-label KYC/KYB**, verifying *your customers*, on your behalf, scoped to one of your projects. You resell verification to your own end-users.

The whole subsystem is opt-in at the deployment level: where identity verification is not configured for a deployment, the gate is a no-op and nothing is blocked.

### What is gated

The gate is evaluated **live on every call**, not once at an unlock step, so an organization whose verification is later revoked or flagged loses access immediately rather than coasting on a stale approval. It sits in front of:

| Gated action | Route |
|--------------|-------|
| Custodial withdrawal | `POST /api/wallet/:walletId/withdraw` |
| Exporting a custodial wallet's private key | `GET /api/wallet/:walletId/private-key` |
| Non-custodial broadcast | `POST /api/wallet/non-custodial/broadcast` |
| Account-abstraction broadcast | `POST /api/wallet/non-custodial/account-abstraction/broadcast` |
| Raw transaction relay | `POST /api/tx/broadcast` |
| Creating a payment link | `POST /api/orgs/:orgId/payment-links` |
| Stablecoin on-ramp / off-ramp, manual payout | billing routes |

A blocked call returns `403` with `{ error, code: "KYC_REQUIRED" }`. Branch on the `code`, not the message.

### Platform KYC flow

| Operation | Method + path | Auth |
|-----------|---------------|------|
| Start a session | `POST /api/kyc/sessions` | bearer, owner/admin only, auth rate limiter |
| Read status | `GET /api/kyc/status` | bearer |
| Read one verification record | `GET /api/kyc/verifications/:id` | bearer |

Start returns `201 { sessionId, url, status }`. Redirect the user to `url`, the verification itself happens in a hosted flow, not in your UI. The org's status flips to `pending` immediately; it becomes `approved` (or not) asynchronously, after the user finishes, which can take minutes. There is no synchronous completion signal, so the client must poll `GET /api/kyc/status` (or wait for the user to return and refresh).

```typescript
// src/lib/mudbase.ts, inside class MudbaseClient
  async startKycSession(language?: string): Promise<KycSession> {
    return this.request<KycSession>('POST', '/api/kyc/sessions', { language })
  }

  async getKycStatus(): Promise<KycStatus> {
    return this.request<KycStatus>('GET', '/api/kyc/status')
  }
```

```typescript
// src/lib/mudbase.ts, types
export interface KycSession {
  sessionId: string
  /** Hosted verification URL, redirect the user here. */
  url: string
  status: string
}

export interface KycStatus {
  status: 'none' | 'pending' | 'approved' | 'declined' | 'in_review'
  verifiedAt: string | null
}
```

```typescript
// src/hooks/useKycStatus.ts
import { useQuery } from '@tanstack/react-query'
import { useMudbase } from '@/lib/mudbase-provider'
import type { KycStatus } from '@/lib/mudbase'

/**
 * Verification completes asynchronously, so there is nothing to await,
 * poll while pending, stop once the outcome is settled.
 */
export function useKycStatus(pollWhilePendingMs = 5_000) {
  const { client } = useMudbase()
  return useQuery<KycStatus>({
    queryKey: ['kyc', 'status'],
    queryFn: () => client.getKycStatus(),
    refetchInterval: (query) =>
      query.state.data?.status === 'pending' || query.state.data?.status === 'in_review'
        ? pollWhilePendingMs
        : false,
    refetchOnWindowFocus: true,
  })
}
```

`refetchOnWindowFocus` matters here: the user leaves your tab for the hosted flow and comes back, and that return is your best signal to re-check.

### White-label verification

Project-scoped, for verifying your own end-users:

| Operation | Method + path |
|-----------|---------------|
| Start end-user KYC | `POST /api/projects/:projectId/kyc/sessions` |
| Start business KYB | `POST /api/projects/:projectId/kyb/sessions` |
| Check reusable identity | `POST /api/projects/:projectId/kyc/check-reuse` |

Start accepts `{ workflowId, vendorData, callback, language, reuseIdentifier }`. When `reuseIdentifier` matches an already-approved identity, mudbase satisfies the new verification from the existing one and returns `200` with `reused: true` instead of `201`, so the user is not asked to verify twice, and you are not billed twice. Results are delivered to the webhook destination you configure at `POST /api/kyc/webhook-config`, signed the same way as the outbound webhooks in `integrations-and-webhooks.md`.

## See also

- `wallets-and-payments.md`, the in-app credit balance add-ons bill against
- `integrations-and-webhooks.md`, the signature scheme white-label KYC results are delivered with
