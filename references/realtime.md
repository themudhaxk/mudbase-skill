# Real-time (Socket.IO, Database Events, Presence, Custom Channels, SSE)

mudbase real-time runs on **Socket.IO**, not a raw WebSocket connection. If you see an older integration using `new WebSocket(...)` against a mudbase URL directly, replace it, that surface is superseded by the client below and no longer how any real-time feature (database events, chat, presence, custom channels) is delivered.

## 1. Connection & Setup

```bash
npm install socket.io-client
```

```typescript
// src/lib/mudbase-socket.ts
import { io, type Socket } from 'socket.io-client'

const MUDBASE_URL = process.env.NEXT_PUBLIC_MUDBASE_URL ?? 'https://cloud.mudbase.dev'

export type SocketStatus = 'disconnected' | 'connecting' | 'connected' | 'error'

export class MudbaseSocket {
  private socket: Socket | null = null
  private token: string | null = null
  private statusListeners: Set<(s: SocketStatus) => void> = new Set()

  connect(token: string): void {
    if (this.socket?.connected) return
    this.token = token

    this.socket = io(MUDBASE_URL, {
      path: '/socket.io/',
      transports: ['websocket', 'polling'],
      auth: { token }, // JWT sent in the handshake
      reconnection: true,
      reconnectionDelay: 1000,
      reconnectionDelayMax: 30000,
      reconnectionAttempts: 10,
    })

    this.socket.on('connect', () => this.emitStatus('connected'))
    this.socket.on('disconnect', () => this.emitStatus('disconnected'))
    this.socket.on('connect_error', (err) => {
      console.error('[mudbase socket] connect_error:', err.message)
      this.emitStatus('error')
    })

    // Plan limit reached
    this.socket.on('error', (data: { message: string; limit?: number }) => {
      console.error('[mudbase socket] error:', data.message)
      this.socket?.disconnect()
    })
  }

  disconnect(): void {
    this.socket?.disconnect()
    this.socket = null
  }

  get raw(): Socket | null {
    return this.socket
  }

  get connected(): boolean {
    return this.socket?.connected ?? false
  }

  onStatus(cb: (s: SocketStatus) => void): () => void {
    this.statusListeners.add(cb)
    return () => this.statusListeners.delete(cb)
  }

  private emitStatus(s: SocketStatus): void {
    this.statusListeners.forEach((cb) => cb(s))
  }

  /** Emit with an optional acknowledgement callback. `data` is omittable for signal-only events. */
  emit(event: string, data?: unknown, ack?: (res: unknown) => void): void {
    if (!this.socket?.connected) {
      console.warn(`[mudbase socket] emit "${event}" while not connected`)
      return
    }
    if (ack) {
      this.socket.emit(event, data, ack)
    } else if (data !== undefined) {
      this.socket.emit(event, data)
    } else {
      this.socket.emit(event)
    }
  }

  /** Subscribe to a server event; returns an unsubscribe fn. */
  on<T = unknown>(event: string, handler: (data: T) => void): () => void {
    const wrapped = (payload: unknown): void => handler(payload as T)
    this.socket?.on(event, wrapped)
    return () => this.socket?.off(event, wrapped)
  }
}

// Singleton
let _socket: MudbaseSocket | null = null

export function getMudbaseSocket(): MudbaseSocket {
  if (!_socket) _socket = new MudbaseSocket()
  return _socket
}
```

### Rooms auto-joined on connect

| Room | Purpose |
|------|---------|
| `org:<orgId>` | Org-level events (builder/dashboard scope) |
| `user:<userId>` | Direct messages and user-specific pushes |

All other rooms require explicit subscription via emit.

### Auth requirements

JWT `scope` must be `api` or `websocket`. Invalid/expired tokens fail the handshake with `connect_error`, message `Authentication error`. A token with `jti` must not be blacklisted; a token with `sid` requires an active session.

### React hook

```typescript
// src/hooks/useSocket.ts
'use client'

import { useEffect, useState } from 'react'
import { getMudbaseSocket, type SocketStatus } from '@/lib/mudbase-socket'
import { useMudbase } from '@/lib/mudbase-provider'

export function useSocket(): { status: SocketStatus } {
  const { session } = useMudbase()
  const [status, setStatus] = useState<SocketStatus>('disconnected')

  useEffect(() => {
    const socket = getMudbaseSocket()

    if (session?.token) {
      socket.connect(session.token)
    }

    const off = socket.onStatus(setStatus)
    return off
  }, [session?.token])

  return { status }
}
```

## 2. Database Events

### Room subscriptions

```typescript
const socket = getMudbaseSocket()

// Subscribe to an entire collection (all documents)
socket.emit('subscribe:collection', {
  projectId: 'YOUR_PROJECT_ID',
  collectionId: 'YOUR_COLLECTION_ID',
}, (ack: unknown) => {
  console.log('subscribed:', ack)
  // { type: 'collection', id: '...', projectId: '...' }
})

// String form (no ack)
socket.emit('subscribe:collection', 'YOUR_COLLECTION_ID')

// Subscribe to a single document
socket.emit('subscribe:document', {
  projectId: 'YOUR_PROJECT_ID',
  collectionId: 'YOUR_COLLECTION_ID',
  documentId: 'YOUR_DOCUMENT_ID',
})

// Subscribe to a named query (filtered push)
socket.emit('subscribe:query', {
  projectId: 'YOUR_PROJECT_ID',
  collectionId: 'YOUR_COLLECTION_ID',
  queryId: 'active-orders', // client-chosen stable ID
  query: { status: 'active' },
})

// Subscribe to project-level events
socket.emit('subscribe:project', 'YOUR_PROJECT_ID')
```

### Schema events (collection lifecycle)

Delivered to the `project:<projectId>` room.

| Event | When |
|-------|------|
| `db:collection_created` | Collection created |
| `db:collection_updated` | Schema or metadata updated |
| `db:collection_deleted` | Collection deleted |

```typescript
interface CollectionEvent {
  projectId: string
  collectionId: string
  action: 'collection_created' | 'collection_updated' | 'collection_deleted'
  data: {
    _id: string
    name: string
    slug: string
    fields: unknown[]
    project: string
  }
  timestamp: string
}

socket.on<CollectionEvent>('db:collection_created', (ev) => {
  console.log('New collection:', ev.data.name)
})
```

### Row CRUD events

Delivered to the `collection:<collectionId>` and `project:<projectId>` rooms.

| Event | When |
|-------|------|
| `db:create` | Document created |
| `db:update` | Document updated |
| `db:delete` | Document deleted |

```typescript
interface RowEvent {
  projectId: string
  collectionId: string
  action: 'create' | 'update' | 'delete'
  data: Record<string, unknown> // full document (or { _id } on delete)
  timestamp: string
}

socket.on<RowEvent>('db:create', (ev) => {
  console.log('New document:', ev.data._id)
})
```

Additional events from `emitDataChange`:
- `data:change`, collection room
- `document:change`, document room
- `project:data:change`, project room

### React Query live sync hook

```typescript
// src/hooks/useCollectionLive.ts
import { useEffect } from 'react'
import { useQueryClient } from '@tanstack/react-query'
import { getMudbaseSocket } from '@/lib/mudbase-socket'
import type { Document } from '@/lib/mudbase'

interface RowEvent {
  projectId: string
  collectionId: string
  action: string
  data: Document
  timestamp: string
}

export function useCollectionLive(
  collectionId: string,
  projectId: string,
  enabled = true
): void {
  const queryClient = useQueryClient()
  const socket = getMudbaseSocket()

  useEffect(() => {
    if (!enabled || !socket.connected) return

    socket.emit('subscribe:collection', { projectId, collectionId })

    const offCreate = socket.on<RowEvent>('db:create', (ev) => {
      if (ev.collectionId !== collectionId) return
      queryClient.setQueryData(['collection', collectionId, 'doc', ev.data._id], ev.data)
      queryClient.invalidateQueries({ queryKey: ['collection', collectionId] })
    })

    const offUpdate = socket.on<RowEvent>('db:update', (ev) => {
      if (ev.collectionId !== collectionId) return
      queryClient.setQueryData(['collection', collectionId, 'doc', ev.data._id], ev.data)
      queryClient.invalidateQueries({ queryKey: ['collection', collectionId] })
    })

    const offDelete = socket.on<RowEvent>('db:delete', (ev) => {
      if (ev.collectionId !== collectionId) return
      queryClient.removeQueries({ queryKey: ['collection', collectionId, 'doc', ev.data._id] })
      queryClient.invalidateQueries({ queryKey: ['collection', collectionId] })
    })

    return () => {
      socket.emit('unsubscribe:collection', { collectionId })
      offCreate()
      offUpdate()
      offDelete()
    }
  }, [collectionId, projectId, enabled, socket.connected])
}
```

Also usable for a lighter, non-React-Query subscription: `src/hooks/useRealtimeSubscription.ts` wraps `client.subscribe(channel, event, callback)` directly for cases that do not need cache reconciliation:

```typescript
// src/hooks/useRealtimeSubscription.ts
import { useEffect, useRef } from 'react'
import { useMudbase } from '@/lib/mudbase-provider'

export function useRealtimeSubscription(
  channel: string,
  event: string,
  callback: (data: unknown) => void,
  enabled = true
): void {
  const { client } = useMudbase()
  const callbackRef = useRef(callback)
  callbackRef.current = callback

  useEffect(() => {
    if (!enabled) return

    const stableCallback = (data: unknown) => callbackRef.current(data)
    const unsubscribe = client.subscribe(channel, event, stableCallback)

    return () => {
      unsubscribe()
    }
  }, [client, channel, event, enabled])
}
```

Common channels for this lighter subscription form: `collection:{collectionId}` (`create`/`update`/`delete`), `user:{userId}` (`notification`/`role_updated`/`session_revoked`), `project:{projectId}` (`analytics`/`function_executed`).

## 3. Presence

Track who is online in a project. Presence is project-scoped.

```typescript
const socket = getMudbaseSocket()

// Announce online
socket.emit('presence:online', { projectId: 'YOUR_PROJECT_ID' })

// Get the current online list
socket.emit('presence:get:online', { projectId: 'YOUR_PROJECT_ID' })

// Update last-seen timestamp
socket.emit('presence:update') // no payload needed

// Go offline
socket.emit('presence:offline')
```

### Server events

```typescript
interface PresenceUser {
  id: string
  email: string
  firstName: string
  lastName: string
}

interface PresenceOnlineEvent {
  userId: string
  user: PresenceUser
  timestamp: string
}

interface PresenceListEvent {
  projectId: string
  users: Array<{
    userId: string
    user: PresenceUser
    lastSeen: string
  }>
  count: number
}

socket.on<PresenceOnlineEvent>('presence:user:online', (ev) => {
  addOnlineUser(ev.userId, ev.user)
})

socket.on<{ userId: string; timestamp: string }>('presence:user:offline', (ev) => {
  removeOnlineUser(ev.userId)
})

socket.on<PresenceListEvent>('presence:online:list', (ev) => {
  setOnlineUsers(ev.users)
})
```

### `usePresence` hook

```typescript
// src/hooks/usePresence.ts
'use client'

import { useState, useEffect, useCallback } from 'react'
import { getMudbaseSocket } from '@/lib/mudbase-socket'

interface OnlineUser {
  userId: string
  user: { id: string; email: string; firstName: string; lastName: string }
  lastSeen: string
}

export function usePresence(projectId: string) {
  const socket = getMudbaseSocket()
  const [onlineUsers, setOnlineUsers] = useState<OnlineUser[]>([])

  useEffect(() => {
    if (!socket.connected) return

    socket.emit('presence:online', { projectId })
    socket.emit('presence:get:online', { projectId })

    const offList = socket.on<{ users: OnlineUser[] }>('presence:online:list', ({ users }) => {
      setOnlineUsers(users)
    })

    const offOnline = socket.on<{ userId: string; user: OnlineUser['user']; timestamp: string }>(
      'presence:user:online',
      (ev) => {
        setOnlineUsers((prev) => {
          if (prev.some((u) => u.userId === ev.userId)) return prev
          return [...prev, { userId: ev.userId, user: ev.user, lastSeen: ev.timestamp }]
        })
      }
    )

    const offOffline = socket.on<{ userId: string }>('presence:user:offline', ({ userId }) => {
      setOnlineUsers((prev) => prev.filter((u) => u.userId !== userId))
    })

    // Keep lastSeen fresh
    const heartbeat = setInterval(() => {
      socket.emit('presence:update')
    }, 30_000)

    return () => {
      socket.emit('presence:offline')
      offList(); offOnline(); offOffline()
      clearInterval(heartbeat)
    }
  }, [projectId, socket.connected])

  const isOnline = useCallback(
    (userId: string) => onlineUsers.some((u) => u.userId === userId),
    [onlineUsers]
  )

  return { onlineUsers, onlineCount: onlineUsers.length, isOnline }
}
```

## 4. Custom Channels

Arbitrary real-time events scoped to a project, use for collaborative features, typing indicators, kanban boards, and similar.

```typescript
const socket = getMudbaseSocket()

// Broadcast a named event to everyone in the project
socket.emit('custom:broadcast', {
  projectId: 'YOUR_PROJECT_ID',
  event: 'board_updated', // becomes "custom:board_updated" on recipients
  payload: { columnId: 'done', cardId: 'abc' },
  room: undefined, // omit to broadcast to the whole project
})

// Recipients listen for custom:<event>
socket.on('custom:board_updated', (ev: {
  columnId: string
  cardId: string
  sender: string
  timestamp: string
}) => {
  moveCard(ev.cardId, ev.columnId)
})
```

### Join a named custom room

```typescript
socket.emit('custom:join', { room: 'whiteboard-session-42' }, () => {
  console.log('joined custom room')
})

socket.emit('custom:leave', { room: 'whiteboard-session-42' })
```

### Direct message to a user

```typescript
socket.emit('custom:message', {
  targetUserId: 'TARGET_USER_ID',
  message: 'Hey, are you there?',
  type: 'text',
})

socket.on('custom:message', (ev: {
  from: string
  message: string
  type: string
  timestamp: string
}) => {
  showDirectMessage(ev)
})
```

### Typing / presence in custom contexts

```typescript
// Collaborative typing indicator (document editor, etc.)
socket.emit('custom:typing', {
  projectId: 'YOUR_PROJECT_ID',
  isTyping: true,
  context: 'doc-editor-123',
})

socket.on('custom:typing', (ev: {
  userId: string
  isTyping: boolean
  context?: string
}) => {
  updateTypingIndicator(ev.userId, ev.isTyping, ev.context)
})

// Custom presence (e.g. "viewing page X")
socket.emit('custom:presence', {
  projectId: 'YOUR_PROJECT_ID',
  status: 'editing',
  metadata: { documentId: 'doc-123', cursorPosition: 42 },
})
```

## 5. Server-Sent Events (SSE)

One-way event stream from mudbase for notifications, task progress, and activity. Simpler than Socket.IO when you do not need bidirectional communication.

**Endpoint:** `GET https://cloud.mudbase.dev/realtime/events`

```typescript
// src/hooks/useSSEStream.ts
'use client'

import { useEffect, useRef, useState } from 'react'

interface SSEEvent {
  type: string
  payload: unknown
}

export function useSSEStream(token: string | null, enabled = true) {
  const [events, setEvents] = useState<SSEEvent[]>([])
  const [connected, setConnected] = useState(false)
  const esRef = useRef<EventSource | null>(null)

  useEffect(() => {
    if (!token || !enabled) return

    const url = `https://cloud.mudbase.dev/realtime/events?token=${encodeURIComponent(token)}`
    const es = new EventSource(url)
    esRef.current = es

    es.onopen = () => setConnected(true)
    es.onerror = () => setConnected(false)

    es.onmessage = (event) => {
      try {
        const data = JSON.parse(event.data) as SSEEvent
        setEvents((prev) => [data, ...prev].slice(0, 200))
      } catch {}
    }

    return () => {
      es.close()
      esRef.current = null
      setConnected(false)
    }
  }, [token, enabled])

  return { events, connected }
}
```

Event format:

```
data: {"type":"notification","payload":{"message":"Your export is ready","url":"/exports/123"}}

data: {"type":"backup.progress","payload":{"percent":75,"id":"backup_abc"}}

data: {"type":"function.execution.completed","payload":{"functionId":"process-order","executionId":"...","result":{}}}
```

Use SSE for: one-way push notifications, progress bars, activity logs, build/job status updates.
Use Socket.IO for: chat, presence, wallet events, bidirectional communication.

## See also

- `messaging.md`, the chat-specific Socket.IO events (`chat:join`, `message:send`, typing indicators) and the separate REST messaging surface (email/SMS/push)
- `integrations-and-webhooks.md`, call signaling events, which ride the same Socket.IO connection
