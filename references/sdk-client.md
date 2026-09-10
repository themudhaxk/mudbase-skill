# SDK Client, Environment Variables, Security Checklist

## Installation

Mudbase has no official npm SDK yet. Use the REST API directly with a typed client.

```bash
npm install axios
# or use native fetch, no external dependency needed
```

## `src/lib/mudbase.ts`, full typed client

```typescript
// src/lib/mudbase.ts

const MUDBASE_BASE_URL = 'https://cloud.mudbase.dev'

// --- Types ---------------------------------------------------------------

export interface MudbaseConfig {
  baseUrl?: string
  projectId: string
}

export interface RegisterParams {
  email: string
  password: string
  firstName: string
  lastName: string
  role?: string
  metadata?: Record<string, unknown>
}

export interface LoginParams {
  email: string
  password: string
}

export interface ConvertParams {
  email: string
  password: string
  firstName: string
  lastName: string
}

export interface AuthResponse {
  message: string
  /** Absent when `requireVerification` is true, the account exists but cannot sign in yet. */
  token?: string
  refreshToken?: string
  /** Access-token lifetime in seconds. */
  expiresIn?: number
  user: UserObject
  requireVerification?: boolean
}

export interface UserObject {
  id: string
  email: string
  firstName: string
  lastName: string
  /** System role (project end-user vs org member). Application roles live in `customRole`. */
  role?: string
  customRole?: string | null
  emailVerified: boolean
  twoFactorEnabled?: boolean
  isAnonymous?: boolean
  metadata?: Record<string, unknown>
}

export interface SessionResponse {
  user: UserObject
  token: string
}

// Collection documents are schemaless at the type level, a dynamic Mongoose model is
// built per collection, so only `_id` and the timestamps are guaranteed.
export interface Document {
  _id: string
  createdAt: string
  updatedAt: string
  [key: string]: unknown
}

export interface PaginationMeta {
  page: number
  limit: number
  total: number
  totalPages: number
  hasMore: boolean
}

export interface ListResponse<T> {
  data: T[]
  pagination: PaginationMeta
}

/**
 * Mudbase takes a raw Mongo-style query object as a single JSON-stringified `filter`
 * param, server-side sanitized (`utils/querySanitizer.js`) before it reaches the driver.
 * There is no operator-array DSL and no free-text `search` param on the data endpoints.
 */
export type MongoFilter = Record<string, unknown>

export interface QueryParams {
  filter?: MongoFilter
  /** Mongoose sort string, e.g. `"-createdAt"` or `"status -publishedAt"`. Defaults to `-createdAt`. */
  sort?: string
  /** 1-indexed. Page size is clamped server-side to max 100 (default 20). */
  page?: number
  limit?: number
  /** Projection allow-list, sent as `?fields=a,b,c`. */
  fields?: string[]
}

export interface FileObject {
  id: string
  originalName: string
  fileName: string
  size: number
  mimetype: string
  url: string
  isPublic: boolean
  uploadedAt: string
}

export interface PresignedUploadResponse {
  /** Object key to send back on confirm, the client never chooses it. */
  key: string
  /** S3-style presigned-POST target. Upload with a multipart POST, not a PUT (see storage.md). */
  url: string
  fields: Record<string, string>
  expiresIn: number
  maxFileUploadBytes: number
}

export interface SignedUrlResponse {
  success: boolean
  signedUrl: string
  expiresAt: string
}

// Enum values come straight from mudbase-backend/models/ApiKey.js, anything else is a 400.
export type ApiKeyResource =
  | 'auth'
  | 'database'
  | 'storage'
  | 'functions'
  | 'realtime'
  | 'messaging'
  | 'wallet'
  | 'transactions'
  | 'addons'
  | 'kyc'
  | 'payments'

export type ApiKeyAction = 'create' | 'read' | 'update' | 'delete'

export interface ApiKeyPermission {
  resource: ApiKeyResource
  actions: ApiKeyAction[]
}

export interface ApiKeyRateLimit {
  requests: number
  /** Window in seconds. */
  window: number
}

export interface ApiKeyParams {
  name: string
  /** Required and non-empty, mudbase rejects an implicit "all permissions" key. */
  permissions: ApiKeyPermission[]
  /** Defaults to the client's own projectId when omitted. */
  projectId?: string
  rateLimit?: ApiKeyRateLimit
  expiresAt?: Date
}

export interface ApiKey {
  _id: string
  name: string
  keyPrefix: string
  keyPreview: string
  permissions: ApiKeyPermission[]
  rateLimit: ApiKeyRateLimit
  isActive: boolean
  expiresAt: string | null
  createdAt: string
}

/** Only ever populated on the create/regenerate response, the raw key is never stored. */
export interface CreatedApiKey extends ApiKey {
  key: string
}

export interface PushParams {
  userId: string
  title: string
  body: string
  data?: Record<string, unknown>
}

export type FunctionExecutionStatus = 'queued' | 'running' | 'success' | 'failed'

/** 202 response from an invoke, the run has not started yet. */
export interface FunctionInvocation {
  executionId: string
  status: 'queued'
}

export interface FunctionExecutionLogs {
  stdout: string
  stderr: string
  truncated: boolean
  bytes: number
}

export interface FunctionExecution<TResult = unknown> {
  executionId: string
  status: FunctionExecutionStatus
  durationMs: number | null
  error: string | null
  errorClass: string | null
  logs: FunctionExecutionLogs | null
  result: TResult | null
  createdAt: string
  startedAt: string | null
  completedAt: string | null
}

export interface TwoFASetupResponse {
  secret: string
  /** Ready-to-render data: URI PNG. No QR library needed client-side. */
  qrCode: string
  manualEntryKey: string
}

export interface MultiRoleRole {
  slug: string
  name: string
  description?: string
  isEnabled: boolean
  isCustom: boolean
  signupEndpoint: string
  requiresApproval: boolean
  requiresPayment: boolean
  requiresKYC: boolean
  privilegeLevel: number
}

export interface MultiRoleConfig {
  isEnabled: boolean
  defaultRole: string | null
  settings: {
    allowMultipleRoles: boolean
    requireRoleSelection: boolean
    autoAssignDefault: boolean
    dataOwnerField: string
  }
  roles: MultiRoleRole[]
}

// --- MudbaseClient ---------------------------------------------------------

export class MudbaseClient {
  private baseUrl: string
  private projectId: string
  private token: string | null = null
  private realtimeListeners: Map<string, Set<(data: unknown) => void>> = new Map()
  private ws: WebSocket | null = null
  private wsReconnectAttempts = 0
  private readonly maxWsReconnects = 5

  constructor(config: MudbaseConfig) {
    this.baseUrl = config.baseUrl ?? MUDBASE_BASE_URL
    this.projectId = config.projectId
    // Restore token from storage on init
    if (typeof window !== 'undefined') {
      this.token = sessionStorage.getItem('mudbase_token') ?? localStorage.getItem('mudbase_token')
    }
  }

  // --- Token Management ------------------------------------------------------

  setToken(token: string, persist = true): void {
    this.token = token
    if (typeof window !== 'undefined') {
      if (persist) {
        localStorage.setItem('mudbase_token', token)
      } else {
        sessionStorage.setItem('mudbase_token', token)
      }
    }
  }

  clearToken(): void {
    this.token = null
    if (typeof window !== 'undefined') {
      localStorage.removeItem('mudbase_token')
      sessionStorage.removeItem('mudbase_token')
    }
  }

  getToken(): string | null {
    return this.token
  }

  getProjectId(): string {
    return this.projectId
  }

  getBaseUrl(): string {
    return this.baseUrl
  }

  // --- HTTP Helper -------------------------------------------------------

  private async request<T>(
    method: string,
    path: string,
    body?: unknown,
    options: { auth?: boolean; formData?: FormData; headers?: Record<string, string> } = {}
  ): Promise<T> {
    const headers: Record<string, string> = { ...options.headers }

    if (!(options.formData)) {
      headers['Content-Type'] = 'application/json'
    }

    if (options.auth !== false && this.token) {
      headers['Authorization'] = `Bearer ${this.token}`
    }

    const res = await fetch(`${this.baseUrl}${path}`, {
      method,
      headers,
      body: options.formData
        ? options.formData
        : body !== undefined
        ? JSON.stringify(body)
        : undefined,
    })

    if (!res.ok) {
      const error = (await res.json().catch(() => ({ message: res.statusText }))) as {
        message?: string
      }
      throw new MudbaseError(error.message ?? 'Request failed', res.status, error)
    }

    const text = await res.text()
    return text ? JSON.parse(text) : ({} as T)
  }

  // --- Auth ----------------------------------------------------------------

  async register(params: RegisterParams): Promise<AuthResponse> {
    const role = params.role ?? 'user'
    const res = await this.request<AuthResponse>(
      'POST',
      `/api/auth/local/signup/${role}`,
      { ...params, projectId: this.projectId },
      { auth: false }
    )
    if (res.token) {
      this.setToken(res.token)
    }
    return res
  }

  async login(params: LoginParams): Promise<AuthResponse> {
    const res = await this.request<AuthResponse>(
      'POST',
      '/api/auth/local/login',
      { ...params, projectId: this.projectId },
      { auth: false }
    )
    if (res.token) {
      this.setToken(res.token)
    }
    return res
  }

  async logout(): Promise<void> {
    try {
      await this.request<void>('POST', '/api/auth/logout', { projectId: this.projectId })
    } finally {
      this.clearToken()
      this.disconnectWebSocket()
    }
  }

  async getSession(): Promise<SessionResponse> {
    return this.request<SessionResponse>('GET', `/api/auth/session?projectId=${this.projectId}`)
  }

  async sendMagicLink(email: string): Promise<void> {
    await this.request<void>(
      'POST',
      '/api/auth/magic-link/send',
      { email, projectId: this.projectId },
      { auth: false }
    )
  }

  async verifyMagicLink(token: string): Promise<AuthResponse> {
    const res = await this.request<AuthResponse>(
      'POST',
      '/api/auth/magic-link/verify',
      { token, projectId: this.projectId },
      { auth: false }
    )
    if (res.token) {
      this.setToken(res.token)
    }
    return res
  }

  async sendOTP(phone: string): Promise<void> {
    await this.request<void>(
      'POST',
      '/api/auth/otp/send',
      { phone, projectId: this.projectId },
      { auth: false }
    )
  }

  async verifyOTP(phone: string, code: string): Promise<AuthResponse> {
    const res = await this.request<AuthResponse>(
      'POST',
      '/api/auth/otp/verify',
      { phone, code, projectId: this.projectId },
      { auth: false }
    )
    if (res.token) {
      this.setToken(res.token)
    }
    return res
  }

  async loginAnonymous(): Promise<AuthResponse> {
    const res = await this.request<AuthResponse>(
      'POST',
      '/api/auth/anonymous',
      { projectId: this.projectId },
      { auth: false }
    )
    if (res.token) {
      this.setToken(res.token)
    }
    return res
  }

  async convertAnonymous(params: ConvertParams): Promise<AuthResponse> {
    const res = await this.request<AuthResponse>('POST', '/api/auth/anonymous/convert', {
      ...params,
      projectId: this.projectId,
    })
    if (res.token) {
      this.setToken(res.token)
    }
    return res
  }

  /**
   * Project-scoped reset. Because `projectId` is present, mudbase emails a 6-digit OTP
   * (10-minute TTL) rather than a reset-link token, the confirm step below takes that OTP.
   */
  async requestPasswordReset(email: string): Promise<{ message: string }> {
    return this.request<{ message: string }>(
      'POST',
      '/api/auth/password-reset',
      { email, projectId: this.projectId },
      { auth: false }
    )
  }

  async confirmPasswordReset(
    email: string,
    otp: string,
    newPassword: string
  ): Promise<{ message: string }> {
    return this.request<{ message: string }>(
      'POST',
      '/api/auth/password-reset/confirm',
      { email, projectId: this.projectId, otp, newPassword },
      { auth: false }
    )
  }

  // --- 2FA (TOTP enrolment for the signed-in user) --------------------------
  // These live under /api/users and require a valid bearer token. mudbase's login
  // endpoints do NOT gate on 2FA, see auth.md 2.6 for what that means for your flow.

  async setup2FA(): Promise<TwoFASetupResponse> {
    return this.request<TwoFASetupResponse>('POST', '/api/users/2fa/setup')
  }

  async verify2FA(totpCode: string): Promise<{ message: string }> {
    return this.request<{ message: string }>('POST', '/api/users/2fa/verify', {
      token: totpCode,
    })
  }

  async disable2FA(password: string, totpCode: string): Promise<{ message: string }> {
    return this.request<{ message: string }>('POST', '/api/users/2fa/disable', {
      password,
      token: totpCode,
    })
  }

  /**
   * Org-authenticated: requires a dashboard/API-key token with `project:read` on this
   * project. It is NOT callable from an unauthenticated signup page, see auth.md 7.1.
   */
  async getMultiRoleConfig(projectId?: string): Promise<MultiRoleConfig> {
    const pid = projectId ?? this.projectId
    const res = await this.request<{ success: boolean; data: MultiRoleConfig }>(
      'GET',
      `/api/projects/${pid}/multi-role`
    )
    return res.data
  }

  // --- Collections (data plane) ----------------------------------------------
  // Path shape: /api/data/projects/:projectId/collections/:collectionId/data[/:documentId]

  private dataPath(collectionId: string, documentId?: string): string {
    const base = `/api/data/projects/${this.projectId}/collections/${collectionId}/data`
    return documentId ? `${base}/${documentId}` : base
  }

  async getDocuments<T = Document>(
    collectionId: string,
    query?: QueryParams
  ): Promise<ListResponse<T>> {
    const params = new URLSearchParams()

    if (query?.filter && Object.keys(query.filter).length > 0) {
      params.set('filter', JSON.stringify(query.filter))
    }
    if (query?.sort) params.set('sort', query.sort)
    if (query?.page !== undefined) params.set('page', String(query.page))
    if (query?.limit !== undefined) params.set('limit', String(query.limit))
    if (query?.fields?.length) params.set('fields', query.fields.join(','))

    const qs = params.toString()
    return this.request<ListResponse<T>>(
      'GET',
      qs ? `${this.dataPath(collectionId)}?${qs}` : this.dataPath(collectionId)
    )
  }

  async getDocument<T = Document>(collectionId: string, documentId: string): Promise<T> {
    const res = await this.request<{ data: T }>('GET', this.dataPath(collectionId, documentId))
    return res.data
  }

  async createDocument<T = Document>(
    collectionId: string,
    data: Record<string, unknown>
  ): Promise<T> {
    const res = await this.request<{ message: string; data: T }>(
      'POST',
      this.dataPath(collectionId),
      data
    )
    return res.data
  }

  async updateDocument<T = Document>(
    collectionId: string,
    documentId: string,
    data: Record<string, unknown>
  ): Promise<T> {
    const res = await this.request<{ message: string; data: T }>(
      'PATCH',
      this.dataPath(collectionId, documentId),
      data
    )
    return res.data
  }

  async deleteDocument(collectionId: string, documentId: string): Promise<void> {
    await this.request<{ message: string }>('DELETE', this.dataPath(collectionId, documentId))
  }

  // --- Files -----------------------------------------------------------------
  // Bucket routes are mounted at /api/bucket; the flat file routes at /api/files.

  /** Multipart field name is `files` and the endpoint accepts up to 10 per request. */
  async uploadFiles(bucketId: string, files: File[], isPublic?: boolean): Promise<FileObject[]> {
    const formData = new FormData()
    for (const file of files) {
      formData.append('files', file)
    }
    if (isPublic !== undefined) formData.append('isPublic', String(isPublic))

    const res = await this.request<{ success: boolean; files: FileObject[] }>(
      'POST',
      `/api/bucket/projects/${this.projectId}/buckets/${bucketId}/files`,
      undefined,
      { formData }
    )
    return res.files
  }

  async uploadFile(bucketId: string, file: File, isPublic?: boolean): Promise<FileObject> {
    const [uploaded] = await this.uploadFiles(bucketId, [file], isPublic)
    if (!uploaded) {
      throw new MudbaseError('Upload returned no file record', 500)
    }
    return uploaded
  }

  /** S3-style presigned POST. `bucket` here is the flat bucket *name*, not a bucket ObjectId. */
  async getPresignedUploadUrl(params: {
    bucket?: string
    originalName: string
    contentType: string
    isPublic?: boolean
  }): Promise<PresignedUploadResponse> {
    return this.request<PresignedUploadResponse>('POST', '/api/files/upload/presigned', {
      projectId: this.projectId,
      bucket: params.bucket ?? 'default',
      originalName: params.originalName,
      contentType: params.contentType,
      isPublic: params.isPublic ?? false,
    })
  }

  async confirmUpload(params: {
    key: string
    originalName: string
    contentType: string
    size: number
    bucket?: string
    isPublic?: boolean
  }): Promise<{ message: string; fileId: string }> {
    return this.request<{ message: string; fileId: string }>('POST', '/api/files/upload/confirm', {
      key: params.key,
      projectId: this.projectId,
      originalName: params.originalName,
      contentType: params.contentType,
      size: params.size,
      bucket: params.bucket ?? 'default',
      isPublic: params.isPublic ?? false,
    })
  }

  async getSignedUrl(bucketId: string, fileId: string, expiresIn = 3600): Promise<string> {
    const res = await this.request<SignedUrlResponse>(
      'POST',
      `/api/bucket/projects/${this.projectId}/buckets/${bucketId}/files/${fileId}/signed-url`,
      { expiresIn }
    )
    return res.signedUrl
  }

  async deleteFile(bucketId: string, fileId: string): Promise<void> {
    await this.request<{ success: boolean; message: string }>(
      'DELETE',
      `/api/bucket/projects/${this.projectId}/buckets/${bucketId}/files/${fileId}`
    )
  }

  // --- Functions (async execution) --------------------------------------------

  /** Enqueues the run and returns 202 immediately, poll or subscribe for the result. */
  async invokeFunction(
    functionId: string,
    payload: Record<string, unknown>
  ): Promise<FunctionInvocation> {
    const res = await this.request<{ success: boolean; data: FunctionInvocation }>(
      'POST',
      `/api/functions/projects/${this.projectId}/functions/${functionId}/execute`,
      { payload }
    )
    return res.data
  }

  async getFunctionExecution<TResult = unknown>(
    functionId: string,
    executionId: string
  ): Promise<FunctionExecution<TResult>> {
    const res = await this.request<{ success: boolean; data: FunctionExecution<TResult> }>(
      'GET',
      `/api/functions/projects/${this.projectId}/functions/${functionId}/executions/${executionId}`
    )
    return res.data
  }

  // --- API Keys ----------------------------------------------------------------
  // Flat routes, org-scoped: /api/api-keys (aliased at /api/tokens). The project is
  // named in the body/path segment, not in the mount path.

  async createApiKey(params: ApiKeyParams): Promise<CreatedApiKey> {
    const res = await this.request<{ message: string; apiKey: CreatedApiKey }>(
      'POST',
      '/api/api-keys',
      {
        name: params.name,
        projectId: params.projectId ?? this.projectId,
        permissions: params.permissions,
        rateLimit: params.rateLimit,
        expiresAt: params.expiresAt?.toISOString(),
      }
    )
    return res.apiKey
  }

  async listApiKeys(page = 1, limit = 20): Promise<{ apiKeys: ApiKey[]; pagination: PaginationMeta }> {
    return this.request<{ apiKeys: ApiKey[]; pagination: PaginationMeta }>(
      'GET',
      `/api/api-keys/project/${this.projectId}?page=${page}&limit=${limit}`
    )
  }

  /** Soft revoke, keeps the audit trail. Use deleteApiKey to remove the record entirely. */
  async revokeApiKey(keyId: string): Promise<ApiKey> {
    const res = await this.request<{ message: string; apiKey: ApiKey }>(
      'PATCH',
      `/api/api-keys/${keyId}`,
      { isActive: false }
    )
    return res.apiKey
  }

  async deleteApiKey(keyId: string): Promise<void> {
    await this.request<{ message: string }>('DELETE', `/api/api-keys/${keyId}`)
  }

  async rotateApiKey(keyId: string): Promise<{ key: string; keyPreview: string }> {
    const res = await this.request<{ message: string; key: string; keyPreview: string }>(
      'POST',
      `/api/api-keys/${keyId}/regenerate`
    )
    return { key: res.key, keyPreview: res.keyPreview }
  }

  // --- Messaging ---------------------------------------------------------------
  // A narrow slice is wired into the client below (push). See messaging.md for the
  // full REST surface (email, SMS, web push, device registration, history, stats).

  async sendPushNotification(params: PushParams): Promise<void> {
    await this.request<void>('POST', `/api/messaging/projects/${this.projectId}/messaging/push`, params)
  }

  // --- Real-time WebSocket -----------------------------------------------------
  // This raw WebSocket path is superseded by the Socket.IO surface in realtime.md
  // for every real-time feature mudbase actually ships (collections, chat, presence,
  // wallets, custom channels). Kept here only because `subscribe()` below is the
  // helper other examples in this skill build on for a minimal generic pub/sub shape.

  connectWebSocket(): void {
    if (this.ws?.readyState === WebSocket.OPEN) return

    const wsUrl = this.baseUrl.replace(/^http/, 'ws')
    const ws = new WebSocket(`${wsUrl}/ws?projectId=${this.projectId}&token=${this.token ?? ''}`)
    this.ws = ws

    ws.onopen = () => {
      this.wsReconnectAttempts = 0
      // Send auth handshake
      ws.send(JSON.stringify({ type: 'auth', token: this.token, projectId: this.projectId }))
    }

    ws.onmessage = (event: MessageEvent) => {
      try {
        const message = JSON.parse(event.data as string) as {
          channel: string
          event: string
          data: unknown
        }
        const key = `${message.channel}:${message.event}`
        const listeners = this.realtimeListeners.get(key)
        if (listeners) {
          listeners.forEach((cb) => cb(message.data))
        }
      } catch {
        // ignore malformed messages
      }
    }

    ws.onclose = () => {
      this.scheduleWsReconnect()
    }

    ws.onerror = () => {
      ws.close()
    }
  }

  private scheduleWsReconnect(): void {
    if (this.wsReconnectAttempts >= this.maxWsReconnects) return
    const delay = Math.min(1000 * 2 ** this.wsReconnectAttempts, 30000)
    this.wsReconnectAttempts++
    setTimeout(() => this.connectWebSocket(), delay)
  }

  disconnectWebSocket(): void {
    this.ws?.close()
    this.ws = null
    this.realtimeListeners.clear()
  }

  subscribe(channel: string, event: string, callback: (data: unknown) => void): () => void {
    const key = `${channel}:${event}`
    let listeners = this.realtimeListeners.get(key)
    if (!listeners) {
      listeners = new Set()
      this.realtimeListeners.set(key, listeners)
    }
    listeners.add(callback)

    if (!this.ws || this.ws.readyState !== WebSocket.OPEN) {
      this.connectWebSocket()
    } else {
      this.ws.send(JSON.stringify({ type: 'subscribe', channel, event }))
    }

    return () => {
      const listeners = this.realtimeListeners.get(key)
      if (listeners) {
        listeners.delete(callback)
        if (listeners.size === 0) {
          this.realtimeListeners.delete(key)
          this.ws?.send(JSON.stringify({ type: 'unsubscribe', channel, event }))
        }
      }
    }
  }

  unsubscribeAll(): void {
    this.realtimeListeners.clear()
    this.ws?.send(JSON.stringify({ type: 'unsubscribe_all' }))
  }
}

// --- MudbaseError ------------------------------------------------------------

export class MudbaseError extends Error {
  constructor(
    message: string,
    public statusCode: number,
    public details?: unknown
  ) {
    super(message)
    this.name = 'MudbaseError'
  }
}

// --- Singleton -----------------------------------------------------------------

let _client: MudbaseClient | null = null

export function getMudbaseClient(): MudbaseClient {
  if (!_client) {
    throw new Error('Mudbase client not initialized. Call initMudbase() first.')
  }
  return _client
}

export function initMudbase(config: MudbaseConfig): MudbaseClient {
  _client = new MudbaseClient(config)
  return _client
}
```

## `src/lib/mudbase-provider.tsx`, React context and hook

```typescript
// src/lib/mudbase-provider.tsx
'use client'

import React, { createContext, useContext, useEffect, useState } from 'react'
import { MudbaseClient, initMudbase, type MudbaseConfig, type SessionResponse } from './mudbase'

interface MudbaseContextValue {
  client: MudbaseClient
  session: SessionResponse | null
  loading: boolean
  refreshSession: () => Promise<void>
}

const MudbaseContext = createContext<MudbaseContextValue | null>(null)

export function MudbaseProvider({
  children,
  config,
}: {
  children: React.ReactNode
  config: MudbaseConfig
}) {
  const [client] = useState(() => initMudbase(config))
  const [session, setSession] = useState<SessionResponse | null>(null)
  const [loading, setLoading] = useState(true)

  const refreshSession = async () => {
    try {
      const s = await client.getSession()
      setSession(s)
    } catch {
      setSession(null)
      client.clearToken()
    }
  }

  useEffect(() => {
    refreshSession().finally(() => setLoading(false))
  }, [])

  return (
    <MudbaseContext.Provider value={{ client, session, loading, refreshSession }}>
      {children}
    </MudbaseContext.Provider>
  )
}

export function useMudbase(): MudbaseContextValue {
  const ctx = useContext(MudbaseContext)
  if (!ctx) throw new Error('useMudbase must be used inside <MudbaseProvider>')
  return ctx
}
```

## App entry point setup

```typescript
// src/main.tsx (React) or src/app/layout.tsx (Next.js)
import { MudbaseProvider } from '@/lib/mudbase-provider'

const projectId = process.env.NEXT_PUBLIC_MUDBASE_PROJECT_ID
if (!projectId) {
  throw new Error('NEXT_PUBLIC_MUDBASE_PROJECT_ID must be set')
}

const mudbaseConfig = {
  projectId,
  // baseUrl defaults to https://cloud.mudbase.dev
}

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <MudbaseProvider config={mudbaseConfig}>
      {children}
    </MudbaseProvider>
  )
}
```

## Environment Variables

```bash
# .env.local

# mudbase
NEXT_PUBLIC_MUDBASE_PROJECT_ID=your_project_id
NEXT_PUBLIC_MUDBASE_URL=https://cloud.mudbase.dev
# Server-side only (never NEXT_PUBLIC_)
MUDBASE_URL=https://cloud.mudbase.dev
MUDBASE_PROJECT_ID=your_project_id
MUDBASE_API_KEY=ak_...                 # project API key for server-side reads (multi-role config, etc.)
MUDBASE_ADMIN_TOKEN=your_org_owner_bearer_token   # only needed for API-key/billing management calls
MUDBASE_WEBHOOK_SECRET=your_webhook_signing_secret

```

## Security Checklist

- [ ] API keys and service account tokens never in the client bundle, only in server-side env vars
- [ ] **Token storage, pick deliberately, mudbase gives you no cookie option.** Every mudbase auth endpoint returns the bearer token in a JSON body; there is no `Set-Cookie`, so mudbase itself cannot give you an httpOnly session. That leaves three honest choices:
  - **Browser, default:** `sessionStorage`, not `localStorage`. Both are XSS-readable, but `sessionStorage` clears on tab close and is not shared across tabs, which shortens the residual-exposure window. Only reach for `localStorage` if "stay signed in across browser restarts" is a real product requirement, and accept that the token then sits at rest on disk indefinitely.
  - **Browser, higher assurance:** proxy authentication through your own backend. Your Next.js route handler (or equivalent) calls `/api/auth/local/login`, keeps the mudbase bearer token server-side (in your session store), and sets your *own* `httpOnly; Secure; SameSite=Lax` session cookie on the browser. The mudbase token then never reaches client JavaScript at all. This is the only configuration where XSS cannot exfiltrate the credential, and it is the right default for anything handling money, health data, or admin capability.
  - **Mobile (React Native / Expo):** `expo-secure-store` or `react-native-keychain`. Never `AsyncStorage`, it is unencrypted plaintext on disk and readable on a rooted or jailbroken device.
- [ ] Anonymous sessions: always convert before storing sensitive user data (PII, payment info)
- [ ] File buckets: set access policy correctly, private for user uploads, public only for truly public assets
- [ ] Real-time channels: always scope to the authenticated user's own data. Never subscribe to `collection:{id}` for all users, subscribe to `user:{userId}:*` or filter by your own userId
- [ ] OAuth: save the return path to `sessionStorage` before redirecting, validate the token against `getSession()` after callback, never trust a token from the URL alone without server validation
- [ ] Functions: validate every payload field on the function side, never trust client-provided IDs or roles
- [ ] Never expose `projectId` in combination with admin-level API keys, use scoped keys with minimum required permissions
- [ ] Password reset and magic link tokens are one-time use, confirm on the server that the token has not already been consumed. Project-scoped resets are OTP-based (6 digits, 10-minute TTL, `POST /api/auth/password-reset/confirm`), not link-token based
- [ ] 2FA: mudbase's login endpoint does **not** gate on 2FA, it returns a usable token regardless. If your product promises a second factor at login, enforce it in your own backend proxy (see the token-storage item above); do not present `user.twoFactorEnabled` to users as though the server were blocking on it
- [ ] Rate limit auth endpoints on the server; mudbase handles this internally, but add client-side debounce on forms
- [ ] Content Security Policy: if loading files from mudbase storage via signed URLs, add the storage domain to your CSP `img-src` / `media-src` directives
- [ ] SSO (see auth.md): if your project has an SSO connection configured, still keep local auth's rate limits and password-reset flow intact for accounts that are not provisioned through the identity provider

## See also

- `auth.md`, every auth flow including SSO
- `collections-and-data.md`, the data-plane REST contract
- `storage.md`, the two storage surfaces and presigned uploads
- `access-control.md`, multi-role signup and API key permission grants
