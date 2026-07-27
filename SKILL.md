---
name: mudbase
description: Complete integration guide for building client apps on Mudbase (backend-as-a-service) — typed SDK client, all auth flows (local/OAuth/magic-link/OTP/anonymous/2FA), collections CRUD with React Query, file storage, Socket.IO realtime (chat/presence/wallets/calls), serverless functions, webhooks, MCP server for AI agents, add-ons, KYC, payment links, and wallet transaction broadcasting.
---

# Mudbase

Mudbase (`mudbase.dev`) is a backend-as-a-service platform: multi-tenant projects, JSON-schema collections with role-based permissions, file storage, Socket.IO realtime, serverless functions, multi-chain crypto wallets, and a REST API at `cloud.mudbase.dev`. This skill teaches an AI coding agent — or any developer — everything needed to build a client application against an existing Mudbase project: the typed SDK client, every auth flow, collections CRUD, file storage, realtime events, serverless functions, the multi-role system, API key management, environment variables, a security checklist, the full Socket.IO event catalog, webhooks, background job behavior, error handling, rate limiting, the MCP server for AI agents, the add-ons marketplace, identity verification, project sharing, the migration importer, payment links, wallet transaction broadcasting, and the in-app credit balance. Full reference documentation lives at https://docs.mudbase.dev.

---

## 1. mudbase SDK Integration

mudbase has no official npm SDK yet. Use the REST API directly with a typed client.

### Installation

```bash
npm install axios
# or use native fetch — no external dep needed
```

### `src/lib/mudbase.ts` — Full Typed Client

```typescript
// src/lib/mudbase.ts

const MUDBASE_BASE_URL = 'https://cloud.mudbase.dev'

// ─── Types ────────────────────────────────────────────────────────────────────

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
  /** Absent when `requireVerification` is true — the account exists but cannot sign in yet. */
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

// Collection documents are schemaless at the type level — a dynamic Mongoose model is
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
 * mudbase takes a raw Mongo-style query object as a single JSON-stringified `filter`
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
  /** Object key to send back on confirm — the client never chooses it. */
  key: string
  /** S3 presigned-POST target. Upload with a multipart POST, not a PUT. */
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

// Enum values come straight from mudbase-backend/models/ApiKey.js — anything else is a 400.
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
  /** Required and non-empty — mudbase rejects an implicit "all permissions" key. */
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

/** Only ever populated on the create/regenerate response — the raw key is never stored. */
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

/** 202 response from an invoke — the run has not started yet. */
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

// ─── MudbaseClient ────────────────────────────────────────────────────────────

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

  // ─── Token Management ──────────────────────────────────────────────────────

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

  // ─── HTTP Helper ───────────────────────────────────────────────────────────

  private async request<T>(
    method: string,
    path: string,
    body?: unknown,
    options: { auth?: boolean; formData?: FormData } = {}
  ): Promise<T> {
    const headers: Record<string, string> = {}

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
      const error = await res.json().catch(() => ({ message: res.statusText }))
      throw new MudbaseError(error.message ?? 'Request failed', res.status, error)
    }

    const text = await res.text()
    return text ? JSON.parse(text) : ({} as T)
  }

  // ─── Auth ──────────────────────────────────────────────────────────────────

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
   * (10-minute TTL) rather than a reset-link token — the confirm step below takes that OTP.
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

  // ─── 2FA (TOTP enrolment for the signed-in user) ────────────────────────────
  // These live under /api/users and require a valid bearer token. mudbase's login
  // endpoints do NOT gate on 2FA — see §2.6 for what that means for your flow.

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
   * project. It is NOT callable from an unauthenticated signup page — see §7.1.
   */
  async getMultiRoleConfig(projectId?: string): Promise<MultiRoleConfig> {
    const pid = projectId ?? this.projectId
    const res = await this.request<{ success: boolean; data: MultiRoleConfig }>(
      'GET',
      `/api/projects/${pid}/multi-role`
    )
    return res.data
  }

  // ─── Collections (data plane) ───────────────────────────────────────────────
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

  // ─── Files ─────────────────────────────────────────────────────────────────
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

  /** S3 presigned POST. `bucket` here is the flat bucket *name*, not a bucket ObjectId. */
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

  // ─── Functions (async execution) ────────────────────────────────────────────

  /** Enqueues the run and returns 202 immediately — poll or subscribe for the result. */
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

  // ─── API Keys ──────────────────────────────────────────────────────────────
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

  /** Soft revoke — keeps the audit trail. Use deleteApiKey to remove the record entirely. */
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

  // ─── Messaging ─────────────────────────────────────────────────────────────

  async sendPushNotification(params: PushParams): Promise<void> {
    await this.request<void>('POST', `/api/messaging/projects/${this.projectId}/push`, params)
  }

  // ─── Real-time WebSocket ───────────────────────────────────────────────────

  connectWebSocket(): void {
    if (this.ws?.readyState === WebSocket.OPEN) return

    const wsUrl = this.baseUrl.replace(/^http/, 'ws')
    this.ws = new WebSocket(`${wsUrl}/ws?projectId=${this.projectId}&token=${this.token ?? ''}`)

    this.ws.onopen = () => {
      this.wsReconnectAttempts = 0
      // Send auth handshake
      this.ws!.send(
        JSON.stringify({ type: 'auth', token: this.token, projectId: this.projectId })
      )
    }

    this.ws.onmessage = (event: MessageEvent) => {
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

    this.ws.onclose = () => {
      this.scheduleWsReconnect()
    }

    this.ws.onerror = () => {
      this.ws?.close()
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
    if (!this.realtimeListeners.has(key)) {
      this.realtimeListeners.set(key, new Set())
    }
    this.realtimeListeners.get(key)!.add(callback)

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

// ─── MudbaseError ─────────────────────────────────────────────────────────────

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

// ─── Singleton ────────────────────────────────────────────────────────────────

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

### `src/lib/mudbase-provider.tsx` — React Context + Hook

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

### App entry point setup

```typescript
// src/main.tsx (React) or src/app/layout.tsx (Next.js)
import { MudbaseProvider } from '@/lib/mudbase-provider'

const mudbaseConfig = {
  projectId: process.env.NEXT_PUBLIC_MUDBASE_PROJECT_ID!,
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

---

## 2. Authentication Patterns

### 2.1 Local Auth — `src/hooks/useAuth.ts`

```typescript
// src/hooks/useAuth.ts
'use client'

import { useState, useEffect, useCallback } from 'react'
import { useRouter } from 'next/navigation'
import { useMudbase } from '@/lib/mudbase-provider'
import type { UserObject, MudbaseError } from '@/lib/mudbase'

interface AuthState {
  user: UserObject | null
  loading: boolean
  error: string | null
}

interface UseAuthReturn extends AuthState {
  register: (email: string, password: string, firstName: string, lastName: string, role?: string) => Promise<void>
  login: (email: string, password: string) => Promise<{ twoFactorEnabled: boolean }>
  logout: () => Promise<void>
  clearError: () => void
}

export function useAuth(): UseAuthReturn {
  const { client, session, loading: sessionLoading, refreshSession } = useMudbase()
  const router = useRouter()
  const [loading, setLoading] = useState(false)
  const [error, setError] = useState<string | null>(null)

  const register = useCallback(
    async (email: string, password: string, firstName: string, lastName: string, role = 'user') => {
      setLoading(true)
      setError(null)
      try {
        const res = await client.register({ email, password, firstName, lastName, role })
        if (res.requireVerification) {
          router.push(`/auth/verify-email?email=${encodeURIComponent(email)}`)
          return
        }
        await refreshSession()
        router.push('/dashboard')
      } catch (err) {
        setError((err as MudbaseError).message ?? 'Registration failed')
      } finally {
        setLoading(false)
      }
    },
    [client, router, refreshSession]
  )

  const login = useCallback(
    async (email: string, password: string): Promise<{ twoFactorEnabled: boolean }> => {
      setLoading(true)
      setError(null)
      try {
        const res = await client.login({ email, password })
        // mudbase's login endpoint issues a full bearer token unconditionally; it reports
        // `user.twoFactorEnabled` but does not withhold the session pending a TOTP code.
        // Treat the flag as informational (e.g. re-prompt before privileged actions) —
        // do not build a login gate on it and assume the server is enforcing one.
        await refreshSession()
        router.push('/dashboard')
        return { twoFactorEnabled: res.user.twoFactorEnabled === true }
      } catch (err) {
        setError((err as MudbaseError).message ?? 'Login failed')
        return { twoFactorEnabled: false }
      } finally {
        setLoading(false)
      }
    },
    [client, router, refreshSession]
  )

  const logout = useCallback(async () => {
    setLoading(true)
    try {
      await client.logout()
      router.push('/auth/login')
    } finally {
      setLoading(false)
    }
  }, [client, router])

  return {
    user: session?.user ?? null,
    loading: loading || sessionLoading,
    error,
    register,
    login,
    logout,
    clearError: () => setError(null),
  }
}
```

### 2.2 OAuth (Social Login)

```typescript
// src/lib/oauth.ts

const MUDBASE_URL = process.env.NEXT_PUBLIC_MUDBASE_URL ?? 'https://cloud.mudbase.dev'
const PROJECT_ID = process.env.NEXT_PUBLIC_MUDBASE_PROJECT_ID!

export type OAuthProvider = 'google' | 'github' | 'facebook' | 'microsoft' | 'discord'

export function loginWithOAuth(provider: OAuthProvider): void {
  const redirectUrl = `${window.location.origin}/auth/callback`
  const url = `${MUDBASE_URL}/api/auth/oauth/${provider}/${PROJECT_ID}?redirect_url=${encodeURIComponent(redirectUrl)}`
  window.location.href = url
}
```

```typescript
// src/app/auth/callback/page.tsx
'use client'

import { useEffect, useState } from 'react'
import { useRouter, useSearchParams } from 'next/navigation'
import { getMudbaseClient } from '@/lib/mudbase'

export default function OAuthCallbackPage() {
  const router = useRouter()
  const params = useSearchParams()
  const [error, setError] = useState<string | null>(null)

  useEffect(() => {
    const token = params.get('token')
    const errorMsg = params.get('error')

    if (errorMsg) {
      setError(decodeURIComponent(errorMsg))
      return
    }

    if (!token) {
      setError('No token received from OAuth provider.')
      return
    }

    const client = getMudbaseClient()
    client.setToken(token)

    // Verify session is valid before redirecting
    client
      .getSession()
      .then(() => {
        const returnTo = sessionStorage.getItem('oauth_return_to') ?? '/dashboard'
        sessionStorage.removeItem('oauth_return_to')
        router.replace(returnTo)
      })
      .catch(() => {
        setError('Failed to validate OAuth session.')
        client.clearToken()
      })
  }, [params, router])

  if (error) {
    return (
      <div className="flex min-h-screen items-center justify-center">
        <div className="text-center">
          <h1 className="text-xl font-semibold text-red-600">Authentication Failed</h1>
          <p className="mt-2 text-gray-600">{error}</p>
          <a href="/auth/login" className="mt-4 inline-block text-blue-600 underline">
            Back to Login
          </a>
        </div>
      </div>
    )
  }

  return (
    <div className="flex min-h-screen items-center justify-center">
      <p className="text-gray-500">Completing login...</p>
    </div>
  )
}
```

### 2.3 Magic Link

```typescript
// src/app/auth/magic-link/page.tsx
'use client'

import { useState } from 'react'
import { getMudbaseClient } from '@/lib/mudbase'

export default function MagicLinkPage() {
  const [email, setEmail] = useState('')
  const [sent, setSent] = useState(false)
  const [loading, setLoading] = useState(false)
  const [error, setError] = useState<string | null>(null)

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault()
    setLoading(true)
    setError(null)
    try {
      await getMudbaseClient().sendMagicLink(email)
      setSent(true)
    } catch (err: unknown) {
      setError((err as Error).message ?? 'Failed to send magic link')
    } finally {
      setLoading(false)
    }
  }

  if (sent) {
    return (
      <div className="flex min-h-screen items-center justify-center">
        <div className="text-center">
          <h1 className="text-xl font-semibold">Check your email</h1>
          <p className="mt-2 text-gray-600">
            We sent a magic link to <strong>{email}</strong>. Click it to sign in.
          </p>
        </div>
      </div>
    )
  }

  return (
    <div className="flex min-h-screen items-center justify-center">
      <form onSubmit={handleSubmit} className="w-full max-w-sm space-y-4">
        <h1 className="text-2xl font-bold">Sign in with Magic Link</h1>
        {error && <p className="text-red-600 text-sm">{error}</p>}
        <input
          type="email"
          value={email}
          onChange={(e) => setEmail(e.target.value)}
          placeholder="you@example.com"
          required
          className="w-full rounded-md border px-3 py-2"
        />
        <button
          type="submit"
          disabled={loading}
          className="w-full rounded-md bg-blue-600 py-2 text-white disabled:opacity-50"
        >
          {loading ? 'Sending...' : 'Send Magic Link'}
        </button>
      </form>
    </div>
  )
}
```

```typescript
// src/app/auth/magic-verify/page.tsx
'use client'

import { useEffect, useState } from 'react'
import { useRouter, useSearchParams } from 'next/navigation'
import { getMudbaseClient } from '@/lib/mudbase'

export default function MagicVerifyPage() {
  const router = useRouter()
  const params = useSearchParams()
  const [error, setError] = useState<string | null>(null)

  useEffect(() => {
    const token = params.get('token')
    if (!token) {
      setError('Invalid magic link — missing token.')
      return
    }

    getMudbaseClient()
      .verifyMagicLink(token)
      .then(() => router.replace('/dashboard'))
      .catch((err: Error) => setError(err.message ?? 'Magic link verification failed.'))
  }, [params, router])

  if (error) {
    return (
      <div className="flex min-h-screen items-center justify-center">
        <div className="text-center">
          <h1 className="text-xl font-semibold text-red-600">Invalid Link</h1>
          <p className="mt-2 text-gray-600">{error}</p>
          <a href="/auth/magic-link" className="mt-4 inline-block text-blue-600 underline">
            Request new link
          </a>
        </div>
      </div>
    )
  }

  return (
    <div className="flex min-h-screen items-center justify-center">
      <p className="text-gray-500">Verifying magic link...</p>
    </div>
  )
}
```

### 2.4 OTP (Phone Auth)

```typescript
// src/components/auth/OTPFlow.tsx
'use client'

import { useState, useRef, KeyboardEvent } from 'react'
import { useRouter } from 'next/navigation'
import { getMudbaseClient } from '@/lib/mudbase'
import { useMudbase } from '@/lib/mudbase-provider'

export function OTPFlow() {
  const router = useRouter()
  const { refreshSession } = useMudbase()
  const [step, setStep] = useState<'phone' | 'verify'>('phone')
  const [phone, setPhone] = useState('')
  const [digits, setDigits] = useState<string[]>(Array(6).fill(''))
  const [loading, setLoading] = useState(false)
  const [error, setError] = useState<string | null>(null)
  const inputRefs = useRef<(HTMLInputElement | null)[]>([])

  const handleSendOTP = async (e: React.FormEvent) => {
    e.preventDefault()
    setLoading(true)
    setError(null)
    try {
      await getMudbaseClient().sendOTP(phone)
      setStep('verify')
    } catch (err: unknown) {
      setError((err as Error).message ?? 'Failed to send OTP')
    } finally {
      setLoading(false)
    }
  }

  const handleDigitChange = (index: number, value: string) => {
    if (!/^\d?$/.test(value)) return
    const next = [...digits]
    next[index] = value
    setDigits(next)
    if (value && index < 5) {
      inputRefs.current[index + 1]?.focus()
    }
  }

  const handleKeyDown = (index: number, e: KeyboardEvent<HTMLInputElement>) => {
    if (e.key === 'Backspace' && !digits[index] && index > 0) {
      inputRefs.current[index - 1]?.focus()
    }
  }

  const handleVerify = async (e: React.FormEvent) => {
    e.preventDefault()
    const code = digits.join('')
    if (code.length < 6) {
      setError('Enter the full 6-digit code.')
      return
    }
    setLoading(true)
    setError(null)
    try {
      await getMudbaseClient().verifyOTP(phone, code)
      await refreshSession()
      router.push('/dashboard')
    } catch (err: unknown) {
      setError((err as Error).message ?? 'Invalid OTP')
      setDigits(Array(6).fill(''))
      inputRefs.current[0]?.focus()
    } finally {
      setLoading(false)
    }
  }

  if (step === 'phone') {
    return (
      <form onSubmit={handleSendOTP} className="space-y-4">
        <h2 className="text-xl font-semibold">Enter your phone number</h2>
        {error && <p className="text-red-600 text-sm">{error}</p>}
        <input
          type="tel"
          value={phone}
          onChange={(e) => setPhone(e.target.value)}
          placeholder="+2348012345678"
          required
          className="w-full rounded-md border px-3 py-2"
        />
        <button
          type="submit"
          disabled={loading}
          className="w-full rounded-md bg-blue-600 py-2 text-white disabled:opacity-50"
        >
          {loading ? 'Sending...' : 'Send OTP'}
        </button>
      </form>
    )
  }

  return (
    <form onSubmit={handleVerify} className="space-y-4">
      <h2 className="text-xl font-semibold">Enter verification code</h2>
      <p className="text-sm text-gray-600">Sent to {phone}</p>
      {error && <p className="text-red-600 text-sm">{error}</p>}
      <div className="flex gap-2">
        {digits.map((d, i) => (
          <input
            key={i}
            ref={(el) => { inputRefs.current[i] = el }}
            type="text"
            inputMode="numeric"
            maxLength={1}
            value={d}
            onChange={(e) => handleDigitChange(i, e.target.value)}
            onKeyDown={(e) => handleKeyDown(i, e)}
            className="h-12 w-12 rounded-md border text-center text-xl font-bold"
          />
        ))}
      </div>
      <button
        type="submit"
        disabled={loading || digits.join('').length < 6}
        className="w-full rounded-md bg-blue-600 py-2 text-white disabled:opacity-50"
      >
        {loading ? 'Verifying...' : 'Verify'}
      </button>
      <button
        type="button"
        onClick={() => setStep('phone')}
        className="text-sm text-blue-600 underline"
      >
        Wrong number?
      </button>
    </form>
  )
}
```

### 2.5 Anonymous Auth + Convert

```typescript
// src/hooks/useAnonymousAuth.ts
import { useCallback, useState } from 'react'
import { useMudbase } from '@/lib/mudbase-provider'
import type { ConvertParams } from '@/lib/mudbase'

export function useAnonymousAuth() {
  const { client, refreshSession } = useMudbase()
  const [loading, setLoading] = useState(false)
  const [error, setError] = useState<string | null>(null)

  const startAnonymousSession = useCallback(async () => {
    setLoading(true)
    setError(null)
    try {
      await client.loginAnonymous()
      await refreshSession()
    } catch (err: unknown) {
      setError((err as Error).message ?? 'Failed to start anonymous session')
    } finally {
      setLoading(false)
    }
  }, [client, refreshSession])

  const convertToAccount = useCallback(
    async (params: ConvertParams) => {
      setLoading(true)
      setError(null)
      try {
        await client.convertAnonymous(params)
        await refreshSession()
        return true
      } catch (err: unknown) {
        setError((err as Error).message ?? 'Failed to convert account')
        return false
      } finally {
        setLoading(false)
      }
    },
    [client, refreshSession]
  )

  return { startAnonymousSession, convertToAccount, loading, error }
}
```

Use cases: guest checkout flow (start anonymous → add to cart → checkout → convert), trial access (start anonymous → use features → prompt to sign up on limit).

### 2.6 Two-Factor Authentication (TOTP)

**Read this before designing the flow.** mudbase's TOTP endpoints live at `POST /api/users/2fa/setup`, `/verify`, `/disable` and all three require a valid bearer token (`authRequired`). They are an *enrolment* surface for an already-signed-in user, not a login challenge:

- `POST /api/auth/local/login` returns a full, immediately usable `token` regardless of whether the account has 2FA enabled. It reports the state as `user.twoFactorEnabled` and nothing more — there is no `requires2FA` flag and no partial/pre-auth token.
- `POST /api/users/2fa/verify` takes `{ token: "123456" }` (the TOTP digits — the field is named `token`, not `code`) and, on success, flips `twoFactorEnabled` to true. It does **not** mint or upgrade a session.
- `POST /api/users/2fa/disable` takes `{ password, token }` — both the account password and a live TOTP code.

So do not build a "hold the credentials in memory and complete the challenge on submit" login modal against these endpoints; there is nothing to complete server-side. If your product needs a genuine second factor at login, enforce it in your own backend: proxy `/api/auth/local/login`, hold the returned mudbase token server-side, require a TOTP verification against your own store, and only then issue your session to the browser. Below is the enrolment component, which is what mudbase actually supports.

```typescript
// src/components/auth/TwoFASetup.tsx
'use client'

import { useState } from 'react'
import { useMudbase } from '@/lib/mudbase-provider'

export function TwoFASetup({ onComplete }: { onComplete: () => void }) {
  const { client, refreshSession } = useMudbase()
  const [qrCode, setQrCode] = useState<string | null>(null)
  const [manualEntryKey, setManualEntryKey] = useState<string | null>(null)
  const [code, setCode] = useState('')
  const [loading, setLoading] = useState(false)
  const [error, setError] = useState<string | null>(null)

  const handleSetup = async (): Promise<void> => {
    setLoading(true)
    setError(null)
    try {
      const res = await client.setup2FA()
      setQrCode(res.qrCode)
      setManualEntryKey(res.manualEntryKey)
    } catch (err: unknown) {
      setError(err instanceof Error ? err.message : 'Could not start 2FA setup')
    } finally {
      setLoading(false)
    }
  }

  const handleVerify = async (e: React.FormEvent): Promise<void> => {
    e.preventDefault()
    setLoading(true)
    setError(null)
    try {
      await client.verify2FA(code)
      // twoFactorEnabled is part of the session payload — refresh so the UI reflects it.
      await refreshSession()
      onComplete()
    } catch (err: unknown) {
      setError(err instanceof Error ? err.message : 'Invalid code. Try again.')
      setCode('')
    } finally {
      setLoading(false)
    }
  }

  if (!qrCode) {
    return (
      <div className="space-y-4">
        <h2 className="text-xl font-semibold">Enable Two-Factor Authentication</h2>
        <p className="text-sm text-gray-600">
          Scan the QR code with your authenticator app (Google Authenticator, Authy, 1Password).
        </p>
        <button
          onClick={handleSetup}
          disabled={loading}
          className="rounded-md bg-blue-600 px-4 py-2 text-white disabled:opacity-50"
        >
          {loading ? 'Loading...' : 'Set up 2FA'}
        </button>
        {error && <p className="text-red-600 text-sm">{error}</p>}
      </div>
    )
  }

  return (
    <form onSubmit={handleVerify} className="space-y-4">
      <h2 className="text-xl font-semibold">Scan QR Code</h2>
      {/* `qrCode` is already a data: URI PNG rendered server-side — no QR library, no third-party image host. */}
      <img src={qrCode} alt="Two-factor authentication QR code" className="rounded border" />
      {manualEntryKey && (
        <p className="text-sm text-gray-600">
          Can&apos;t scan? Enter this key manually: <code className="font-mono">{manualEntryKey}</code>
        </p>
      )}
      <p className="text-sm text-gray-600">Enter the 6-digit code from your app to confirm setup.</p>
      {error && <p className="text-red-600 text-sm">{error}</p>}
      <input
        type="text"
        inputMode="numeric"
        maxLength={6}
        value={code}
        onChange={(e) => setCode(e.target.value.replace(/\D/g, ''))}
        placeholder="000000"
        className="w-full rounded-md border px-3 py-2 text-center tracking-widest text-xl"
        required
      />
      <button
        type="submit"
        disabled={loading || code.length < 6}
        className="w-full rounded-md bg-green-600 py-2 text-white disabled:opacity-50"
      >
        {loading ? 'Verifying...' : 'Confirm 2FA'}
      </button>
    </form>
  )
}
```

```typescript
// src/components/auth/TwoFADisable.tsx — requires the account password AND a live code
'use client'

import { useState } from 'react'
import { useMudbase } from '@/lib/mudbase-provider'

export function TwoFADisable({ onDisabled }: { onDisabled: () => void }) {
  const { client, refreshSession } = useMudbase()
  const [password, setPassword] = useState('')
  const [code, setCode] = useState('')
  const [loading, setLoading] = useState(false)
  const [error, setError] = useState<string | null>(null)

  const handleSubmit = async (e: React.FormEvent): Promise<void> => {
    e.preventDefault()
    setLoading(true)
    setError(null)
    try {
      await client.disable2FA(password, code)
      await refreshSession()
      onDisabled()
    } catch (err: unknown) {
      setError(err instanceof Error ? err.message : 'Could not disable 2FA')
    } finally {
      // Never keep the password in state past the request.
      setPassword('')
      setCode('')
      setLoading(false)
    }
  }

  return (
    <form onSubmit={handleSubmit} className="space-y-4">
      <h2 className="text-xl font-semibold">Disable Two-Factor Authentication</h2>
      {error && <p className="text-red-600 text-sm">{error}</p>}
      <input
        type="password"
        autoComplete="current-password"
        value={password}
        onChange={(e) => setPassword(e.target.value)}
        placeholder="Current password"
        className="w-full rounded-md border px-3 py-2"
        required
      />
      <input
        type="text"
        inputMode="numeric"
        maxLength={6}
        value={code}
        onChange={(e) => setCode(e.target.value.replace(/\D/g, ''))}
        placeholder="000000"
        className="w-full rounded-md border px-3 py-2 text-center tracking-widest text-xl"
        required
      />
      <button
        type="submit"
        disabled={loading || code.length < 6 || password.length === 0}
        className="w-full rounded-md bg-red-600 py-2 text-white disabled:opacity-50"
      >
        {loading ? 'Disabling...' : 'Disable 2FA'}
      </button>
    </form>
  )
}
```

---

## 3. Collections & Data

### 3.0 The REST contract

All document CRUD lives under one path family. There is no `/api/collections/...` route.

| Operation | Method + path |
|-----------|---------------|
| List | `GET /api/data/projects/:projectId/collections/:collectionId/data` |
| Read one | `GET /api/data/projects/:projectId/collections/:collectionId/data/:documentId` |
| Create | `POST /api/data/projects/:projectId/collections/:collectionId/data` |
| Update | `PATCH /api/data/projects/:projectId/collections/:collectionId/data/:documentId` |
| Delete | `DELETE /api/data/projects/:projectId/collections/:collectionId/data/:documentId` |

Query parameters on the list endpoint:

| Param | Type | Notes |
|-------|------|-------|
| `filter` | JSON string | A raw Mongo-style query object, server-sanitized before execution. Not an operator array. |
| `sort` | string | Mongoose sort syntax, e.g. `-createdAt`. Defaults to `-createdAt`. |
| `page` | number | 1-indexed. |
| `limit` | number | Default 20, clamped to a maximum of 100. |
| `fields` | CSV | Projection allow-list, e.g. `title,slug,publishedAt`. |

There is **no** `search` parameter here — full-text search is a separate surface (`/api/search`). Use a `filter` with `$regex` for simple substring matching, and note that an unanchored `$regex` cannot use an index.

Response shapes:

```jsonc
// GET .../data
{ "data": [ /* documents */ ], "pagination": { "page": 1, "limit": 20, "total": 57, "totalPages": 3, "hasMore": true } }

// GET .../data/:documentId
{ "data": { "_id": "…", "…": "…" } }

// POST / PATCH .../data
{ "message": "Data created successfully", "data": { "_id": "…", "…": "…" } }

// DELETE .../data/:documentId
{ "message": "Data deleted successfully" }
```

Create and update take the document body **flat** — do not nest it under a `data` key and do not send `projectId` in the body; the project is already in the path. Collection-level permission conditions (`$userId`, `$orgId`, `$userCustomRole`) are auto-populated server-side on create and rejected if the client tries to assign a record to somebody else.

### 3.1 Schema Design Patterns

mudbase collections use JSON-schema style definitions. Common patterns:

```
Collection: user_profiles
Fields: userId (string, unique ref to auth user), avatar (string), bio (string),
        location (string), socialLinks (object), onboardingComplete (boolean)

Collection: posts
Fields: authorId (string), title (string), slug (string, unique), content (string),
        status (enum: draft|published|archived), tags (array<string>),
        publishedAt (datetime), viewCount (number)

Collection: orders
Fields: userId (string), items (array<OrderItem>), subtotal (number),
        currency (string), status (enum: pending|paid|shipped|delivered|cancelled),
        paymentRef (string), shippingAddress (object), metadata (object)
```

### 3.2 React Query Hooks — `src/hooks/useCollection.ts`

```typescript
// src/hooks/useCollection.ts
import {
  useQuery,
  useMutation,
  useQueryClient,
  type UseQueryOptions,
} from '@tanstack/react-query'
import { useMudbase } from '@/lib/mudbase-provider'
import type { Document, ListResponse, QueryParams } from '@/lib/mudbase'

// ─── useDocuments ─────────────────────────────────────────────────────────────

export function useDocuments<T extends Document = Document>(
  collectionId: string,
  query?: QueryParams,
  options?: Omit<UseQueryOptions<ListResponse<T>>, 'queryKey' | 'queryFn'>
) {
  const { client } = useMudbase()
  return useQuery<ListResponse<T>>({
    queryKey: ['collection', collectionId, query],
    queryFn: () => client.getDocuments<T>(collectionId, query),
    staleTime: 1000 * 30,
    ...options,
  })
}

// ─── useDocument ──────────────────────────────────────────────────────────────

export function useDocument<T extends Document = Document>(
  collectionId: string,
  documentId: string | null | undefined,
  options?: Omit<UseQueryOptions<T>, 'queryKey' | 'queryFn'>
) {
  const { client } = useMudbase()
  return useQuery<T>({
    queryKey: ['collection', collectionId, 'doc', documentId],
    queryFn: () => {
      // `enabled` below already gates this, but narrow explicitly rather than asserting.
      if (!documentId) throw new Error('useDocument called without a documentId')
      return client.getDocument<T>(collectionId, documentId)
    },
    enabled: !!documentId,
    staleTime: 1000 * 30,
    ...options,
  })
}

// ─── useCreateDocument ────────────────────────────────────────────────────────

export function useCreateDocument<T extends Document = Document>(collectionId: string) {
  const { client } = useMudbase()
  const queryClient = useQueryClient()

  return useMutation<T, Error, Record<string, unknown>>({
    mutationFn: (data) => client.createDocument<T>(collectionId, data),
    onSuccess: (created) => {
      // Add to list caches
      queryClient.invalidateQueries({ queryKey: ['collection', collectionId] })
      // Prime the individual document cache
      queryClient.setQueryData(['collection', collectionId, 'doc', created._id], created)
    },
  })
}

// ─── useUpdateDocument ────────────────────────────────────────────────────────

export function useUpdateDocument<T extends Document = Document>(collectionId: string) {
  const { client } = useMudbase()
  const queryClient = useQueryClient()

  return useMutation<T, Error, { documentId: string; data: Record<string, unknown> }>({
    mutationFn: ({ documentId, data }) =>
      client.updateDocument<T>(collectionId, documentId, data),
    onMutate: async ({ documentId, data }) => {
      // Optimistic update
      await queryClient.cancelQueries({
        queryKey: ['collection', collectionId, 'doc', documentId],
      })
      const previous = queryClient.getQueryData<T>([
        'collection',
        collectionId,
        'doc',
        documentId,
      ])
      if (previous) {
        queryClient.setQueryData(['collection', collectionId, 'doc', documentId], {
          ...previous,
          ...data,
        })
      }
      return { previous, documentId }
    },
    onError: (_err, { documentId }, context) => {
      const ctx = context as { previous: T | undefined; documentId: string }
      if (ctx?.previous) {
        queryClient.setQueryData(
          ['collection', collectionId, 'doc', documentId],
          ctx.previous
        )
      }
    },
    onSettled: (_data, _err, { documentId }) => {
      queryClient.invalidateQueries({ queryKey: ['collection', collectionId, 'doc', documentId] })
      queryClient.invalidateQueries({ queryKey: ['collection', collectionId] })
    },
  })
}

// ─── useDeleteDocument ────────────────────────────────────────────────────────

export function useDeleteDocument(collectionId: string) {
  const { client } = useMudbase()
  const queryClient = useQueryClient()

  return useMutation<void, Error, string>({
    mutationFn: (documentId) => client.deleteDocument(collectionId, documentId),
    onSuccess: (_data, documentId) => {
      queryClient.removeQueries({
        queryKey: ['collection', collectionId, 'doc', documentId],
      })
      queryClient.invalidateQueries({ queryKey: ['collection', collectionId] })
    },
  })
}
```

### 3.3 Query Parameters Example

```typescript
// src/app/blog/page.tsx
import { useDocuments } from '@/hooks/useCollection'
import type { Document } from '@/lib/mudbase'

interface Post extends Document {
  title: string
  slug: string
  status: 'draft' | 'published' | 'archived'
  tags: string[]
  publishedAt: string
  viewCount: number
}

// Published posts tagged "typescript", newest first, page 2 of 10-per-page.
// `filter` is a plain Mongo query object — array membership is a bare equality match.
const { data: page2, isLoading } = useDocuments<Post>('posts', {
  filter: { status: 'published', tags: 'typescript' },
  sort: '-publishedAt',
  page: 2,
  limit: 10,
})

// Operators are written the Mongo way, not as a DSL.
const { data: popular } = useDocuments<Post>('posts', {
  filter: { status: 'published', viewCount: { $gte: 1000 } },
  sort: '-viewCount',
  limit: 20,
})

// Substring matching stands in for the absent `search` param. Anchor the pattern
// (`^`) where you can — an unanchored $regex forces a collection scan.
const { data: matches } = useDocuments<Post>('posts', {
  filter: { status: 'published', title: { $regex: '^React', $options: 'i' } },
  limit: 20,
})

// Trim the payload to the columns a list view actually renders.
const { data: index } = useDocuments<Post>('posts', {
  filter: { status: 'published' },
  fields: ['title', 'slug', 'publishedAt'],
  sort: '-publishedAt',
  limit: 50,
})

const total: number = page2?.pagination.total ?? 0
const hasNextPage: boolean = page2?.pagination.hasMore ?? false
const posts: Post[] = page2?.data ?? []
```

---

## 4. File Storage

### 4.0 The two storage surfaces

mudbase exposes storage through two distinct route trees. Pick one per use case; they are not interchangeable.

| Surface | Mount | Addressing | Use for |
|---------|-------|-----------|---------|
| Buckets | `/api/bucket` | `projects/:projectId/buckets/:bucketId/...`, `bucketId` is an ObjectId | Managed buckets with per-bucket size caps, allowed-type lists, ACLs, image transforms |
| Flat files | `/api/files` | `projectId` + a bucket **name** string in the body | Direct-to-S3 presigned uploads, download URLs |

Key contract details that differ from most BaaS platforms:

- The multipart field name for a bucket upload is **`files`** (plural, array), max 10 per request — not `file`.
- The presigned flow returns an **S3 presigned POST**, not a PUT URL. You must send a `multipart/form-data` POST with every entry of the returned `fields` object appended *before* the file part. A raw `PUT` to that URL fails.
- A signed download URL is obtained with a **POST** carrying `{ expiresIn }`, and the response field is `signedUrl` (plus `expiresAt`), not `url`.
- Uploading into a public bucket does not force the object public — pass `isPublic` per upload to store a private object inside a public bucket.
- mudbase refuses to sign a public object: for `isPublic: true` files the download endpoint returns the permanent public URL plus a `warning`, because signing a world-readable object buys nothing.

### 4.1 Direct Upload (simple, for small files)

```typescript
// src/hooks/useFileUpload.ts
import { useState, useCallback } from 'react'
import { useMudbase } from '@/lib/mudbase-provider'
import type { FileObject } from '@/lib/mudbase'

interface UploadState {
  uploading: boolean
  progress: number
  error: string | null
  file: FileObject | null
}

interface UseFileUploadResult extends UploadState {
  upload: (file: File, isPublic?: boolean) => Promise<FileObject | null>
  reset: () => void
}

export function useFileUpload(bucketId: string): UseFileUploadResult {
  const { client } = useMudbase()
  const [state, setState] = useState<UploadState>({
    uploading: false,
    progress: 0,
    error: null,
    file: null,
  })

  const upload = useCallback(
    async (file: File, isPublic = false): Promise<FileObject | null> => {
      const token = client.getToken()
      if (!token) {
        setState({ uploading: false, progress: 0, error: 'Not authenticated', file: null })
        return null
      }

      setState({ uploading: true, progress: 0, error: null, file: null })
      try {
        // fetch() has no upload-progress event, so bucket uploads go through XHR.
        const uploaded = await uploadWithProgress(
          { token, projectId: client.getProjectId(), baseUrl: client.getBaseUrl() },
          bucketId,
          file,
          isPublic,
          (p) => setState((prev) => ({ ...prev, progress: p }))
        )
        setState({ uploading: false, progress: 100, error: null, file: uploaded })
        return uploaded
      } catch (err: unknown) {
        const msg = err instanceof Error ? err.message : 'Upload failed'
        setState({ uploading: false, progress: 0, error: msg, file: null })
        return null
      }
    },
    [client, bucketId]
  )

  const reset = useCallback((): void => {
    setState({ uploading: false, progress: 0, error: null, file: null })
  }, [])

  return { ...state, upload, reset }
}

interface UploadContext {
  token: string
  projectId: string
  baseUrl: string
}

interface BucketUploadResponse {
  success: boolean
  files: FileObject[]
}

function uploadWithProgress(
  ctx: UploadContext,
  bucketId: string,
  file: File,
  isPublic: boolean,
  onProgress: (percent: number) => void
): Promise<FileObject> {
  return new Promise<FileObject>((resolve, reject) => {
    const formData = new FormData()
    // Field name is `files` — multer is configured as .array("files", 10).
    formData.append('files', file)
    formData.append('isPublic', String(isPublic))

    const xhr = new XMLHttpRequest()
    xhr.open(
      'POST',
      `${ctx.baseUrl}/api/bucket/projects/${ctx.projectId}/buckets/${bucketId}/files`
    )
    xhr.setRequestHeader('Authorization', `Bearer ${ctx.token}`)

    xhr.upload.onprogress = (e: ProgressEvent): void => {
      if (e.lengthComputable) {
        onProgress(Math.round((e.loaded / e.total) * 100))
      }
    }

    xhr.onload = (): void => {
      let parsed: unknown
      try {
        parsed = JSON.parse(xhr.responseText || '{}')
      } catch {
        reject(new Error(`Upload failed: unreadable response (status ${xhr.status})`))
        return
      }

      if (xhr.status >= 200 && xhr.status < 300) {
        const body = parsed as BucketUploadResponse
        const uploaded = body.files?.[0]
        if (!uploaded) {
          reject(new Error('Upload succeeded but returned no file record'))
          return
        }
        resolve(uploaded)
        return
      }

      // 413 carries maxFileUploadBytes; 400 carries a MIME-policy `code`.
      const body = parsed as { error?: string; maxFileUploadBytes?: number }
      const detail = body.maxFileUploadBytes
        ? ` (max ${body.maxFileUploadBytes} bytes)`
        : ''
      reject(new Error(`${body.error ?? `Upload failed with status ${xhr.status}`}${detail}`))
    }

    xhr.onerror = (): void => reject(new Error('Network error during upload'))
    xhr.send(formData)
  })
}
```

The `getProjectId()` / `getBaseUrl()` accessors above are two one-line additions to `MudbaseClient`:

```typescript
// src/lib/mudbase.ts — inside class MudbaseClient
  getProjectId(): string {
    return this.projectId
  }

  getBaseUrl(): string {
    return this.baseUrl
  }
```

### 4.2 Presigned Upload (production pattern for large files)

Three steps: ask mudbase for a presigned POST, POST the file straight to object storage, then tell mudbase the object landed so it can virus-scan it and create the metadata record.

```typescript
// src/lib/presigned-upload.ts
import { getMudbaseClient } from '@/lib/mudbase'

export interface PresignedUploadResult {
  fileId: string
  key: string
}

export async function uploadWithPresignedPost(
  file: File,
  options: { bucket?: string; isPublic?: boolean } = {}
): Promise<PresignedUploadResult> {
  const client = getMudbaseClient()

  // 1. Ask mudbase for the presigned POST. mudbase derives the object key itself —
  //    the client never picks it, which is what stops key-collision and path traversal.
  const presigned = await client.getPresignedUploadUrl({
    bucket: options.bucket ?? 'default',
    originalName: file.name,
    contentType: file.type,
    isPublic: options.isPublic ?? false,
  })

  if (file.size > presigned.maxFileUploadBytes) {
    throw new Error(
      `File is ${file.size} bytes; this plan allows ${presigned.maxFileUploadBytes}.`
    )
  }

  // 2. POST directly to object storage. The `fields` entries are the signed policy and
  //    MUST be appended before the file part — S3 ignores anything after `file`.
  const form = new FormData()
  for (const [name, value] of Object.entries(presigned.fields)) {
    form.append(name, value)
  }
  form.append('file', file)

  const uploadRes = await fetch(presigned.url, { method: 'POST', body: form })
  if (!uploadRes.ok) {
    throw new Error(`Presigned upload failed: ${uploadRes.status} ${uploadRes.statusText}`)
  }

  // 3. Confirm — keyed by the object key, not a client-generated file id. This is the
  //    step that scans the object and writes the File record; skip it and the upload
  //    is an orphaned blob mudbase knows nothing about.
  const confirmed = await client.confirmUpload({
    key: presigned.key,
    originalName: file.name,
    contentType: file.type,
    size: file.size,
    bucket: options.bucket ?? 'default',
    isPublic: options.isPublic ?? false,
  })

  return { fileId: confirmed.fileId, key: presigned.key }
}
```

A quarantined file (virus scan hit) comes back as `400` with `{ message: "File quarantined", details }` — surface that distinctly from a generic upload failure, because retrying will not help.

### 4.3 File Upload Component

```typescript
// src/components/FileUpload.tsx
'use client'

import { useRef, useState } from 'react'
import { useFileUpload } from '@/hooks/useFileUpload'
import { useMudbase } from '@/lib/mudbase-provider'

interface FileUploadProps {
  bucketId: string
  accept?: string
  maxSizeMB?: number
  isPublic?: boolean
  onUploaded?: (url: string, fileId: string) => void
}

export function FileUpload({
  bucketId,
  accept = '*/*',
  maxSizeMB = 10,
  isPublic = false,
  onUploaded,
}: FileUploadProps) {
  const inputRef = useRef<HTMLInputElement>(null)
  const { client } = useMudbase()
  const { upload, uploading, progress, error, file, reset } = useFileUpload(bucketId)
  const [localError, setLocalError] = useState<string | null>(null)

  const handleFileChange = async (e: React.ChangeEvent<HTMLInputElement>): Promise<void> => {
    const selected = e.target.files?.[0]
    if (!selected) return

    setLocalError(null)
    if (selected.size > maxSizeMB * 1024 * 1024) {
      setLocalError(`File must be under ${maxSizeMB}MB`)
      return
    }

    const uploaded = await upload(selected, isPublic)
    if (!uploaded || !onUploaded) return

    if (uploaded.isPublic) {
      // Public objects already carry a permanent URL; mudbase declines to sign them.
      onUploaded(uploaded.url, uploaded.id)
      return
    }

    try {
      const signedUrl = await client.getSignedUrl(bucketId, uploaded.id)
      onUploaded(signedUrl, uploaded.id)
    } catch (err: unknown) {
      setLocalError(err instanceof Error ? err.message : 'Could not sign the uploaded file')
    }
  }

  const message = error ?? localError

  return (
    <div className="space-y-2">
      <input
        ref={inputRef}
        type="file"
        accept={accept}
        onChange={handleFileChange}
        disabled={uploading}
        className="hidden"
      />
      <button
        type="button"
        onClick={() => {
          reset()
          setLocalError(null)
          inputRef.current?.click()
        }}
        disabled={uploading}
        className="w-full rounded-md border border-dashed px-4 py-8 text-sm text-gray-500 hover:bg-gray-50 disabled:opacity-50"
      >
        {uploading
          ? `Uploading... ${progress}%`
          : file
            ? `Uploaded: ${file.originalName}`
            : 'Click to upload file'}
      </button>
      {uploading && (
        <div className="h-2 w-full rounded bg-gray-200">
          <div
            className="h-full rounded bg-blue-500 transition-all"
            style={{ width: `${progress}%` }}
          />
        </div>
      )}
      {message && <p className="text-sm text-red-600">{message}</p>}
    </div>
  )
}
```

---

## 5. Real-time WebSocket Events

### 5.1 `useRealtimeSubscription` Hook

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

### 5.2 Typed Event Hooks

```typescript
// src/hooks/useCollectionRealtime.ts
import { useQueryClient } from '@tanstack/react-query'
import { useRealtimeSubscription } from './useRealtimeSubscription'
import type { Document } from '@/lib/mudbase'

// Live-sync a collection with React Query cache
export function useCollectionRealtime(collectionId: string, enabled = true): void {
  const queryClient = useQueryClient()

  useRealtimeSubscription(
    `collection:${collectionId}`,
    'create',
    (data) => {
      const doc = data as Document
      queryClient.setQueryData(['collection', collectionId, 'doc', doc._id], doc)
      queryClient.invalidateQueries({ queryKey: ['collection', collectionId] })
    },
    enabled
  )

  useRealtimeSubscription(
    `collection:${collectionId}`,
    'update',
    (data) => {
      const doc = data as Document
      queryClient.setQueryData(['collection', collectionId, 'doc', doc._id], doc)
      queryClient.invalidateQueries({ queryKey: ['collection', collectionId] })
    },
    enabled
  )

  useRealtimeSubscription(
    `collection:${collectionId}`,
    'delete',
    (data) => {
      const { _id } = data as { _id: string }
      queryClient.removeQueries({ queryKey: ['collection', collectionId, 'doc', _id] })
      queryClient.invalidateQueries({ queryKey: ['collection', collectionId] })
    },
    enabled
  )
}
```

```typescript
// src/hooks/useUserNotifications.ts
import { useState, useCallback } from 'react'
import { useRealtimeSubscription } from './useRealtimeSubscription'
import { useMudbase } from '@/lib/mudbase-provider'

interface Notification {
  id: string
  title: string
  body: string
  data?: Record<string, unknown>
  readAt?: string
  createdAt: string
}

export function useUserNotifications() {
  const { session } = useMudbase()
  const [notifications, setNotifications] = useState<Notification[]>([])
  const userId = session?.user?.id

  const handleNotification = useCallback((data: unknown) => {
    const notif = data as Notification
    setNotifications((prev) => [notif, ...prev])
  }, [])

  useRealtimeSubscription(
    `user:${userId}`,
    'notification',
    handleNotification,
    !!userId
  )

  const markRead = useCallback((id: string) => {
    setNotifications((prev) =>
      prev.map((n) => (n.id === id ? { ...n, readAt: new Date().toISOString() } : n))
    )
  }, [])

  const unreadCount = notifications.filter((n) => !n.readAt).length

  return { notifications, unreadCount, markRead }
}
```

Common mudbase real-time event channels:
- `collection:{collectionId}` events: `create`, `update`, `delete`
- `user:{userId}` events: `notification`, `role_updated`, `session_revoked`
- `project:{projectId}` events: `analytics`, `function_executed`

---

## 6. Serverless Functions

### 6.0 Execution is asynchronous

`POST /api/functions/projects/:projectId/functions/:functionId/execute` returns **`202 Accepted`** with `{ success: true, data: { executionId, status: "queued" } }`. It never returns the function's result. The run is enqueued onto a BullMQ queue and picked up by `functionExecutionWorker`; the result lands somewhere else. Two ways to collect it:

1. **Poll** `GET /api/functions/projects/:projectId/functions/:functionId/executions/:executionId` — returns `{ executionId, status, durationMs, error, errorClass, logs, result, createdAt, startedAt, completedAt }` with `status` in `queued | running | success | failed`.
2. **Subscribe** to the Socket.IO `function.execution.completed` / `function.execution.failed` events documented in §12. Cheaper and lower-latency when you already hold a socket; the poller below is still the right fallback for a page loaded fresh with an `executionId` in the URL.

Anything written against an "invoke returns the result" assumption will silently receive `{ executionId, status }` where it expected a domain object.

### 6.1 Invoking Functions from Client

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
 * Poll an execution to a terminal state. Timing out does not cancel the run —
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
      return execution.result as TResult
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

### 6.2 React Query Hooks for Functions

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
 * Pair with the Socket.IO listener in §12 when you want push instead of pull;
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
          Confirmed — reference {execution.data.result.transactionRef}, arriving{' '}
          {execution.data.result.estimatedDelivery}
        </p>
      )}
    </div>
  )
}
```

### 6.3 Decision: Function vs Client Logic

| Use a Function | Use Client Logic |
|----------------|-----------------|
| Calls third-party APIs with secret keys | UI state management |
| Needs access to other collections/users' data | Form validation |
| Scheduled or triggered by events | Client-only computations |
| Complex data transformations | Local filtering/sorting of fetched data |
| Webhook handling | Animation/interaction |

Because execution is queued, a function is the wrong tool for anything on a synchronous critical path — a form submit that must show a result within a few hundred milliseconds is better served by a data-plane write plus a document trigger.

---

## 7. Multi-Role System

### 7.1 Role Selector on Signup

**The role config endpoint is not public.** `GET /api/projects/:projectId/multi-role` runs behind `authOrApiKey` + `validateProjectAccess` + `rbacCheck("project", "read")` — an anonymous browser on your signup page cannot call it. Calling `client.getMultiRoleConfig()` from an unauthenticated page returns `401`.

Two workable patterns:

1. **Bake the role list in at build time** (preferred for a signup page). The roles a project offers change on the order of months, not requests. Fetch them in a server component / build step using a server-side API key and render a static list.
2. **Proxy it** through your own backend route that holds the API key, if you need it live.

Below is pattern 1, with the fetch on the server and the selector as a client component.

```typescript
// src/app/auth/signup/page.tsx — server component, no 'use client'
import { RoleSelector } from '@/components/auth/RoleSelector'
import type { MultiRoleConfig, MultiRoleRole } from '@/lib/mudbase'

const MUDBASE_URL = process.env.MUDBASE_URL ?? 'https://cloud.mudbase.dev'
const PROJECT_ID = process.env.MUDBASE_PROJECT_ID
const API_KEY = process.env.MUDBASE_API_KEY // server-side only — never NEXT_PUBLIC_

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
import type { MultiRoleRole } from '@/lib/mudbase'
import { SignupForm } from '@/components/auth/SignupForm'

export function RoleSelector({ roles }: { roles: MultiRoleRole[] }) {
  // A single-role project skips the picker entirely.
  const [selectedRole, setSelectedRole] = useState<string | null>(
    roles.length === 1 ? roles[0].slug : null
  )

  if (selectedRole !== null) {
    return (
      <SignupForm
        role={selectedRole}
        onChangeRole={roles.length === 1 ? undefined : () => setSelectedRole(null)}
      />
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

Roles are identified by `slug` (not `id`), and each carries a `signupEndpoint` — the role-scoped signup path the `register()` client method targets.

### 7.2 Role-Based UI Guards

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

---

## 8. API Keys Management

API keys are for server-to-server communication. Never expose them to the browser.

### 8.1 The REST contract

The routes are **flat and org-scoped**, mounted at `/api/api-keys` (and aliased at `/api/tokens`). There is no `/api/projects/:id/api-keys`. The project is named in the request body on create, and as a path segment on the per-project list.

| Operation | Method + path |
|-----------|---------------|
| List (whole org) | `GET /api/api-keys` |
| List (one project) | `GET /api/api-keys/project/:projectId` |
| Create | `POST /api/api-keys` — body carries `projectId` |
| Update / revoke | `PATCH /api/api-keys/:id` — `{ isActive: false }` to revoke |
| Delete | `DELETE /api/api-keys/:id` |
| Usage stats | `GET /api/api-keys/:id/usage` |
| Rotate | `POST /api/api-keys/:id/regenerate` |

### 8.2 Permissions are structured, and required

`permissions` is **not** a string array. It is `{ resource, actions }[]`, validated against a fixed enum. An empty or missing array is rejected with `400` — mudbase deliberately has no "grant everything" default, so you must state the grant explicitly.

| Resources | Actions |
|-----------|---------|
| `auth`, `database`, `storage`, `functions`, `realtime`, `messaging`, `wallet`, `transactions`, `addons`, `kyc`, `payments` | `create`, `read`, `update`, `delete` |

Two further server-side rules:

- Only `owner` and `admin` may grant `delete` on any resource. A `developer` creating a key with any `delete` action gets `403`.
- The raw key is returned exactly once, on create (`apiKey.key`) and on rotate (`key`). Every later read exposes only `keyPrefix` / `keyPreview`. There is no recovery path — losing it means rotating.

Creation is also rate-limited (50/hour by default) and honours an `X-Idempotency-Key` header.

```typescript
// src/app/api/admin/api-keys/route.ts
// Server-side only — Next.js Route Handler.
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
  // or write it to your secret store here — do not log it, do not persist it in your DB.
  return NextResponse.json({
    id: body.apiKey._id,
    key: body.apiKey.key,
    name: body.apiKey.name,
    keyPrefix: body.apiKey.keyPrefix,
  })
}
```

### 8.3 Least-privilege examples

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

/** An MCP key for an AI coding agent — scope to what it is actually building against. */
export const AGENT_GRANT: ApiKeyPermission[] = [
  { resource: 'database', actions: ['create', 'read', 'update'] },
  { resource: 'functions', actions: ['create', 'read', 'update'] },
  { resource: 'storage', actions: ['read'] },
]
```

The MCP server (§24) authenticates with exactly these keys and checks each write tool against the grant, so a narrow key is a real containment boundary for an agent, not just a label.

**When to use API keys vs JWTs:**

| API Key | JWT (User Token) |
|---------|-----------------|
| Server-to-server calls | User-facing client requests |
| CI/CD pipelines, cron jobs | Browser, mobile app sessions |
| Stored in env vars / secrets manager | Stored in memory / httpOnly cookie |
| Long-lived (months/years) | Short-lived (hours/days) |
| Scoped to service permissions | Scoped to user's own data |

---

## 9. Environment Variables

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

---

## 10. Security Checklist

- [ ] API keys and service account tokens never in client bundle — only in server-side env vars
- [ ] **Token storage — pick deliberately, mudbase gives you no cookie option.** Every mudbase auth endpoint returns the bearer token in a JSON body; there is no `Set-Cookie`, so mudbase itself cannot give you an httpOnly session. That leaves three honest choices:
  - **Browser, default:** `sessionStorage`, not `localStorage`. Both are XSS-readable, but `sessionStorage` clears on tab close and is not shared across tabs, which shortens the residual-exposure window. Only reach for `localStorage` if "stay signed in across browser restarts" is a real product requirement, and accept that the token then sits at rest on disk indefinitely.
  - **Browser, higher assurance:** proxy authentication through your own backend. Your Next.js route handler (or equivalent) calls `/api/auth/local/login`, keeps the mudbase bearer token server-side (in your session store), and sets your *own* `httpOnly; Secure; SameSite=Lax` session cookie on the browser. The mudbase token then never reaches client JavaScript at all. This is the only configuration where XSS cannot exfiltrate the credential, and it is the right default for anything handling money, health data, or admin capability.
  - **Mobile (React Native / Expo):** `expo-secure-store` or `react-native-keychain`. Never `AsyncStorage` — it is unencrypted plaintext on disk and readable on a rooted or jailbroken device.
- [ ] Anonymous sessions: always convert before storing sensitive user data (PII, payment info)
- [ ] File buckets: set access policy correctly — private for user uploads, public only for truly public assets
- [ ] Real-time channels: always scope to the authenticated user's own data. Never subscribe to `collection:{id}` for all users — subscribe to `user:{userId}:*` or filter by your own userId
- [ ] OAuth: save the return path to `sessionStorage` before redirecting, validate the token against `getSession()` after callback — never trust token from URL alone without server validation
- [ ] Functions: validate every payload field on the function side — never trust client-provided IDs or roles
- [ ] Never expose `projectId` in combination with admin-level API keys — use scoped keys with minimum required permissions
- [ ] Password reset and magic link tokens are one-time use — confirm on the server that the token has not already been consumed. Project-scoped resets are OTP-based (6 digits, 10-minute TTL, `POST /api/auth/password-reset/confirm`), not link-token based
- [ ] 2FA: mudbase's login endpoint does **not** gate on 2FA — it returns a usable token regardless. If your product promises a second factor at login, enforce it in your own backend proxy (see the token-storage item above); do not present `user.twoFactorEnabled` to users as though the server were blocking on it
- [ ] Rate limit auth endpoints on the server — mudbase handles this internally, but add client-side debounce on forms
- [ ] Content Security Policy: if loading files from mudbase storage via signed URLs, add the storage domain to your CSP `img-src` / `media-src` directives

---

## 11. Socket.IO — Connection & Setup

mudbase real-time uses **Socket.IO** on path `/socket.io/` at `https://cloud.mudbase.dev`. The raw WebSocket client in Section 1 is replaced by this Socket.IO client for all real-time features.

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
      auth: { token },           // JWT sent in handshake
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

  /** Emit with optional acknowledgement callback */
  emit(event: string, data: unknown, ack?: (res: unknown) => void): void {
    if (!this.socket?.connected) {
      console.warn(`[mudbase socket] emit "${event}" while not connected`)
      return
    }
    if (ack) {
      this.socket.emit(event, data, ack)
    } else {
      this.socket.emit(event, data)
    }
  }

  /** Subscribe to a server event; returns unsubscribe fn */
  on<T = unknown>(event: string, handler: (data: T) => void): () => void {
    this.socket?.on(event, handler as (...args: unknown[]) => void)
    return () => this.socket?.off(event, handler as (...args: unknown[]) => void)
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

JWT `scope` must be `api` or `websocket`. Invalid/expired tokens → `connect_error` with message `Authentication error`. Token with `jti` must not be blacklisted; token with `sid` requires active session.

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

---

## 12. Database Events (Socket.IO — Expanded)

Replaces / expands Section 5. Uses Socket.IO, not raw WebSocket.

### Room subscriptions

```typescript
const socket = getMudbaseSocket()

// Subscribe to entire collection (all documents)
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
  queryId: 'active-orders',   // client-chosen stable ID
  query: { status: 'active' },
})

// Subscribe to project-level events
socket.emit('subscribe:project', 'YOUR_PROJECT_ID')
```

### Schema events (collection lifecycle)

Delivered to `project:<projectId>` room.

| Event | When |
|-------|------|
| `db:collection_created` | Collection created |
| `db:collection_updated` | Schema / metadata updated |
| `db:collection_deleted` | Collection deleted |

```typescript
// Payload shape (db:collection_created example)
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

Delivered to `collection:<collectionId>` and `project:<projectId>` rooms.

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
  data: Record<string, unknown>  // full document (or { _id } on delete)
  timestamp: string
}

socket.on<RowEvent>('db:create', (ev) => {
  console.log('New document:', ev.data._id)
})
```

Additional events from `emitDataChange`:
- `data:change` → collection room
- `document:change` → document room
- `project:data:change` → project room

### React Query live sync hook (Socket.IO version)

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

---

## 13. Chat & Messaging

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
socket.emit('message:react', { chatId, projectId, messageId, emoji: '👍' })
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
      // Auto-clear after 3s in case stop event is missed
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

---

## 14. Presence

Track who is online in a project. Presence is project-scoped.

```typescript
const socket = getMudbaseSocket()

// Announce online
socket.emit('presence:online', { projectId: 'YOUR_PROJECT_ID' })

// Get current online list
socket.emit('presence:get:online', { projectId: 'YOUR_PROJECT_ID' })

// Update last-seen timestamp
socket.emit('presence:update')   // no payload needed

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

---

## 15. Wallets & Blockchain

mudbase supports multi-chain crypto wallets as a first-class feature. Max 50 wallet rooms per socket.

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

// By chain (ethereum, bitcoin, solana, etc. — lowercase)
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
  address: string             // on-chain address, e.g. "0xabc..."
  chain: string               // "ethereum", "bitcoin", etc.
  project: string
  org: string
  previousBalance: string     // decimal string
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
  balances: Record<string, string>  // chain → balance
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

---

## 16. Integrations Framework

mudbase has a native integrations system (Stripe, Twilio, third-party APIs). Use the integration room to execute endpoints and receive webhook mirrors in real-time.

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
// Subscribe to webhook mirror stream
socket.emit('integration:webhook:subscribe', {
  projectId: 'YOUR_PROJECT_ID',
  integrationId: 'INTEGRATION_ID',
}, () => {
  console.log('webhook mirror active')
})

socket.on('integration:webhook:received', (ev: {
  integrationId: string
  event: string              // e.g. "stripe.invoice.paid"
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

---

## 17. Custom Channels

Arbitrary real-time events scoped to a project — use for collaborative features, typing indicators, kanban boards, etc.

```typescript
const socket = getMudbaseSocket()

// Broadcast named event to everyone in the project
socket.emit('custom:broadcast', {
  projectId: 'YOUR_PROJECT_ID',
  event: 'board_updated',       // becomes "custom:board_updated" on recipients
  payload: { columnId: 'done', cardId: 'abc' },
  room: undefined,               // omit to broadcast to whole project
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

---

## 18. Calls & Signaling

mudbase provides signaling events for coordinating VoIP/video call UI. Not a full WebRTC stack — use a TURN/STUN server (Twilio, LiveKit, 100ms) for media transport; mudbase handles the signal coordination.

```typescript
const socket = getMudbaseSocket()

// Caller initiates
socket.emit('call:initiate', {
  chatId: 'CHAT_ID',
  type: 'video',               // 'video' | 'audio'
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
          ? { status: 'active', chatId: ev.chatId, type: (prev as { type: 'video' | 'audio' }).type ?? 'video' }
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

---

## 19. Server-Sent Events (SSE)

One-way event stream from mudbase for notifications, task progress, and activity. Simpler than Socket.IO when you don't need bidirectional communication.

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

data: {"type":"function.execution.completed","payload":{"functionId":"process-order","executionId":"…","result":{}}}
```

Use SSE for: one-way push notifications, progress bars, activity logs, build/job status updates.
Use Socket.IO for: chat, presence, wallet events, bidirectional communication.

---

## 20. Webhooks (Outbound — mudbase → your server)

A mudbase project has **one** webhook destination, configured as part of the project record — not a collection of webhook subscriptions. You set the URL, the signing secret, and the event allow-list together, and mudbase delivers every subscribed event to that single endpoint.

### Configuring the endpoint

`PUT /api/webhooks/projects/:projectId/config`, authenticated with a bearer token or API key holding `project:update`.

| Body field | Type | Notes |
|------------|------|-------|
| `webhookUrl` | string | Validated against the SSRF guard — no private ranges, no DNS rebinding. Setting the first URL counts against the plan's webhooks-per-project limit |
| `webhookSecret` | string | HMAC-SHA256 signing key. Write-only; reads report only `hasSecret: true` |
| `webhookEvents` | string[] | Allow-list. An **empty array means "all events"**, which is the legacy default — always send an explicit list |
| `webhookVersion` | string | Payload version pin |
| `transformations` | object[] | Optional payload reshaping applied before delivery |

`GET /api/webhooks/projects/:projectId/config` returns the same fields minus the secret.

```typescript
// src/lib/mudbase-webhooks.ts — server-side only
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
    // An empty array is read as "deliver everything" server-side — almost never what you want.
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

- Document CRUD emits **`collection.insert` / `collection.update` / `collection.delete`**. There is no `document.*` event. (`data.created` / `data.updated` / `data.deleted` are accepted as subscription aliases in the enum, but the data routes dispatch the `collection.*` names — subscribe to those.)
- Function outcomes are **`function.execution.completed` / `function.execution.failed`**, matching the Socket.IO event names in §12. There is no `function.completed`.

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
| `wallet.created` / `wallet.transaction` / `wallet.balance_changed` | Wallet activity (see §15, §30) |
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
X-MUDBASE-Delivery-ID: <WebhookLog id — use this for idempotency>
X-MUDBASE-Delivery: <same value, legacy header name>
```

```jsonc
{ "event": "collection.insert", "data": { /* the document */ }, "timestamp": "2026-07-27T09:12:44.001Z", "projectId": "…" }
```

There is no `id` field inside the body — the deduplication key is the `X-MUDBASE-Delivery-ID` header.

### Receiving and verifying

The signature is the hex HMAC-SHA256 of the exact JSON body mudbase sent. Compare it in constant time — and check the length **before** calling `timingSafeEqual`, which throws `RangeError` on mismatched buffer lengths rather than returning `false`. A handler that lets that throw turns a malformed signature into a 500 instead of a clean 401.

```typescript
// src/app/api/webhooks/mudbase/route.ts (Next.js App Router)
import { NextRequest, NextResponse } from 'next/server'
import crypto from 'crypto'
import type { MudbaseWebhookEvent } from '@/lib/mudbase-webhooks'

const WEBHOOK_SECRET = process.env.MUDBASE_WEBHOOK_SECRET

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

  // Idempotency keyed on the delivery header — retries reuse the same id.
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
      // Unknown or unhandled event — still 200 so mudbase stops retrying.
      break
  }

  return NextResponse.json({ ok: true })
}
```

### Retry policy

Three attempts total, at fixed intervals — not five, and not exponential out to hours. A background sweep runs every 5 minutes and picks up anything still pending.

| Attempt | Delay after the previous failure |
|---------|----------------------------------|
| 1 | Immediate |
| 2 | 1 minute |
| 3 | 5 minutes |

(The third retry delay of 30 minutes is configured but only reached when `WEBHOOK_MAX_ATTEMPTS` is raised above the default 3; `WEBHOOK_MAX_ATTEMPTS` and `WEBHOOK_RETRY_DELAYS` are deployment-level env overrides, not per-project settings.)

Return `200` as soon as you have durably accepted the payload, then do the work asynchronously. After the final attempt the delivery is abandoned — `POST /api/webhooks/retry/:webhookId` re-drives a specific failed delivery, and `GET /api/webhooks/projects/:projectId` lists the delivery log.

---

## 21. Background Processing — what actually runs where

Two separate things, often conflated. Be precise about which one you are relying on.

### 21.1 Your functions: a real queue

Function execution is genuinely queued, not simulated. `POST .../execute` writes a `FunctionExecution` record, pushes onto a BullMQ queue, and returns `202`; `jobs/functionExecutionWorker.js` drains it. §6 covers the client contract (poll the execution endpoint, or listen for `function.execution.completed` / `.failed`). Scheduled functions are driven by `jobs/functionCronJob.js` — you configure the schedule in the console, and the client only ever observes results.

There is no separate "jobs API". If you want a background job, you deploy a function and invoke or schedule it.

### 21.2 Platform workers: server-side, not client-callable

mudbase also runs a set of its own background processes. You cannot invoke these, and there is no client API for them — but they explain latency you will otherwise misread as a bug:

| Process | Why it matters to a client app |
|---------|-------------------------------|
| Usage metering worker | API-call and storage counters flush asynchronously, so a usage dashboard can lag a burst of writes by a short interval |
| Wallet indexers / non-EVM pollers | Incoming deposits are detected by polling and indexing, not synchronously — a deposit appears some blocks after it confirms on chain |
| Webhook retry sweep | Runs every 5 minutes; a failed delivery is retried on that cadence, not instantly |
| Project clone / export / migration-import workers | Long operations return a `jobId` and complete out of band — poll the corresponding job-status endpoint (§27, §28) |
| Add-on worker | Add-on invocations may return `202` with a pending job; poll it (§25) |
| Billing crons (overage, credit low-balance, payout scheduler) | Invoices, credit alerts and payouts materialise on a schedule, not the instant a threshold is crossed |
| Daily storage / usage-integrity crons | Storage totals and integrity corrections settle daily |

Practical consequence: treat any counter, balance, or usage figure mudbase reports as eventually consistent, and never build a client-side gate ("block the user once usage hits the limit") on a freshly-read counter alone. The server enforces its own limits at request time; that is the authoritative check.

---

## 22. Error Handling & Retry Logic

### mudbase HTTP error codes

| Code | Meaning | Client action |
|------|---------|---------------|
| `400` | Bad Request — invalid body or params | Fix the request; don't retry |
| `401` | Unauthorized — missing or invalid token | Refresh token or redirect to login |
| `403` | Forbidden — insufficient permissions | Show permission error to user |
| `404` | Not Found — resource does not exist | Show not-found UI |
| `429` | Too Many Requests — rate limit hit | Respect `Retry-After` header; exponential backoff |
| `5xx` | Server error | Retry with backoff (3 attempts max) |

### MudbaseError — typed error class

The client in Section 1 already throws `MudbaseError`. Use it for targeted handling:

```typescript
import { MudbaseError } from '@/lib/mudbase'

async function loadDashboard() {
  try {
    const data = await getMudbaseClient().getDocuments('orders')
    return data
  } catch (err) {
    if (!(err instanceof MudbaseError)) throw err

    switch (err.statusCode) {
      case 401:
        // Token expired — refresh session
        await getMudbaseClient().getSession()
        return loadDashboard()  // retry once
      case 403:
        router.push('/unauthorized')
        return null
      case 404:
        return { data: [], pagination: { page: 1, limit: 20, total: 0, totalPages: 0, hasMore: false } }
      case 429:
        // Retry-After is in the response headers — handled by retryWithBackoff below
        throw err
      default:
        if (err.statusCode >= 500) throw err
        throw err
    }
  }
}
```

### Exponential backoff with `Retry-After`

```typescript
// src/lib/retry.ts

interface RetryOptions {
  maxAttempts?: number
  initialDelayMs?: number
  maxDelayMs?: number
}

export async function retryWithBackoff<T>(
  fn: () => Promise<T>,
  options: RetryOptions = {}
): Promise<T> {
  const { maxAttempts = 3, initialDelayMs = 500, maxDelayMs = 30_000 } = options
  let attempt = 0

  while (attempt < maxAttempts) {
    try {
      return await fn()
    } catch (err) {
      attempt++
      if (attempt >= maxAttempts) throw err

      const isRetryable =
        err instanceof MudbaseError &&
        (err.statusCode === 429 || err.statusCode >= 500)

      if (!isRetryable) throw err

      // Respect Retry-After if present
      const retryAfter = (err.details as { retryAfter?: number })?.retryAfter
      const delay = retryAfter
        ? retryAfter * 1000
        : Math.min(initialDelayMs * 2 ** (attempt - 1), maxDelayMs)

      await new Promise((resolve) => setTimeout(resolve, delay))
    }
  }

  throw new Error('Max retry attempts reached')
}

// Usage
const data = await retryWithBackoff(() =>
  getMudbaseClient().getDocuments('orders')
)
```

### React Query retry config for mudbase

```typescript
// src/lib/query-client.ts
import { QueryClient } from '@tanstack/react-query'
import { MudbaseError } from '@/lib/mudbase'

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      retry: (failureCount, error) => {
        if (error instanceof MudbaseError) {
          // Never retry 4xx errors (fix the request)
          if (error.statusCode >= 400 && error.statusCode < 500) return false
          // Retry up to 3× on 5xx / network errors
          return failureCount < 3
        }
        return failureCount < 2
      },
      retryDelay: (attempt) => Math.min(500 * 2 ** attempt, 15_000),
      staleTime: 1000 * 30,
    },
    mutations: {
      retry: false,  // never auto-retry mutations — user should confirm
    },
  },
})
```

---

## 23. Rate Limiting — Client-Side Strategies

mudbase rate limiting is **windowed and per-IP**, applied by `express-rate-limit` layers — not a per-second per-project token bucket. Several buckets stack: the general API limiter runs on everything under `/api`, and the more specific limiter for the route you hit runs as well, so an auth call is bounded by both.

Defaults (each is env-tunable per deployment, and the org-adjustable ones can be raised per organisation from the console under `org.settings.rateLimits`):

| Bucket | Window | Default max | Org-adjustable |
|--------|--------|-------------|----------------|
| General API (`/api/*`) | 15 min | 2000 | yes |
| Auth (`/api/auth/*`) | 15 min | 20 | no |
| Login (`/api/auth/local/login`) | 15 min | 5 | no |
| Registration (`/api/auth/local/register`) | 60 min | 3 | no |
| Password-reset request | 15 min | per-project limiter, tight | no |
| Password-reset OTP confirm | 5 min | 3 | no |
| Token refresh | 15 min | 60 | no |
| Data mutations (create/update/delete document) | 1 min | 600 | yes |
| File upload | 60 min | 500 | yes |
| Collection creation | 15 min | 200 | yes |
| Project creation | 60 min | 30 | no |
| API-key creation | 60 min | 50 | no |
| Webhook trigger | 15 min | 120 | yes |
| Public endpoints | 15 min | 200 | no |

Two consequences for client design:

- The binding constraint on a bulk import is the **data-mutation** bucket — 600 writes/minute, i.e. roughly 10/second sustained — not the general 2000/15min figure. Size batches against that.
- Auth limits are per-IP. Users behind a shared NAT (an office, a mobile carrier gateway) share the budget, so a login form that retries automatically on failure can lock out an entire building. Never auto-retry a `401`.

### Detecting and handling 429

```typescript
// In MudbaseClient.request() — add Retry-After header handling
if (res.status === 429) {
  const retryAfter = parseInt(res.headers.get('Retry-After') ?? '1', 10)
  const error = await res.json().catch(() => ({}))
  throw new MudbaseError('Rate limit exceeded', 429, { retryAfter, ...error })
}
```

### Client-side request queue

For bulk operations (e.g. importing 500 documents), batch requests and respect the rate limit:

```typescript
// src/lib/request-queue.ts

export async function batchWithRateLimit<T>(
  items: T[],
  fn: (item: T) => Promise<unknown>,
  options: { batchSize?: number; delayMs?: number } = {}
): Promise<void> {
  const { batchSize = 10, delayMs = 200 } = options

  for (let i = 0; i < items.length; i += batchSize) {
    const batch = items.slice(i, i + batchSize)
    await Promise.all(batch.map(fn))
    if (i + batchSize < items.length) {
      await new Promise((resolve) => setTimeout(resolve, delayMs))
    }
  }
}

// Usage: import 500 documents inside the data-mutation budget (600 writes / 60s).
// 8 writes per 1000ms = 480/min, comfortably under the limit with headroom for
// whatever else the app is doing on the same IP.
await batchWithRateLimit(
  documents,
  (doc) => getMudbaseClient().createDocument('products', doc),
  { batchSize: 8, delayMs: 1000 }
)
```

### Debounce on search / filter inputs

```typescript
// src/hooks/useDebouncedSearch.ts
import { useState, useEffect } from 'react'

interface DebouncedSearch {
  query: string
  setQuery: (next: string) => void
  debouncedQuery: string
}

export function useDebouncedSearch(delay = 300): DebouncedSearch {
  const [query, setQuery] = useState('')
  const [debouncedQuery, setDebouncedQuery] = useState('')

  useEffect(() => {
    const timer = setTimeout(() => setDebouncedQuery(query), delay)
    return () => clearTimeout(timer)
  }, [query, delay])

  return { query, setQuery, debouncedQuery }
}
```

```typescript
// src/components/SearchBar.tsx
'use client'

import { useDebouncedSearch } from '@/hooks/useDebouncedSearch'
import { useDocuments } from '@/hooks/useCollection'
import type { Document } from '@/lib/mudbase'

// The data endpoint has no `search` param — build a filter. Escape the input before
// it becomes part of a regular expression, or a user typing "(" produces a 400.
function escapeRegex(value: string): string {
  return value.replace(/[.*+?^${}()|[\]\\]/g, '\\$&')
}

export function SearchBar({ collectionId }: { collectionId: string }) {
  const { query, setQuery, debouncedQuery } = useDebouncedSearch(300)

  const { data } = useDocuments<Document>(collectionId, {
    filter:
      debouncedQuery.length > 0
        ? { title: { $regex: escapeRegex(debouncedQuery), $options: 'i' } }
        : {},
    limit: 20,
  })

  return (
    <div>
      <input
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder="Search"
        className="w-full rounded-md border px-3 py-2"
      />
      <p className="mt-1 text-sm text-gray-500">{data?.pagination.total ?? 0} results</p>
    </div>
  )
}
```

### Socket.IO connection limit

mudbase enforces a plan-based socket connection limit. Handle the `error` event:

```typescript
// In MudbaseSocket.connect() — already handled:
this.socket.on('error', (data: { message: string; limit?: number }) => {
  if (data.message.includes('connection limit')) {
    // Notify user — can't add more sockets on this plan
    showToast(`Realtime connection limit reached (${data.limit} max on your plan)`, 'warning')
  }
  this.socket?.disconnect()
})
```

---

## 24. MCP Server (for AI coding agents)

This is a **different integration surface from everything above**. The REST client in §1 is what your application calls at runtime. The MCP server is what an AI coding agent — Claude Code, Cursor, any Model Context Protocol client — connects to while *building* the application, so it can create collections, inspect schemas, deploy functions and read data directly instead of guessing at your backend.

You do not call it from this SDK. Nothing in `MudbaseClient` touches `/mcp`.

### Connection contract

| | |
|---|---|
| Endpoint | `POST /mcp` on the org's API host (shared platform host, or the dedicated host for orgs with dedicated infrastructure) |
| Transport | Streamable HTTP, **stateless** — a fresh server and transport per request. `GET /mcp` and `DELETE /mcp` return `405`; there are no long-lived session streams |
| Auth | A project **API key**, sent as `X-API-Key: ak_…` or `Authorization: Bearer ak_…`. Not a user JWT |
| Plan gate | Paid plans only (`MCP_ALLOWED_PLANS`, default `starter,growth,scale,enterprise`). A free-tier key gets `402` with `code: "PLAN_REQUIRED"` |
| Scope | The key resolves to exactly one project and org. Write-capable tools are checked against the key's `permissions` grant (§8.2), so a read-only key cannot be talked into writing |
| Metering | Every MCP request is metered against the org's `apiCalls` quota, same as a REST call |

### Discovering whether it's available

`GET /mcp/config` uses the normal dashboard session (not an API key) and is what a console settings page calls:

```typescript
// src/lib/mcp-config.ts
export interface McpConfig {
  enabled: boolean
  plan: string
  allowedPlans: string[]
  endpoint: string
  tools: Array<{ name: string; description: string }>
}

export async function fetchMcpConfig(baseUrl: string, bearerToken: string): Promise<McpConfig> {
  const res = await fetch(`${baseUrl}/mcp/config`, {
    headers: { Authorization: `Bearer ${bearerToken}` },
  })
  if (!res.ok) {
    throw new Error(`Failed to load MCP config: ${res.status}`)
  }
  return (await res.json()) as McpConfig
}
```

The `endpoint` it returns is the URL to hand to the agent — resolve it rather than hardcoding `cloud.mudbase.dev`, because dedicated-infrastructure orgs run on their own host.

### Wiring an agent up

```bash
# Claude Code — scope the key to what the agent actually needs (see §8.3)
claude mcp add --transport http mudbase https://cloud.mudbase.dev/mcp \
  --header "X-API-Key: ak_your_project_key"
```

The tool list is not duplicated here — it is served live by `GET /mcp/config` and documented in full in mudbase's own MCP guide. Treat that as the source of truth; a tool inventory copied into a skill file goes stale the moment a tool is added.

Security note: an API key handed to an agent is a standing credential in that agent's config. Mint a dedicated key for it, grant the narrowest useful permission set, and rotate it (`POST /api/api-keys/:id/regenerate`) when the agent's access should end.

---

## 25. Add-ons Marketplace

Add-ons are small metered utilities the platform runs for you — QR/PDF/CSV/ICS/vCard generation, GeoIP, currency and crypto price lookup, UUID/hash/slug/password generation, Markdown rendering. Each successful invocation is billed per call against the org's in-app credit balance (§31).

| Operation | Method + path | Auth |
|-----------|---------------|------|
| Catalog | `GET /api/addons` | bearer or API key, `addon:read` |
| Invoke | `POST /api/projects/:projectId/addons/:addon/invoke` | bearer or API key, `addon:create`, project ownership |
| Job status | `GET /api/projects/:projectId/addons/jobs/:id` | bearer or API key, `addon:read` |

Invoke returns **`200` when the job finished inline** and **`202` when it is still processing** — both with the same `{ job }` envelope. Write the client against the `job.status` field, not the HTTP status, and poll when it is not yet terminal. Pass `X-Idempotency-Key` (or `idempotencyKey` in the body) so a retried invocation reuses the existing job rather than billing twice.

```typescript
// src/lib/mudbase.ts — inside class MudbaseClient
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
// src/lib/mudbase.ts — types
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

A `file`-mode result carries a `url` with an `expiresAt` — download or re-host it before it lapses rather than storing the URL as a permanent reference.

---

## 26. Identity Verification (KYC / KYB)

mudbase gates financial-egress actions behind organisation-level identity verification. Two distinct things share the name:

- **Platform KYC** — verifying *your* organisation, so mudbase will let you move money. This is what unblocks the gated actions below.
- **White-label KYC/KYB** — verifying *your customers*, on your behalf, scoped to one of your projects. You resell verification to your own end-users.

The whole subsystem is opt-in at the deployment level: if the verification provider is not configured, `requireKycApproved` is a no-op and nothing is gated.

### What is gated

`requireKycApproved` is evaluated **live on every call**, not once at an unlock step — so an organisation whose verification is later revoked or flagged loses access immediately rather than coasting on a stale approval. It sits in front of:

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

Start returns `201 { sessionId, url, status }`. Redirect the user to `url` — the verification itself happens in the provider's hosted flow, not in your UI. The org's `kycStatus` flips to `pending` immediately; it becomes `approved` (or not) when the provider's webhook lands, which is **after** the user finishes and can take minutes. There is no synchronous completion signal, so the client must poll `GET /api/kyc/status` (or wait for the user to return and refresh).

```typescript
// src/lib/mudbase.ts — inside class MudbaseClient
  async startKycSession(language?: string): Promise<KycSession> {
    return this.request<KycSession>('POST', '/api/kyc/sessions', { language })
  }

  async getKycStatus(): Promise<KycStatus> {
    return this.request<KycStatus>('GET', '/api/kyc/status')
  }
```

```typescript
// src/lib/mudbase.ts — types
export interface KycSession {
  sessionId: string
  /** Hosted verification URL — redirect the user here. */
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
 * Verification completes via a provider webhook, so there is nothing to await —
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

Start accepts `{ workflowId, vendorData, callback, language, reuseIdentifier }`. When `reuseIdentifier` matches an already-approved identity, mudbase satisfies the new verification from the existing one and returns `200` with `reused: true` instead of `201` — so the user is not asked to verify twice, and you are not billed twice. Results are delivered to the webhook destination you configure at `POST /api/kyc/webhook-config`, signed the same way as §20 webhooks.

---

## 27. Project Sharing, Cloning & Forking

**Management-plane, not data-plane.** These are console operations — an app built *on* a mudbase project never calls them. Reach for this section only when you are building tooling against mudbase's own management API (an internal admin surface, a template gallery, a provisioning script).

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

Three behaviours worth knowing before you build UI around this:

- **Publishing requires metadata.** A project cannot go `public` without a `communityMeta.title` and `communityMeta.summary`. Collect them in the same form as the visibility toggle or the request 400s.
- **Publishing production data requires a second, explicit confirmation.** If `communityMeta.dataProfile` is `production`, the request is rejected with `code: "PRODUCTION_DATA_PUBLIC_CONFIRMATION_REQUIRED"` unless you also send `confirmProductionDataPublic: true`. Surface that as a distinct, deliberately awkward confirmation step — it exists because a mislabelled dataset becomes copyable by anyone the instant it is published.
- **Clone and fork are partly asynchronous.** Without `includeFiles`, the response is `201` and the project already exists. With `includeFiles`, file copying is queued: the response carries a `jobId` and `status`, and you poll `GET /api/projects/clone-jobs/:jobId`. The message text differs too (`"Clone started"` vs `"Project cloned successfully"`) — branch on the presence of `jobId`, not on the message.
- **Forking versus cloning** differ in the access check, not the mechanics: `/clone` requires org access to the *source* project, `/community/:id/fork` requires only that the source is public or shared with you. Both require membership of the target org.

---

## 28. Vendor Migration Importer

Also management-plane. mudbase can pull an existing project in from another vendor (and from raw databases) so a customer does not have to hand-write a migration. You would call this from a console/import wizard, not from an application.

| Operation | Method + path | Auth |
|-----------|---------------|------|
| Test credentials | `POST /api/migration-import/test-connection` | bearer |
| Discover what's importable | `POST /api/migration-import/discover` | bearer |
| Start the import | `POST /api/migration-import/start/:projectId` | bearer, owner/admin, project access |
| Job status | `GET /api/migration-import/:jobId` | bearer |

`test-connection` and `discover` both take `{ vendor, credentials }`; `start` additionally takes `{ selection, estimatedBytes }` and returns `201 { message, jobId, status }`. The import itself runs on a queue (`migrationImportWorker`), so `jobId` is the only handle you get — poll it.

The credentials are the customer's own vendor keys. They are used for the duration of the request (test/discover) or held encrypted-at-rest on the job (start), and never logged. Build the wizard so those credentials are entered once, submitted directly, and not retained in your own client state or storage after the job is created.

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

---

## 29. Payment Links

A payment link is a hosted, tokenised checkout for a stablecoin payment. You create it org-side; your customer opens it in a browser with no account and no auth.

| Operation | Method + path | Auth |
|-----------|---------------|------|
| Create | `POST /api/orgs/:orgId/payment-links` | bearer, owner/admin, **KYC-approved** |
| List | `GET /api/orgs/:orgId/payment-links` | bearer, `payment:read` |
| Cancel | `POST /api/orgs/:orgId/payment-links/:linkId/cancel` | bearer, owner/admin |
| Public checkout read | `GET /api/payment-links/:token` | none |

Create takes `{ amount, currency, network, description, redirectUrl, expiresInHours }` — `currency` and `network` are required; an open-amount link is created by omitting `amount`. It returns `201 { link }`.

KYC is re-checked live at creation time, not at the point the org first enabled stablecoin payments, because a link points a stranger at real money. Handle `403 KYC_REQUIRED` (§26) on this endpoint specifically.

The public read returns a deliberately narrowed DTO — `token`, `amount`, `currency`, `network`, `address`, `description`, `redirectUrl`, `status`, `expiresAt`. No org id, no internal `_id`. Your checkout page should render from exactly those fields.

```typescript
// src/lib/mudbase.ts — inside class MudbaseClient
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
// src/lib/mudbase.ts — types
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

/** Fields the unauthenticated checkout endpoint returns — no org id, no internal _id. */
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

Payment confirmation is detected by the wallet indexers (§21.2), not synchronously at checkout — so a checkout page should poll the public read, or subscribe to the wallet events in §15, rather than assuming the payment is settled the moment the customer says they sent it.

---

## 30. Wallets — Broadcasting Transactions

Three distinct surfaces, with genuinely different trust models. Picking the wrong one is an architecture mistake, not a routing detail. For the real-time side of wallets — deposit and confirmation events — see §15; this section covers only the outbound paths.

| Surface | Route | Who holds the key | KYC gate |
|---------|-------|-------------------|----------|
| Custodial withdrawal | `POST /api/wallet/:walletId/withdraw` | mudbase | yes |
| Non-custodial broadcast | `POST /api/wallet/non-custodial/broadcast` | your user | yes |
| Account abstraction (EIP-4337) | `POST /api/wallet/non-custodial/account-abstraction/broadcast` | your user's smart account | yes |
| Stateless relay | `POST /api/tx/broadcast` | whoever signed it | yes |

### Custodial withdrawal

mudbase generated and holds the key; you ask it to send. Body is `{ toAddress, amount, network, options }`; auth is a user bearer token, and the wallet must belong to that user. Layered behind withdrawal-specific security middleware, tenant isolation, and a dedicated rate limiter on top of the KYC gate.

Use this when your product's users should not have to manage keys at all — and accept that you are then operating a custodial service, with everything that implies.

### Non-custodial broadcast

Your user signs locally; mudbase only relays and then tracks. Body is `{ chain, signedTx, fromAddress }` — `fromAddress` is required, because it is what links the broadcast to a registered address for later tracking, speed-up and cancel. Register addresses first via `POST /api/wallet/non-custodial/register-address`.

Companion endpoints: `POST /estimate-gas` before signing, `POST /speed-up` and `POST /cancel` to obtain replacement-transaction params for a stuck EVM transaction.

The account-abstraction variant takes the same trust model up a level for smart-contract accounts.

### Stateless relay

`POST /api/tx/broadcast` is the thinnest surface: `{ chain, signedTx, fromAddress?, projectId? }`, no wallet registration required, accepts a user bearer token *or* an API key. It answers "get this signed blob onto this chain" and nothing else. `fromAddress` is optional and used only to associate the broadcast with a registered wallet if one matches.

It is idempotent when you send `X-Idempotency-Key` — do, because a retried broadcast without one is a genuine double-spend risk on chains that accept the same signed transaction twice.

```typescript
// src/lib/mudbase.ts — inside class MudbaseClient
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
// src/lib/mudbase.ts — types
export interface BroadcastResult {
  txHash: string
  status: 'broadcast'
}
```

That method needs one addition to the `request()` helper — an optional per-call header bag:

```typescript
// src/lib/mudbase.ts — amend the options parameter of request()
  private async request<T>(
    method: string,
    path: string,
    body?: unknown,
    options: { auth?: boolean; formData?: FormData; headers?: Record<string, string> } = {}
  ): Promise<T> {
    const headers: Record<string, string> = { ...options.headers }
    // …rest of the method unchanged
```

Broadcasting is where a `403 KYC_REQUIRED` most commonly surprises people: the org is verified for its dashboard but a *different* org's key is in play, or verification lapsed. Always branch on the error `code`.

---

## 31. In-App Credit Balance

Distinct from the org's subscription billing. Credit is a **prepaid, spend-only** balance: you top it up, and it is drawn down automatically against overage invoices, capacity purchases and per-call add-on invocations (§25). It is deliberately not a wallet — there is no withdrawal, no transfer, and no refund path.

| Operation | Method + path | Auth |
|-----------|---------------|------|
| Balance | `GET /api/billing/credit/balance` | bearer |
| Ledger history | `GET /api/billing/credit/transactions` | bearer |
| Start a top-up | `POST /api/billing/credit/topup` | bearer |
| Verify a top-up | `POST /api/billing/credit/verify-payment?tx_ref=…` | bearer, org owner |
| Set low-balance threshold | `PUT /api/billing/credit/low-balance-threshold` | bearer, owner/admin |

All amounts are **integer cents**. Minimum top-up is 1000 (`$10`); the console pre-fills that amount. The low-balance alert threshold defaults to 200 (`$2`), is org-configurable, `0` disables it, and the ceiling is 100000 (`$1,000`) — above that the warning could never fire usefully.

Top-up returns a hosted checkout `link`; redirect the user there, then verify on return with the `tx_ref` you were given. Verification is idempotent — a duplicate call reports success without double-crediting.

```typescript
// src/lib/mudbase.ts — inside class MudbaseClient
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
// src/lib/mudbase.ts — types
export interface CreditBalance {
  balanceCents: number
  currency: string
  lowBalanceThresholdCents: number
}

export interface CreditTopup {
  /** Hosted checkout URL — redirect the user here. */
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

Do not render the balance as a spendable wallet or offer a "withdraw" affordance — there is no such endpoint, and presenting it that way misrepresents what the customer bought.

