# Chat & Messaging

Two distinct surfaces share the name "messaging": real-time **chat** (Socket.IO, for in-app conversations) and the **REST messaging API** (transactional and bulk email/SMS/push, outside any chat room). Pick the right one, they are not interchangeable and the REST surface is not documented anywhere else in this skill.

## 1. Real-time chat (Socket.IO)

Uses the same `getMudbaseSocket()` client set up in `realtime.md`.

### Join / leave a chat room

```typescript
const socket = getMudbaseSocket()

// Join
socket.emit('chat:join', {
  chatId: 'CHAT_ID',
  projectId: 'YOUR_PROJECT_ID',
}, (ack: unknown) => {
  console.log('chat:joined', ack)
})

// Leave
socket.emit('chat:leave', { chatId: 'CHAT_ID' })
```

### Typing indicators

```typescript
socket.emit('chat:typing:start', { chatId: 'CHAT_ID' })
socket.emit('chat:typing:stop', { chatId: 'CHAT_ID' })

// Listen
socket.on<{ userId: string; chatId: string }>('user:typing:start', (ev) => {
  setTypingUsers((prev) => new Set([...prev, ev.userId]))
})
socket.on<{ userId: string; chatId: string }>('user:typing:stop', (ev) => {
  setTypingUsers((prev) => { const s = new Set(prev); s.delete(ev.userId); return s })
})
```

### Send message

```typescript
interface SendMessagePayload {
  chatId: string
  projectId: string
  type: 'text' | 'image' | 'file' | 'audio'
  content: string
  replyTo?: string | null
  mentions?: string[]
}

interface MessageSentAck {
  messageId: string
  timestamp: string
}

socket.emit('message:send', {
  chatId: 'CHAT_ID',
  projectId: 'YOUR_PROJECT_ID',
  type: 'text',
  content: 'Hello!',
  replyTo: null,
  mentions: [],
} satisfies SendMessagePayload, (ack: MessageSentAck) => {
  console.log('sent:', ack.messageId)
})
```

### Receive new messages

```typescript
interface NewMessageEvent {
  message: {
    _id: string
    content: string
    sender: string
    chat: string
    type: string
    replyTo?: string
    mentions?: string[]
    createdAt: string
  }
  chat: {
    _id: string
    name: string
    type: 'direct' | 'group'
  }
}

socket.on<NewMessageEvent>('message:new', (ev) => {
  addMessage(ev.message)
})
```

### React, edit, delete, mark read

```typescript
// React
socket.emit('message:react', { chatId, projectId, messageId, emoji: 'thumbsup' })
socket.on('message:reaction', (ev: { messageId: string; emoji: string; userId: string }) => {
  updateReaction(ev)
})

// Edit
socket.emit('message:edit', { chatId, projectId, messageId, content: 'Edited text' })
socket.on('message:edited', (ev: { messageId: string; content: string; editedAt: string }) => {
  updateMessage(ev.messageId, ev.content)
})

// Delete
socket.emit('message:delete', { chatId, projectId, messageId })
socket.on('message:deleted', (ev: { messageId: string }) => {
  removeMessage(ev.messageId)
})

// Mark read
socket.emit('messages:mark:read', { chatId, projectId, messageIds: ['id1', 'id2'] })
socket.on('messages:read', (ev: { chatId: string; messageIds: string[]; readBy: string }) => {
  markMessagesRead(ev.messageIds, ev.readBy)
})
```

### `useChat` hook

```typescript
// src/hooks/useChat.ts
'use client'

import { useState, useEffect, useCallback, useRef } from 'react'
import { getMudbaseSocket } from '@/lib/mudbase-socket'
import { useMudbase } from '@/lib/mudbase-provider'

interface ChatMessage {
  _id: string
  content: string
  sender: string
  type: string
  replyTo?: string
  createdAt: string
  reactions?: Record<string, string[]>
}

export function useChat(chatId: string, projectId: string) {
  const { session } = useMudbase()
  const socket = getMudbaseSocket()
  const [messages, setMessages] = useState<ChatMessage[]>([])
  const [typingUsers, setTypingUsers] = useState<Set<string>>(new Set())
  const [joined, setJoined] = useState(false)
  const typingTimers = useRef<Map<string, ReturnType<typeof setTimeout>>>(new Map())

  useEffect(() => {
    if (!socket.connected || !chatId) return

    socket.emit('chat:join', { chatId, projectId }, () => setJoined(true))

    const offNew = socket.on<{ message: ChatMessage }>('message:new', ({ message }) => {
      setMessages((prev) => [...prev, message])
    })

    const offTypingStart = socket.on<{ userId: string }>('user:typing:start', ({ userId }) => {
      setTypingUsers((prev) => new Set([...prev, userId]))
      // Auto-clear after 3s in case the stop event is missed
      const existing = typingTimers.current.get(userId)
      if (existing) clearTimeout(existing)
      typingTimers.current.set(userId, setTimeout(() => {
        setTypingUsers((prev) => { const s = new Set(prev); s.delete(userId); return s })
      }, 3000))
    })

    const offTypingStop = socket.on<{ userId: string }>('user:typing:stop', ({ userId }) => {
      setTypingUsers((prev) => { const s = new Set(prev); s.delete(userId); return s })
      clearTimeout(typingTimers.current.get(userId))
      typingTimers.current.delete(userId)
    })

    const offDeleted = socket.on<{ messageId: string }>('message:deleted', ({ messageId }) => {
      setMessages((prev) => prev.filter((m) => m._id !== messageId))
    })

    const offEdited = socket.on<{ messageId: string; content: string }>('message:edited', ({ messageId, content }) => {
      setMessages((prev) =>
        prev.map((m) => (m._id === messageId ? { ...m, content } : m))
      )
    })

    return () => {
      socket.emit('chat:leave', { chatId })
      setJoined(false)
      offNew(); offTypingStart(); offTypingStop(); offDeleted(); offEdited()
      typingTimers.current.forEach(clearTimeout)
      typingTimers.current.clear()
    }
  }, [chatId, projectId, socket.connected])

  const sendMessage = useCallback((content: string, type: ChatMessage['type'] = 'text') => {
    socket.emit('message:send', { chatId, projectId, type, content, replyTo: null, mentions: [] })
  }, [chatId, projectId])

  const startTyping = useCallback(() => {
    socket.emit('chat:typing:start', { chatId })
  }, [chatId])

  const stopTyping = useCallback(() => {
    socket.emit('chat:typing:stop', { chatId })
  }, [chatId])

  const reactToMessage = useCallback((messageId: string, emoji: string) => {
    socket.emit('message:react', { chatId, projectId, messageId, emoji })
  }, [chatId, projectId])

  return {
    messages, typingUsers, joined,
    sendMessage, startTyping, stopTyping, reactToMessage,
    currentUserId: session?.user?.id,
  }
}
```

## 2. REST messaging API (email, SMS, push, device registration)

Mounted at `/api/messaging`, all routes scoped under `/api/messaging/projects/:projectId/...` unless noted. This is a separate surface from chat above, use it for transactional or bulk sends (welcome emails, OTP SMS, marketing push) that are not tied to a chat room.

| Operation | Method + path |
|-----------|---------------|
| Send an email | `POST /projects/:projectId/messaging/email` |
| Read/update the SMS provider config | `GET` / `PATCH /projects/:projectId/messaging/sms-provider` |
| Send an SMS | `POST /projects/:projectId/messaging/sms` |
| Read/update the push config | `GET` / `PATCH /projects/:projectId/messaging/push-config` |
| Read/update the web push config | `GET` / `PATCH /projects/:projectId/messaging/web-push-config` |
| Get the web push public key | `GET /projects/:projectId/messaging/web-push/public-key` |
| Manage web push subscriptions | `POST` / `GET` / `DELETE /projects/:projectId/messaging/web-push/subscriptions` |
| Register/list/remove a push device | `POST` / `GET` / `DELETE /projects/:projectId/messaging/devices` |
| Send a push notification | `POST /projects/:projectId/messaging/push` |
| Read message history | `GET /projects/:projectId/messaging/history` |
| Read send stats | `GET /projects/:projectId/messaging/stats` |

Providers (which SMS gateway, which push service) are configured per project through the `sms-provider` / `push-config` / `web-push-config` endpoints rather than hardcoded, so an app targets `mudbase`'s messaging API uniformly regardless of which provider the project owner has connected.

The `request()` method on `MudbaseClient` (see `sdk-client.md`) is private to the class, so calls to endpoints it does not already wrap go through a small helper built on the same public accessors (`getBaseUrl()`, `getProjectId()`, `getToken()`) rather than reaching into the client's internals:

```typescript
// src/lib/messaging.ts
import { getMudbaseClient } from '@/lib/mudbase'

async function messagingRequest<T>(method: string, path: string, body?: unknown): Promise<T> {
  const client = getMudbaseClient()
  const token = client.getToken()
  const res = await fetch(
    `${client.getBaseUrl()}/api/messaging/projects/${client.getProjectId()}/messaging${path}`,
    {
      method,
      headers: {
        'Content-Type': 'application/json',
        ...(token ? { Authorization: `Bearer ${token}` } : {}),
      },
      body: body !== undefined ? JSON.stringify(body) : undefined,
    }
  )
  if (!res.ok) {
    const err = await res.json().catch(() => ({ message: res.statusText }))
    throw new Error(err.message ?? `Messaging request failed (${res.status})`)
  }
  return res.json() as Promise<T>
}

interface SendEmailPayload {
  to: string | string[]
  subject: string
  html?: string
  text?: string
  templateId?: string
  templateData?: Record<string, unknown>
}

export function sendEmail(payload: SendEmailPayload): Promise<{ messageId: string }> {
  return messagingRequest('POST', '/email', payload)
}

interface SendSmsPayload {
  to: string
  body: string
}

export function sendSms(payload: SendSmsPayload): Promise<{ messageId: string }> {
  return messagingRequest('POST', '/sms', payload)
}

interface SendPushPayload {
  deviceTokens?: string[]
  userId?: string
  title: string
  body: string
  data?: Record<string, unknown>
}

export function sendPush(payload: SendPushPayload): Promise<{ sent: number; failed: number }> {
  return messagingRequest('POST', '/push', payload)
}
```

The `sdk-client.md` typed client only wires up `sendPushNotification` directly on `MudbaseClient`; the rest of this REST surface (email, SMS, web push subscriptions, device registration, history, stats) is intentionally left as the small helper above rather than baked into the shared client, since most apps only need one or two of these endpoints and the payload shapes are project-config-dependent.

### Plan messaging quotas

Combined email + push sends count against one monthly quota per plan (SMS and web push are metered separately by the underlying provider you connect, not by this quota):

| Plan | Messages / month |
|------|-------------------|
| Free | 100 |
| Basic | 2,500 |
| Starter | 20,000 |
| Growth | 50,000 |
| Scale | 100,000 |
| Enterprise | 500,000 |

Exceeding the quota fails the send with a plan-limit error rather than silently queuing; surface this distinctly in your app (e.g. "messaging quota reached, upgrade or wait for next cycle") rather than retrying blindly.

## See also

- `realtime.md`, the shared Socket.IO client (`getMudbaseSocket`) chat rides on top of
- `sdk-client.md`, `sendPushNotification` and the base `request()` method used for the raw REST calls above
