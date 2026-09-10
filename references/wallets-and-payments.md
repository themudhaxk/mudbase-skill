# Wallets, Blockchain & Payments

mudbase supports multi-chain crypto wallets as a first-class feature, plus a separate merchant payment-processing surface (card/local-rail payments) and an in-app prepaid credit balance for billing. These are three related but distinct systems, covered in order below.

## 1. Wallet Real-Time Events

Max 50 wallet rooms per socket. Uses the same `getMudbaseSocket()` client from `realtime.md`.

### Subscribe

```typescript
const socket = getMudbaseSocket()

// By project (all wallets in this project)
socket.emit('wallet:subscribe:project', { projectId: 'YOUR_PROJECT_ID' }, (ack: unknown) => {
  console.log('wallet subscribed:', ack)
  // { projectId: '...', room: 'wallet:project:...' }
})

// By specific wallet address
socket.emit('wallet:subscribe:address', { addressId: 'WALLET_ADDRESS_ID' })

// By chain (ethereum, bitcoin, solana, etc., lowercase)
socket.emit('wallet:subscribe:chain', { chain: 'ethereum' })

// By currency symbol
socket.emit('wallet:subscribe:currency', { currency: 'USDT' })

// Unsubscribe
socket.emit('wallet:unsubscribe:address', { addressId: 'WALLET_ADDRESS_ID' })
socket.emit('wallet:unsubscribe:project', { projectId: 'YOUR_PROJECT_ID' })
```

### Server events

```typescript
interface WalletBalanceEvent {
  addressId: string
  address: string // on-chain address, e.g. "0xabc..."
  chain: string // "ethereum", "bitcoin", etc.
  project: string
  org: string
  previousBalance: string // decimal string
  newBalance: string
  balance: string
  timestamp: string
}

interface WalletTxEvent {
  addressId: string
  address: string
  chain: string
  txHash: string
  amount: string
  currency: string
  direction: 'incoming' | 'outgoing'
  project: string
  timestamp: string
}

socket.on<WalletBalanceEvent>('wallet:balance:updated', (ev) => {
  updateWalletBalance(ev.addressId, ev.newBalance, ev.chain)
})

socket.on<WalletTxEvent>('wallet:tx:detected', (ev) => {
  addPendingTransaction(ev)
})

socket.on<WalletTxEvent>('wallet:tx:broadcast', (ev) => {
  updateTxStatus(ev.txHash, 'broadcast')
})

socket.on<WalletTxEvent>('wallet:tx:confirmed', (ev) => {
  updateTxStatus(ev.txHash, 'confirmed')
})

socket.on<WalletTxEvent>('wallet:tx:failed', (ev) => {
  updateTxStatus(ev.txHash, 'failed')
})
```

Events may be emitted to multiple rooms simultaneously (address, project, org, chain, currency rooms).

### `useWallet` hook

```typescript
// src/hooks/useWallet.ts
'use client'

import { useState, useEffect } from 'react'
import { getMudbaseSocket } from '@/lib/mudbase-socket'

type TxStatus = 'detected' | 'broadcast' | 'confirmed' | 'failed'

interface WalletTransaction {
  txHash: string
  amount: string
  currency: string
  direction: 'incoming' | 'outgoing'
  status: TxStatus
  chain: string
  timestamp: string
}

interface WalletState {
  balances: Record<string, string> // chain -> balance
  transactions: WalletTransaction[]
}

export function useWalletAddressEvents(addressId: string) {
  const socket = getMudbaseSocket()
  const [state, setState] = useState<WalletState>({ balances: {}, transactions: [] })

  useEffect(() => {
    if (!socket.connected || !addressId) return

    socket.emit('wallet:subscribe:address', { addressId })

    const offBalance = socket.on<{ chain: string; newBalance: string }>('wallet:balance:updated', (ev) => {
      setState((prev) => ({
        ...prev,
        balances: { ...prev.balances, [ev.chain]: ev.newBalance },
      }))
    })

    function handleTx(status: TxStatus) {
      return socket.on<WalletTransaction & { txHash: string }>(`wallet:tx:${status}`, (ev) => {
        setState((prev) => {
          const idx = prev.transactions.findIndex((t) => t.txHash === ev.txHash)
          if (idx >= 0) {
            const updated = [...prev.transactions]
            updated[idx] = { ...updated[idx], status }
            return { ...prev, transactions: updated }
          }
          return {
            ...prev,
            transactions: [{ ...ev, status }, ...prev.transactions],
          }
        })
      })
    }

    const offDetected  = handleTx('detected')
    const offBroadcast = handleTx('broadcast')
    const offConfirmed = handleTx('confirmed')
    const offFailed    = handleTx('failed')

    return () => {
      socket.emit('wallet:unsubscribe:address', { addressId })
      offBalance(); offDetected(); offBroadcast(); offConfirmed(); offFailed()
    }
  }, [addressId, socket.connected])

  return state
}

export function useProjectWalletEvents(projectId: string) {
  const socket = getMudbaseSocket()
  const [events, setEvents] = useState<Array<{ type: string; data: unknown }>>([])

  useEffect(() => {
    if (!socket.connected || !projectId) return

    socket.emit('wallet:subscribe:project', { projectId })

    const evTypes = ['wallet:balance:updated', 'wallet:tx:detected', 'wallet:tx:confirmed', 'wallet:tx:failed'] as const
    const offs = evTypes.map((type) =>
      socket.on(type, (data) => setEvents((prev) => [{ type, data }, ...prev].slice(0, 100)))
    )

    return () => {
      socket.emit('wallet:unsubscribe:project', { projectId })
      offs.forEach((off) => off())
    }
  }, [projectId, socket.connected])

  return events
}
```

## 2. Payment Links

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

Payment confirmation is detected by the wallet indexers (see `functions.md`, section 5.2), not synchronously at checkout, so a checkout page should poll the public read, or subscribe to the wallet events in section 1 above, rather than assuming the payment is settled the moment the customer says they sent it.

## 3. Broadcasting Transactions

Three distinct surfaces, with genuinely different trust models. Picking the wrong one is an architecture mistake, not a routing detail. For the real-time side of wallets, deposit and confirmation events, see section 1 above; this section covers only the outbound paths.

| Surface | Route | Who holds the key | KYC gate |
|---------|-------|-------------------|----------|
| Custodial withdrawal | `POST /api/wallet/:walletId/withdraw` | mudbase | yes |
| Non-custodial broadcast | `POST /api/wallet/non-custodial/broadcast` | your user | yes |
| Account abstraction (EIP-4337) | `POST /api/wallet/non-custodial/account-abstraction/broadcast` | your user's smart account | yes |
| Stateless relay | `POST /api/tx/broadcast` | whoever signed it | yes |

### Custodial withdrawal

mudbase generated and holds the key; you ask it to send. Body is `{ toAddress, amount, network, options }`; auth is a user bearer token, and the wallet must belong to that user. Layered behind withdrawal-specific security middleware, tenant isolation, and a dedicated rate limiter on top of the KYC gate.

Use this when your product's users should not have to manage keys at all, and accept that you are then operating a custodial service, with everything that implies.

### Non-custodial broadcast

Your user signs locally; mudbase only relays and then tracks. Body is `{ chain, signedTx, fromAddress }`, `fromAddress` is required, because it is what links the broadcast to a registered address for later tracking, speed-up and cancel. Register addresses first via `POST /api/wallet/non-custodial/register-address`.

Companion endpoints: `POST /estimate-gas` before signing, `POST /speed-up` and `POST /cancel` to obtain replacement-transaction params for a stuck EVM transaction.

The account-abstraction variant takes the same trust model up a level for smart-contract accounts.

### Stateless relay

`POST /api/tx/broadcast` is the thinnest surface: `{ chain, signedTx, fromAddress?, projectId? }`, no wallet registration required, accepts a user bearer token *or* an API key. It answers "get this signed blob onto this chain" and nothing else. `fromAddress` is optional and used only to associate the broadcast with a registered wallet if one matches.

It is idempotent when you send `X-Idempotency-Key`, do, because a retried broadcast without one is a genuine double-spend risk on chains that accept the same signed transaction twice.

```typescript
// src/lib/mudbase.ts, inside class MudbaseClient
  async broadcastSignedTransaction(params: {
    chain: string
    signedTx: string
    fromAddress?: string
    idempotencyKey: string
  }): Promise<BroadcastResult> {
    const res = await this.request<{ success: boolean; message: string; data: BroadcastResult }>(
      'POST',
      '/api/tx/broadcast',
      {
        chain: params.chain,
        signedTx: params.signedTx,
        fromAddress: params.fromAddress,
        projectId: this.projectId,
      },
      { headers: { 'X-Idempotency-Key': params.idempotencyKey } }
    )
    return res.data
  }
```

```typescript
// src/lib/mudbase.ts, types
export interface BroadcastResult {
  txHash: string
  status: 'broadcast'
}
```

That method needs one addition to the `request()` helper, an optional per-call header bag (already reflected in `sdk-client.md`'s `request()` signature):

```typescript
// src/lib/mudbase.ts, the options parameter of request()
  private async request<T>(
    method: string,
    path: string,
    body?: unknown,
    options: { auth?: boolean; formData?: FormData; headers?: Record<string, string> } = {}
  ): Promise<T> {
    const headers: Record<string, string> = { ...options.headers }
    // ...rest of the method unchanged
```

Broadcasting is where a `403 KYC_REQUIRED` most commonly surprises people: the org is verified for its dashboard but a *different* org's key is in play, or verification lapsed. Always branch on the error `code`.

## 4. Merchant Payment Processing (fees)

Separate from wallets: mudbase also processes card and local-rail merchant payments (the checkout your project's customers pay through, not crypto). The fee mudbase charges the merchant is **cost-plus**: the underlying processor's real cost for that country/method, passed through, plus a mudbase margin, never a flat marked-up rate presented as if it were the processor's own price.

Margin tiers (all amounts USD):

| Tier | Rate | Fixed |
|------|------|-------|
| Local African rails | 1.0% | $0.20 |
| East/Southern Africa card | 0.9% | $0.20 |
| International | 1.0% | $0.30 |

Every computed fee is floored at **$0.30** so a very small transaction never charges an unrealistically thin fee. This cost-plus model applies to payment processing specifically; it is distinct from the legacy flat 7% + $0.50 rate, which still exists in the codebase but is now scoped only to the platform's own revenue split on **subscription** billing, not to merchant payment processing.

Never surface which processor mudbase routes a given country/method through in product copy or API responses, present the capability as first-party ("card and local-rail payments", not "payments via [processor]").

## 5. In-App Credit Balance

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

- `addons-and-kyc.md`, the KYC gate that fronts payment links and broadcasting, and add-on invocation billing against credit
- `realtime.md`, the shared Socket.IO client wallet events ride on top of
- `functions.md`, section 5.2, why wallet indexers are eventually consistent
