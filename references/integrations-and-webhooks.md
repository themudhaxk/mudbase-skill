# Integrations, Calls & Signaling, and Outbound Webhooks

## 1. Integrations Framework

mudbase has a native integrations system for connecting a project to third-party APIs your users configure (payment processors, communications providers, and similar). Use the integration room to execute endpoints and receive webhook mirrors in real-time.

### Join / leave

```typescript
const socket = getMudbaseSocket()

socket.emit('integration:join', {
  projectId: 'YOUR_PROJECT_ID',
  integrationId: 'INTEGRATION_ID',
}, (ack: unknown) => {
  const result = ack as { integrationId: string; projectId: string; room: string }
  console.log('joined:', result.room)
})

socket.emit('integration:leave', { integrationId: 'INTEGRATION_ID' })
```

### Execute an integration endpoint

```typescript
socket.emit('integration:execute', {
  projectId: 'YOUR_PROJECT_ID',
  integrationId: 'INTEGRATION_ID',
  endpoint: 'getUserData',
  params: { userId: '123' },
  requestData: {},
})

// Others in the room receive:
socket.on('integration:execution:started', (ev: { integrationId: string; endpoint: string }) => {
  setExecuting(true)
})

// Caller (and room) receives:
socket.on('integration:execution:completed', (ev: {
  integrationId: string
  endpoint: string
  result: { success: boolean; data: unknown }
  timestamp: string
}) => {
  setResult(ev.result.data)
  setExecuting(false)
})

socket.on('integration:execution:failed', (ev: { integrationId: string; error: string }) => {
  setError(ev.error)
  setExecuting(false)
})
```

### Webhook mirror

Receive inbound webhooks to an integration as Socket.IO events in real-time:

```typescript
// Subscribe to the webhook mirror stream
socket.emit('integration:webhook:subscribe', {
  projectId: 'YOUR_PROJECT_ID',
  integrationId: 'INTEGRATION_ID',
}, () => {
  console.log('webhook mirror active')
})

socket.on('integration:webhook:received', (ev: {
  integrationId: string
  event: string // e.g. a provider-specific event name from the connected third party
  data: unknown
  timestamp: string
}) => {
  console.log('webhook mirrored:', ev.event, ev.data)
  handleWebhookEvent(ev.event, ev.data)
})

// Unsubscribe
socket.emit('integration:webhook:unsubscribe', { integrationId: 'INTEGRATION_ID' })
```

### Usage monitoring

```typescript
socket.on('integration:usage:updated', (ev: {
  integrationId: string
  usage: { calls: number; errors: number; lastUsed: string }
}) => {
  updateUsageDisplay(ev.usage)
})
```

## 2. Calls & Signaling

mudbase provides signaling events for coordinating VoIP/video call UI. Not a full WebRTC stack, use a TURN/STUN provider for media transport; mudbase handles the signal coordination.

```typescript
const socket = getMudbaseSocket()

// Caller initiates
socket.emit('call:initiate', {
  chatId: 'CHAT_ID',
  type: 'video', // 'video' | 'audio'
  participants: ['userId1', 'userId2'],
})

// Callee receives
socket.on('call:incoming', (ev: {
  chatId: string
  callerId: string
  type: 'video' | 'audio'
  timestamp: string
}) => {
  showIncomingCallUI(ev)
})

// Accept
socket.emit('call:accept', { chatId: 'CHAT_ID', callerId: 'CALLER_ID' })
socket.on('call:accepted', (ev: { chatId: string; acceptedBy: string }) => {
  startMediaSession(ev.chatId)
})

// Reject
socket.emit('call:reject', { chatId: 'CHAT_ID', callerId: 'CALLER_ID' })
socket.on('call:rejected', (ev: { chatId: string; rejectedBy: string }) => {
  showCallRejectedUI()
})

// End
socket.emit('call:end', { chatId: 'CHAT_ID', participants: ['userId1', 'userId2'] })
socket.on('call:ended', (ev: { chatId: string; endedBy: string }) => {
  terminateMediaSession()
})
```

### `useCall` hook

```typescript
// src/hooks/useCall.ts
'use client'

import { useState, useCallback, useEffect } from 'react'
import { getMudbaseSocket } from '@/lib/mudbase-socket'

type CallState =
  | { status: 'idle' }
  | { status: 'ringing'; chatId: string; callerId: string; type: 'video' | 'audio' }
  | { status: 'active'; chatId: string; type: 'video' | 'audio' }
  | { status: 'ended' }

export function useCall() {
  const socket = getMudbaseSocket()
  const [callState, setCallState] = useState<CallState>({ status: 'idle' })

  useEffect(() => {
    const offIncoming = socket.on<{ chatId: string; callerId: string; type: 'video' | 'audio' }>(
      'call:incoming',
      (ev) => setCallState({ status: 'ringing', ...ev })
    )
    const offAccepted = socket.on<{ chatId: string }>(
      'call:accepted',
      (ev) => setCallState((prev) =>
        prev.status === 'ringing' || prev.status === 'active'
          ? { status: 'active', chatId: ev.chatId, type: prev.type }
          : prev
      )
    )
    const offEnded = socket.on<unknown>('call:ended', () => {
      setCallState({ status: 'ended' })
      setTimeout(() => setCallState({ status: 'idle' }), 3000)
    })
    const offRejected = socket.on<unknown>('call:rejected', () => setCallState({ status: 'idle' }))

    return () => { offIncoming(); offAccepted(); offEnded(); offRejected() }
  }, [])

  const initiateCall = useCallback((chatId: string, type: 'video' | 'audio', participants: string[]) => {
    socket.emit('call:initiate', { chatId, type, participants })
    setCallState({ status: 'active', chatId, type })
  }, [])

  const acceptCall = useCallback(() => {
    if (callState.status !== 'ringing') return
    socket.emit('call:accept', { chatId: callState.chatId, callerId: callState.callerId })
  }, [callState])

  const rejectCall = useCallback(() => {
    if (callState.status !== 'ringing') return
    socket.emit('call:reject', { chatId: callState.chatId, callerId: callState.callerId })
    setCallState({ status: 'idle' })
  }, [callState])

  const endCall = useCallback((participants: string[]) => {
    if (callState.status !== 'active') return
    socket.emit('call:end', { chatId: callState.chatId, participants })
  }, [callState])

  return { callState, initiateCall, acceptCall, rejectCall, endCall }
}
```

## 3. Outbound Webhooks (mudbase to your server)

A mudbase project has **one** webhook destination, configured as part of the project record, not a collection of webhook subscriptions. You set the URL, the signing secret, and the event allow-list together, and mudbase delivers every subscribed event to that single endpoint.

### Configuring the endpoint

`PUT /api/webhooks/projects/:projectId/config`, authenticated with a bearer token or API key holding `project:update`.

| Body field | Type | Notes |
|------------|------|-------|
| `webhookUrl` | string | Validated against the SSRF guard, no private ranges, no DNS rebinding. Setting the first URL counts against the plan's webhooks-per-project limit |
| `webhookSecret` | string | HMAC-SHA256 signing key. Write-only; reads report only `hasSecret: true` |
| `webhookEvents` | string[] | Allow-list. An **empty array means "all events"**, which is the legacy default, always send an explicit list |
| `webhookVersion` | string | Payload version pin |
| `transformations` | object[] | Optional payload reshaping applied before delivery |

`GET /api/webhooks/projects/:projectId/config` returns the same fields minus the secret.

```typescript
// src/lib/mudbase-webhooks.ts, server-side only
const MUDBASE_URL = process.env.MUDBASE_URL ?? 'https://cloud.mudbase.dev'
const PROJECT_ID = process.env.MUDBASE_PROJECT_ID
const ADMIN_TOKEN = process.env.MUDBASE_ADMIN_TOKEN
const WEBHOOK_SECRET = process.env.MUDBASE_WEBHOOK_SECRET

export type MudbaseWebhookEvent =
  | 'user.created'
  | 'user.updated'
  | 'user.deleted'
  | 'collection.insert'
  | 'collection.update'
  | 'collection.delete'
  | 'collection.created'
  | 'collection.updated'
  | 'collection.deleted'
  | 'data.created'
  | 'data.updated'
  | 'data.deleted'
  | 'file.uploaded'
  | 'file.deleted'
  | 'function.execution.completed'
  | 'function.execution.failed'
  | 'wallet.created'
  | 'wallet.transaction'
  | 'wallet.balance_changed'
  | 'subscription.created'
  | 'subscription.updated'
  | 'subscription.canceled'
  | 'payment.succeeded'
  | 'payment.failed'
  | 'email.delivered'
  | 'email.bounced'
  | 'email.complained'
  | 'api_key.created'
  | 'api_key.revoked'
  | 'backup.completed'
  | 'backup.failed'
  | 'backup.restored'
  | 'backup.restore_failed'

interface WebhookConfigResponse {
  success: boolean
  data: {
    webhookUrl: string | null
    webhookEvents: MudbaseWebhookEvent[]
    webhookVersion: string | null
  }
}

export async function configureWebhook(
  endpointUrl: string,
  events: MudbaseWebhookEvent[]
): Promise<WebhookConfigResponse['data']> {
  if (!PROJECT_ID || !ADMIN_TOKEN || !WEBHOOK_SECRET) {
    throw new Error('MUDBASE_PROJECT_ID, MUDBASE_ADMIN_TOKEN and MUDBASE_WEBHOOK_SECRET are required')
  }
  if (events.length === 0) {
    // An empty array is read as "deliver everything" server-side, almost never what you want.
    throw new Error('Refusing to configure a webhook with an empty event allow-list')
  }

  const res = await fetch(`${MUDBASE_URL}/api/webhooks/projects/${PROJECT_ID}/config`, {
    method: 'PUT',
    headers: {
      'Content-Type': 'application/json',
      Authorization: `Bearer ${ADMIN_TOKEN}`,
    },
    body: JSON.stringify({
      webhookUrl: endpointUrl,
      webhookSecret: WEBHOOK_SECRET,
      webhookEvents: events,
      webhookVersion: '2026-02-01',
    }),
  })

  if (!res.ok) {
    const body = (await res.json().catch(() => ({}))) as { error?: string; limit?: number }
    throw new Error(body.error ?? `Webhook config failed with status ${res.status}`)
  }

  const body = (await res.json()) as WebhookConfigResponse
  return body.data
}
```

### Event names

The names that actually appear in the `event` field come from the enum on the project model. Two traps:

- Document CRUD emits **`collection.insert` / `collection.update` / `collection.delete`**. There is no `document.*` event. (`data.created` / `data.updated` / `data.deleted` are accepted as subscription aliases in the enum, but the data routes dispatch the `collection.*` names, subscribe to those.)
- Function outcomes are **`function.execution.completed` / `function.execution.failed`**, matching the Socket.IO event names in `realtime.md`. There is no `function.completed`.

| Event | When |
|-------|------|
| `user.created` / `user.updated` / `user.deleted` | Project end-user lifecycle |
| `collection.insert` | Document created in a collection |
| `collection.update` | Document modified |
| `collection.delete` | Document removed (payload is `{ id }`) |
| `collection.created` / `collection.updated` / `collection.deleted` | The collection *definition* changed |
| `file.uploaded` / `file.deleted` | Storage object lifecycle |
| `function.execution.completed` / `function.execution.failed` | Serverless function run reached a terminal state |
| `payment.succeeded` / `payment.failed` | Payment outcome |
| `email.delivered` / `email.bounced` / `email.complained` | Provider delivery status |
| `wallet.created` / `wallet.transaction` / `wallet.balance_changed` | Legacy from the crypto-wallet stack archived to MudChain on 2026-08-23. Still a valid `webhookEvents` enum value, but nothing in mudbase emits it, do not build against it |
| `api_key.created` / `api_key.revoked` | Key lifecycle |
| `backup.completed` / `backup.failed` / `backup.restored` / `backup.restore_failed` | Backup jobs |

### Delivery headers and payload

```
Content-Type: application/json
User-Agent: MUDBASE-Webhook/1.0
X-MUDBASE-Event: collection.insert
X-MUDBASE-Project: <projectId>
X-MUDBASE-Signature: <hex HMAC-SHA256 of the JSON payload>
X-MUDBASE-Timestamp: <unix seconds>
X-MUDBASE-Api-Version: 2026-02-01
X-MUDBASE-Delivery-ID: <WebhookLog id, use this for idempotency>
X-MUDBASE-Delivery: <same value, legacy header name>
```

```jsonc
{ "event": "collection.insert", "data": { /* the document */ }, "timestamp": "2026-07-27T09:12:44.001Z", "projectId": "..." }
```

There is no `id` field inside the body, the deduplication key is the `X-MUDBASE-Delivery-ID` header.

### Receiving and verifying

The signature is the hex HMAC-SHA256 of the exact JSON body mudbase sent. Compare it in constant time, and check the length **before** calling `timingSafeEqual`, which throws `RangeError` on mismatched buffer lengths rather than returning `false`. A handler that lets that throw turns a malformed signature into a 500 instead of a clean 401.

```typescript
// src/app/api/webhooks/mudbase/route.ts (Next.js App Router)
import { NextRequest, NextResponse } from 'next/server'
import crypto from 'crypto'
import { revalidateTag } from 'next/cache'
import type { MudbaseWebhookEvent } from '@/lib/mudbase-webhooks'

const WEBHOOK_SECRET = process.env.MUDBASE_WEBHOOK_SECRET

// Illustrative idempotency guard, swap for a Redis SETNX with a 24h TTL in production.
// An in-memory Set does not survive a restart or work across multiple instances.
const processedWebhookDeliveries = new Set<string>()
const WEBHOOK_DEDUPE_TTL_MS = 24 * 60 * 60 * 1000

async function checkWebhookProcessed(deliveryId: string): Promise<boolean> {
  return processedWebhookDeliveries.has(deliveryId)
}

async function markWebhookProcessed(deliveryId: string): Promise<void> {
  processedWebhookDeliveries.add(deliveryId)
  setTimeout(() => processedWebhookDeliveries.delete(deliveryId), WEBHOOK_DEDUPE_TTL_MS)
}

// Each handler below invalidates the Next.js cache tags a real UI would have used to
// fetch that data (`fetch(url, { next: { tags: [...] } })`), which is the minimal real
// reaction to a mudbase event inside a Route Handler. Replace with your own side effects
// (queueing a job, writing an audit log, sending a notification) as needed.
async function handleUserCreated(data: unknown): Promise<void> {
  void data
  revalidateTag('users')
}

async function handleDocumentCreated(data: unknown): Promise<void> {
  void data
  revalidateTag('collection-documents')
}

async function handleFunctionCompleted(data: unknown): Promise<void> {
  void data
  revalidateTag('function-executions')
}

async function handlePaymentSucceeded(data: unknown): Promise<void> {
  void data
  revalidateTag('payments')
}

interface MudbaseWebhookPayload {
  event: MudbaseWebhookEvent
  data: unknown
  timestamp: string
  projectId: string
}

/** Fails closed on a missing, malformed, or wrong-length signature. */
function verifySignature(rawBody: string, signature: string, secret: string): boolean {
  const expected = crypto.createHmac('sha256', secret).update(rawBody).digest('hex')

  // Length check first: timingSafeEqual throws RangeError on unequal lengths.
  // The comparison below is still constant-time for equal-length inputs, which is
  // where the timing signal would actually be.
  if (signature.length !== expected.length) return false

  try {
    return crypto.timingSafeEqual(Buffer.from(signature, 'hex'), Buffer.from(expected, 'hex'))
  } catch {
    // Non-hex input produces a short buffer; treat as invalid rather than crashing.
    return false
  }
}

export async function POST(req: NextRequest): Promise<NextResponse> {
  if (!WEBHOOK_SECRET) {
    return NextResponse.json({ error: 'Webhook secret not configured' }, { status: 500 })
  }

  const rawBody = await req.text()
  const signature = req.headers.get('x-mudbase-signature') ?? ''
  const deliveryId = req.headers.get('x-mudbase-delivery-id') ?? ''

  if (!verifySignature(rawBody, signature, WEBHOOK_SECRET)) {
    return NextResponse.json({ error: 'Invalid signature' }, { status: 401 })
  }
  if (!deliveryId) {
    return NextResponse.json({ error: 'Missing delivery id' }, { status: 400 })
  }

  let event: MudbaseWebhookPayload
  try {
    event = JSON.parse(rawBody) as MudbaseWebhookPayload
  } catch {
    return NextResponse.json({ error: 'Malformed JSON body' }, { status: 400 })
  }

  // Idempotency keyed on the delivery header, retries reuse the same id.
  if (await checkWebhookProcessed(deliveryId)) {
    return NextResponse.json({ ok: true })
  }
  await markWebhookProcessed(deliveryId) // TTL: 24h

  switch (event.event) {
    case 'user.created':
      await handleUserCreated(event.data)
      break
    case 'collection.insert':
      await handleDocumentCreated(event.data)
      break
    case 'function.execution.completed':
      await handleFunctionCompleted(event.data)
      break
    case 'payment.succeeded':
      await handlePaymentSucceeded(event.data)
      break
    default:
      // Unknown or unhandled event, still 200 so mudbase stops retrying.
      break
  }

  return NextResponse.json({ ok: true })
}
```

### Retry policy

Three attempts total, at fixed intervals, not five, and not exponential out to hours. A background sweep runs every 5 minutes and picks up anything still pending.

| Attempt | Delay after the previous failure |
|---------|----------------------------------|
| 1 | Immediate |
| 2 | 1 minute |
| 3 | 5 minutes |

(A fourth retry delay of 30 minutes is configured but only reached when `WEBHOOK_MAX_ATTEMPTS` is raised above the default 3; `WEBHOOK_MAX_ATTEMPTS` and `WEBHOOK_RETRY_DELAYS` are deployment-level env overrides, not per-project settings.)

Return `200` as soon as you have durably accepted the payload, then do the work asynchronously. After the final attempt the delivery is abandoned, `POST /api/webhooks/retry/:webhookId` re-drives a specific failed delivery, and `GET /api/webhooks/projects/:projectId` lists the delivery log.

## See also

- `realtime.md`, the shared Socket.IO client these call/integration events ride on top of
- `messaging.md`, the `email.*` domain events this webhook surface can deliver
- `wallets-and-payments.md`, the `payment.*` domain events this webhook surface can deliver (`wallet.*` is a legacy enum value nothing emits anymore, see the table above)
