# Serverless Functions & Background Processing

## 1. Execution is asynchronous

`POST /api/functions/projects/:projectId/functions/:functionId/execute` returns **`202 Accepted`** with `{ success: true, data: { executionId, status: "queued" } }`. It never returns the function's result. The run is enqueued onto a queue and picked up by a worker process; the result lands somewhere else. Two ways to collect it:

1. **Poll** `GET /api/functions/projects/:projectId/functions/:functionId/executions/:executionId`, returns `{ executionId, status, durationMs, error, errorClass, logs, result, createdAt, startedAt, completedAt }` with `status` in `queued | running | success | failed`.
2. **Subscribe** to the Socket.IO `function.execution.completed` / `function.execution.failed` events (see `realtime.md`). Cheaper and lower-latency when you already hold a socket; the poller below is still the right fallback for a page loaded fresh with an `executionId` in the URL.

Anything written against an "invoke returns the result" assumption will silently receive `{ executionId, status }` where it expected a domain object.

## 2. Invoking Functions from Client

```typescript
// src/lib/functions.ts
import { getMudbaseClient } from '@/lib/mudbase'
import type { FunctionExecution, FunctionInvocation } from '@/lib/mudbase'

interface ProcessOrderPayload {
  orderId: string
  items: Array<{ productId: string; quantity: number }>
  shippingAddress: {
    street: string
    city: string
    country: string
  }
}

export interface ProcessOrderResult {
  transactionRef: string
  estimatedDelivery: string
}

/** Fire-and-collect-later: returns the handle, not the outcome. */
export async function startProcessOrder(
  payload: ProcessOrderPayload
): Promise<FunctionInvocation> {
  return getMudbaseClient().invokeFunction('process-order', { ...payload })
}

export class FunctionExecutionError extends Error {
  constructor(
    message: string,
    public readonly executionId: string,
    public readonly errorClass: string | null,
    public readonly logs: FunctionExecution['logs']
  ) {
    super(message)
    this.name = 'FunctionExecutionError'
  }
}

interface AwaitOptions {
  /** Total wall-clock budget before giving up. */
  timeoutMs?: number
  intervalMs?: number
  signal?: AbortSignal
}

/**
 * Poll an execution to a terminal state. Timing out does not cancel the run,
 * the function keeps executing server-side; only the client stops waiting.
 */
export async function awaitFunctionResult<TResult>(
  functionId: string,
  executionId: string,
  options: AwaitOptions = {}
): Promise<TResult> {
  const { timeoutMs = 60_000, intervalMs = 1_000, signal } = options
  const client = getMudbaseClient()
  const deadline = Date.now() + timeoutMs

  while (Date.now() < deadline) {
    if (signal?.aborted) {
      throw new Error('Aborted while awaiting function result')
    }

    const execution = await client.getFunctionExecution<TResult>(functionId, executionId)

    if (execution.status === 'success') {
      if (execution.result === null) {
        throw new FunctionExecutionError(
          'Function reported success but returned no result',
          executionId,
          execution.errorClass,
          execution.logs
        )
      }
      return execution.result
    }
    if (execution.status === 'failed') {
      throw new FunctionExecutionError(
        execution.error ?? 'Function execution failed',
        executionId,
        execution.errorClass,
        execution.logs
      )
    }

    await new Promise<void>((resolve) => setTimeout(resolve, intervalMs))
  }

  throw new Error(
    `Function ${functionId} execution ${executionId} did not finish within ${timeoutMs}ms`
  )
}

/** Convenience wrapper for callers that genuinely want request/response semantics. */
export async function processOrder(
  payload: ProcessOrderPayload
): Promise<ProcessOrderResult> {
  const { executionId } = await startProcessOrder(payload)
  return awaitFunctionResult<ProcessOrderResult>('process-order', executionId)
}
```

## 3. React Query Hooks for Functions

```typescript
// src/hooks/useFunction.ts
import { useMutation, useQuery } from '@tanstack/react-query'
import { useMudbase } from '@/lib/mudbase-provider'
import type { FunctionExecution, FunctionInvocation } from '@/lib/mudbase'

/** Starts the run. `data.executionId` is what you feed to useFunctionExecution below. */
export function useInvokeFunction<TPayload extends Record<string, unknown>>(
  functionId: string
) {
  const { client } = useMudbase()
  return useMutation<FunctionInvocation, Error, TPayload>({
    mutationFn: (payload) => client.invokeFunction(functionId, payload),
  })
}

/**
 * Polls a single execution until it reaches a terminal state, then stops.
 * Pair with the Socket.IO listener in `realtime.md` when you want push instead of pull;
 * this hook is the fallback for a cold page load holding only an executionId.
 */
export function useFunctionExecution<TResult>(
  functionId: string,
  executionId: string | null,
  pollIntervalMs = 1_000
) {
  const { client } = useMudbase()
  return useQuery<FunctionExecution<TResult>>({
    queryKey: ['function', functionId, 'execution', executionId],
    queryFn: () => {
      if (executionId === null) throw new Error('useFunctionExecution called without an executionId')
      return client.getFunctionExecution<TResult>(functionId, executionId)
    },
    enabled: executionId !== null,
    refetchInterval: (query) => {
      const status = query.state.data?.status
      return status === 'success' || status === 'failed' ? false : pollIntervalMs
    },
    // A queued execution has no result yet; that is expected, not stale data.
    staleTime: 0,
  })
}
```

```typescript
// src/components/OrderSubmit.tsx
'use client'

import { useState } from 'react'
import { useInvokeFunction, useFunctionExecution } from '@/hooks/useFunction'
import type { ProcessOrderResult } from '@/lib/functions'

export function OrderSubmit({ orderId }: { orderId: string }) {
  const [executionId, setExecutionId] = useState<string | null>(null)
  const invoke = useInvokeFunction<{ orderId: string }>('process-order')
  const execution = useFunctionExecution<ProcessOrderResult>('process-order', executionId)

  const handleSubmit = async (): Promise<void> => {
    const started = await invoke.mutateAsync({ orderId })
    setExecutionId(started.executionId)
  }

  const pending =
    invoke.isPending ||
    execution.data?.status === 'queued' ||
    execution.data?.status === 'running'

  return (
    <div className="space-y-2">
      <button
        type="button"
        onClick={handleSubmit}
        disabled={pending}
        className="rounded-md bg-blue-600 px-4 py-2 text-white disabled:opacity-50"
      >
        {pending ? 'Processing order...' : 'Submit order'}
      </button>
      {execution.data?.status === 'failed' && (
        <p className="text-sm text-red-600">{execution.data.error ?? 'Order processing failed'}</p>
      )}
      {execution.data?.status === 'success' && execution.data.result && (
        <p className="text-sm text-green-700">
          Confirmed, reference {execution.data.result.transactionRef}, arriving{' '}
          {execution.data.result.estimatedDelivery}
        </p>
      )}
    </div>
  )
}
```

## 4. Decision: Function vs Client Logic

| Use a Function | Use Client Logic |
|----------------|-----------------|
| Calls third-party APIs with secret keys | UI state management |
| Needs access to other collections/users' data | Form validation |
| Scheduled or triggered by events | Client-only computations |
| Complex data transformations | Local filtering/sorting of fetched data |
| Webhook handling | Animation/interaction |

Because execution is queued, a function is the wrong tool for anything on a synchronous critical path, a form submit that must show a result within a few hundred milliseconds is better served by a data-plane write plus a document trigger.

## 5. Background Processing: what actually runs where

Two separate things, often conflated. Be precise about which one you are relying on.

### 5.1 Your functions: a real queue

Function execution is genuinely queued, not simulated. `POST .../execute` writes a `FunctionExecution` record, pushes onto a queue, and returns `202`; a dedicated worker process drains it. Section 2 above covers the client contract (poll the execution endpoint, or listen for `function.execution.completed` / `.failed`). Scheduled functions are driven by a cron job process, you configure the schedule in the console, and the client only ever observes results.

There is no separate "jobs API". If you want a background job, you deploy a function and invoke or schedule it.

### 5.2 Platform workers: server-side, not client-callable

mudbase also runs a set of its own background processes. You cannot invoke these, and there is no client API for them, but they explain latency you will otherwise misread as a bug:

| Process | Why it matters to a client app |
|---------|-------------------------------|
| Usage metering worker | API-call and storage counters flush asynchronously, so a usage dashboard can lag a burst of writes by a short interval |
| Incoming-payment webhook | Stablecoin payment link confirmation is detected asynchronously by an incoming-payment webhook, not synchronously at checkout, a payment link can take a short interval to settle after the customer sends it (see `wallets-and-payments.md`) |
| Webhook retry sweep | Runs every 5 minutes; a failed delivery is retried on that cadence, not instantly |
| Project clone / export / migration-import workers | Long operations return a `jobId` and complete out of band, poll the corresponding job-status endpoint (see `migration-import.md`) |
| Add-on worker | Add-on invocations may return `202` with a pending job; poll it (see `addons-and-kyc.md`) |
| Billing crons (overage, credit low-balance, payout scheduler) | Invoices, credit alerts and payouts materialize on a schedule, not the instant a threshold is crossed |
| Daily storage / usage-integrity crons | Storage totals and integrity corrections settle daily |

Practical consequence: treat any counter, balance, or usage figure mudbase reports as eventually consistent, and never build a client-side gate ("block the user once usage hits the limit") on a freshly-read counter alone. The server enforces its own limits at request time; that is the authoritative check.

## See also

- `realtime.md`, the `function.execution.completed` / `.failed` Socket.IO events
- `mcp-and-ai-agents.md`, function invoke/logs as MCP tools available to an AI agent
- `migration-import.md`, `addons-and-kyc.md`, the job-status polling pattern for long-running platform work
