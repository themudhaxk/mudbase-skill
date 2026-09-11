# Payments & Credit Balance

mudbase does not offer its own custodial or non-custodial crypto wallets. That whole stack (wallet creation, real-time wallet/tx socket events, and transaction broadcasting) was archived out of mudbase on 2026-08-23 and now lives in MudChain, a separate product with its own API - it is not part of mudbase's capability surface and this SDK has no client for it. If your app needs to create wallets, track balances, or broadcast signed transactions, integrate with MudChain directly, not mudbase.

What mudbase itself still provides is a merchant payment-processing surface (card/local-rail payments and stablecoin payment links) and an in-app prepaid credit balance for billing. These are two related but distinct systems, covered in order below.

## 1. Payment Links

A payment link is a hosted, tokenized checkout for a stablecoin payment. You create it org-side; your customer opens it in a browser with no account and no auth.

| Operation | Method + path | Auth |
|-----------|---------------|------|
| Create | `POST /api/orgs/:orgId/payment-links` | bearer, owner/admin, **KYC-approved** |
| List | `GET /api/orgs/:orgId/payment-links` | bearer, `payment:read` |
| Cancel | `POST /api/orgs/:orgId/payment-links/:linkId/cancel` | bearer, owner/admin |
| Public checkout read | `GET /api/payment-links/:token` | none |

Create takes `{ amount, currency, network, description, redirectUrl, expiresInHours }`, `currency` and `network` are required; an open-amount link is created by omitting `amount`. It returns `201 { link }`.

KYC is re-checked live at creation time, not at the point the org first enabled stablecoin payments, because a link points a stranger at real money. Handle `403 KYC_REQUIRED` (see `addons-and-kyc.md`) on this endpoint specifically.

The public read returns a deliberately narrowed DTO: `token`, `amount`, `currency`, `network`, `address`, `description`, `redirectUrl`, `status`, `expiresAt`. No org id, no internal `_id`. Your checkout page should render from exactly those fields.

```typescript
// src/lib/mudbase.ts, inside class MudbaseClient
  async createPaymentLink(
    orgId: string,
    params: {
      currency: string
      network: string
      amount?: string
      description?: string
      redirectUrl?: string
      expiresInHours?: number
    }
  ): Promise<PaymentLink> {
    const res = await this.request<{ link: PaymentLink }>(
      'POST',
      `/api/orgs/${orgId}/payment-links`,
      params
    )
    return res.link
  }

  async listPaymentLinks(orgId: string, limit?: number): Promise<PaymentLink[]> {
    const qs = limit !== undefined ? `?limit=${limit}` : ''
    const res = await this.request<{ links: PaymentLink[] }>(
      'GET',
      `/api/orgs/${orgId}/payment-links${qs}`
    )
    return res.links
  }

  async cancelPaymentLink(orgId: string, linkId: string): Promise<PaymentLink> {
    const res = await this.request<{ link: PaymentLink }>(
      'POST',
      `/api/orgs/${orgId}/payment-links/${linkId}/cancel`
    )
    return res.link
  }
```

```typescript
// src/lib/mudbase.ts, types
export type PaymentLinkStatus = 'pending' | 'paid' | 'expired' | 'cancelled'

export interface PaymentLink {
  _id: string
  token: string
  amount: string | null
  currency: string
  network: string
  address: string
  description: string | null
  redirectUrl: string | null
  status: PaymentLinkStatus
  paidTxHash: string | null
  paidAmount: string | null
  paidAt: string | null
  expiresAt: string | null
  createdAt: string
}

/** Fields the unauthenticated checkout endpoint returns, no org id, no internal _id. */
export type PublicPaymentLink = Pick<
  PaymentLink,
  'token' | 'amount' | 'currency' | 'network' | 'address' | 'description' | 'redirectUrl' | 'status' | 'expiresAt'
>
```

```typescript
// src/lib/payment-link-url.ts
import type { PublicPaymentLink } from '@/lib/mudbase'

/**
 * The checkout URL you send to a customer. `/api/payment-links/:token` is the JSON
 * read your own hosted checkout page calls; the page itself is yours to build.
 */
export function paymentLinkCheckoutUrl(appOrigin: string, token: string): string {
  return `${appOrigin}/checkout/${token}`
}

export async function fetchPublicPaymentLink(
  baseUrl: string,
  token: string
): Promise<PublicPaymentLink> {
  const res = await fetch(`${baseUrl}/api/payment-links/${token}`)
  if (!res.ok) {
    const body = (await res.json().catch(() => ({}))) as { error?: string }
    throw new Error(body.error ?? `Payment link not available (${res.status})`)
  }
  const body = (await res.json()) as { link: PublicPaymentLink }
  return body.link
}
```

Payment confirmation is detected asynchronously by an incoming-payment webhook, not synchronously at checkout, so a checkout page should poll the public read rather than assuming the payment is settled the moment the customer says they sent it.

## 2. Merchant Payment Processing (fees)

Separate from payment links: mudbase also processes card and local-rail merchant payments (the checkout your project's customers pay through, not crypto).

| Operation | Method + path | Auth |
|-----------|---------------|------|
| List supported payout countries | `GET /api/orgs/:orgId/payment-processing/countries` | bearer or API key, `payment:read` |
| Get the bank list for a country | `GET /api/orgs/:orgId/payment-processing/banks?country=NG` | bearer or API key, `payment:read` |
| Submit payout onboarding | `POST /api/orgs/:orgId/payment-processing/enable` | bearer, owner/admin, KYC-checked |
| Onboarding status | `GET /api/orgs/:orgId/payment-processing/status` | bearer or API key, `payment:read` |
| Create a payment | `POST /api/orgs/:orgId/payment-processing/initialize-payment` | bearer or API key, `payment:create` |
| Preview the fee | `GET /api/orgs/:orgId/payment-processing/fee-breakdown` | bearer or API key, `payment:read` |
| List payment records | `GET /api/orgs/:orgId/payment-processing/records` | bearer or API key, `payment:read` |

The supported payout-country set is dynamic, do not hardcode a country list in your integration. Call `GET .../countries` and render whatever it returns: each entry carries `code`, `name`, `currency`, `banksListSupported`, `mobileMoney`, `international`, and its own `fields` schema (labels, patterns, whether a bank dropdown or a free-text code applies), so the same onboarding form works across countries without special-casing each one. NG is currently the only market that additionally requires a BVN; US and UK collect routing-number/sort-code style fields plus an account holder name instead of a bank list.

Submitting `/enable` does not activate payments immediately: it validates and stores the payout details, then queues the org for platform-admin review. The response carries `{ onboarded, alreadyEnabled, approvalStatus }`, not a raw payout-account id. `/initialize-payment` only starts succeeding once the org is approved, so poll `/status` (or read `approvalStatus` on a resubmit) rather than assuming payouts are live the moment `/enable` returns `200`.

`/initialize-payment` returns `{ link, txRef, providerRef, amount, currency, orgReceives, fee, feeRate }`, a single all-in fee and the effective rate, never a processor-side split. `/fee-breakdown` returns the same shape without creating a payment, and accepts optional `country`, `method` (`card` | `momo` | `bank_transfer` | `ussd` | `eft`), and `international` query params so you can preview the fee for a specific rail before charging it.

The fee mudbase charges the merchant is **cost-plus**: the underlying rail's real cost for that country/method, passed through, plus a mudbase margin, never a flat marked-up rate presented as if it were the rail's own price.

Margin tiers (all amounts USD):

| Tier | Rate | Fixed |
|------|------|-------|
| Local African rails | 1.0% | $0.20 |
| East/Mid Africa card (KE, UG, TZ, RW, ZM) | 0.9% | $0.20 |
| International (US, UK, cross-border cards) | 1.0% | $0.30 |

Every computed fee is floored at **$0.30** so a very small transaction never charges an unrealistically thin fee. This cost-plus model applies to payment processing specifically; it is distinct from the legacy flat 7% + $0.50 rate, which still exists in the codebase but is now scoped only to the platform's own revenue split on **subscription** billing, not to merchant payment processing.

Never surface which processor mudbase routes a given country/method through in product copy or API responses, present the capability as first-party ("card and local-rail payments", not "payments via [processor]").

## 3. In-App Credit Balance

Distinct from the org's subscription billing. Credit is a **prepaid, spend-only** balance: you top it up, and it is drawn down automatically against overage invoices, capacity purchases and per-call add-on invocations (see `addons-and-kyc.md`). It is deliberately not a wallet, there is no withdrawal, no transfer, and no refund path.

| Operation | Method + path | Auth |
|-----------|---------------|------|
| Balance | `GET /api/billing/credit/balance` | bearer |
| Ledger history | `GET /api/billing/credit/transactions` | bearer |
| Start a top-up | `POST /api/billing/credit/topup` | bearer |
| Verify a top-up | `POST /api/billing/credit/verify-payment?tx_ref=...` | bearer, org owner |
| Set low-balance threshold | `PUT /api/billing/credit/low-balance-threshold` | bearer, owner/admin |

All amounts are **integer cents**. Minimum top-up is 1000 ($10); the console pre-fills that amount. The low-balance alert threshold defaults to 200 ($2), is org-configurable, `0` disables it, and the ceiling is 100000 ($1,000), above that the warning could never fire usefully.

Top-up returns a hosted checkout `link`; redirect the user there, then verify on return with the `tx_ref` you were given. Verification is idempotent, a duplicate call reports success without double-crediting.

```typescript
// src/lib/mudbase.ts, inside class MudbaseClient
  async getCreditBalance(): Promise<CreditBalance> {
    const res = await this.request<{ success: boolean; data: CreditBalance }>(
      'GET',
      '/api/billing/credit/balance'
    )
    return res.data
  }

  async topUpCredit(params: {
    amountCents: number
    redirectUrl?: string
    email?: string
    name?: string
  }): Promise<CreditTopup> {
    if (!Number.isInteger(params.amountCents) || params.amountCents < 1000) {
      throw new MudbaseError('Top-up must be a whole number of cents, minimum 1000 ($10)', 400)
    }
    const res = await this.request<{ success: boolean; data: CreditTopup }>(
      'POST',
      '/api/billing/credit/topup',
      params
    )
    return res.data
  }

  async verifyCreditTopup(txRef: string): Promise<{ success: boolean }> {
    return this.request<{ success: boolean }>(
      'POST',
      `/api/billing/credit/verify-payment?tx_ref=${encodeURIComponent(txRef)}`
    )
  }

  async setLowBalanceThreshold(thresholdCents: number): Promise<{ lowBalanceThresholdCents: number }> {
    const res = await this.request<{ success: boolean; data: { lowBalanceThresholdCents: number } }>(
      'PUT',
      '/api/billing/credit/low-balance-threshold',
      { thresholdCents }
    )
    return res.data
  }
```

```typescript
// src/lib/mudbase.ts, types
export interface CreditBalance {
  balanceCents: number
  currency: string
  lowBalanceThresholdCents: number
}

export interface CreditTopup {
  /** Hosted checkout URL, redirect the user here. */
  link: string
  /** Pass back to verify-payment on return. */
  txRef: string
  amountCents: number
  currency: string
}
```

```typescript
// src/hooks/useCreditBalance.ts
import { useMutation, useQuery, useQueryClient } from '@tanstack/react-query'
import { useMudbase } from '@/lib/mudbase-provider'
import type { CreditBalance, CreditTopup } from '@/lib/mudbase'

export function useCreditBalance() {
  const { client } = useMudbase()
  return useQuery<CreditBalance>({
    queryKey: ['billing', 'credit', 'balance'],
    queryFn: () => client.getCreditBalance(),
    // Add-on invocations debit in real time, so keep this fresh on any billing screen.
    staleTime: 1000 * 15,
  })
}

export function useTopUpCredit() {
  const { client } = useMudbase()
  return useMutation<CreditTopup, Error, { amountCents: number; redirectUrl?: string }>({
    mutationFn: (params) => client.topUpCredit(params),
  })
}

/** Call on return from checkout with the tx_ref from the query string. */
export function useVerifyCreditTopup() {
  const { client } = useMudbase()
  const queryClient = useQueryClient()
  return useMutation<{ success: boolean }, Error, string>({
    mutationFn: (txRef) => client.verifyCreditTopup(txRef),
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ['billing', 'credit'] })
    },
  })
}
```

Do not render the balance as a spendable wallet or offer a "withdraw" affordance, there is no such endpoint, and presenting it that way misrepresents what the customer bought.

## See also

- `addons-and-kyc.md`, the KYC gate that fronts payment link creation, and add-on invocation billing against credit
