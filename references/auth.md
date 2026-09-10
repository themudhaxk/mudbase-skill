# Authentication Patterns

Covers every auth flow: local (email/password), OAuth, magic link, OTP (phone), anonymous, two-factor (TOTP), and SSO (OIDC/SAML). The typed client these hooks call is in `sdk-client.md`.

## 1. Local Auth, `src/hooks/useAuth.ts`

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
        // Treat the flag as informational (e.g. re-prompt before privileged actions),
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

## 2. OAuth (Social Login)

```typescript
// src/lib/oauth.ts

const MUDBASE_URL = process.env.NEXT_PUBLIC_MUDBASE_URL ?? 'https://cloud.mudbase.dev'
const PROJECT_ID = process.env.NEXT_PUBLIC_MUDBASE_PROJECT_ID
if (!PROJECT_ID) {
  throw new Error('NEXT_PUBLIC_MUDBASE_PROJECT_ID must be set')
}

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

## 3. Magic Link

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
      setError('Invalid magic link, missing token.')
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

## 4. OTP (Phone Auth)

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

## 5. Anonymous Auth + Convert

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

Use cases: guest checkout flow (start anonymous, add to cart, checkout, convert), trial access (start anonymous, use features, prompt to sign up on limit).

## 6. Two-Factor Authentication (TOTP)

**Read this before designing the flow.** mudbase's TOTP endpoints live at `POST /api/users/2fa/setup`, `/verify`, `/disable` and all three require a valid bearer token (`authRequired`). They are an *enrolment* surface for an already-signed-in user, not a login challenge:

- `POST /api/auth/local/login` returns a full, immediately usable `token` regardless of whether the account has 2FA enabled. It reports the state as `user.twoFactorEnabled` and nothing more, there is no `requires2FA` flag and no partial/pre-auth token.
- `POST /api/users/2fa/verify` takes `{ token: "123456" }` (the TOTP digits, the field is named `token`, not `code`) and, on success, flips `twoFactorEnabled` to true. It does **not** mint or upgrade a session.
- `POST /api/users/2fa/disable` takes `{ password, token }`, both the account password and a live TOTP code.

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
      // twoFactorEnabled is part of the session payload, refresh so the UI reflects it.
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
      {/* `qrCode` is already a data: URI PNG rendered server-side, no QR library, no third-party image host. */}
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
// src/components/auth/TwoFADisable.tsx, requires the account password AND a live code
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

## 7. Single Sign-On (SSO: OIDC + SAML)

Shipped and customer-facing, gated to the Growth plan and above. An org on a lower plan that tries to configure a connection gets a plain-language plan error ("Enterprise SSO is not available on this plan. It is included on Growth, Scale, and Enterprise."), so surface that message directly rather than a generic 403 in your admin UI.

SSO is an **alternative front door into the same session system** described in sections 1 to 6 above, not a separate user model. A user who signs in through SSO ends up with the same bearer token shape and the same `getSession()` contract as local auth; the difference is only how the session gets minted.

Routes are mounted at `/api/auth/sso`:

| Operation | Method + path | Notes |
|-----------|---------------|-------|
| Discover a project's connection | `GET /api/auth/sso/discover` | Rate-limited, typically called from your login page to decide whether to show an "SSO" button and for which identity provider |
| Start the flow (OIDC) | `GET /api/auth/sso/:projectId/start` | Redirect the browser here; it forwards to the identity provider |
| Callback | `GET /api/auth/sso/callback` | The identity provider redirects back here; mudbase completes the exchange and issues a session |
| List connections | `GET /api/auth/sso/projects/:projectId/connections` | Admin/console surface, bearer token with project access |
| Create a connection | `POST /api/auth/sso/projects/:projectId/connections` | `{ protocol: 'oidc' \| 'saml', ... }` |
| Read one connection | `GET /api/auth/sso/projects/:projectId/connections/:connectionId` | |
| Update a connection | `PATCH /api/auth/sso/projects/:projectId/connections/:connectionId` | |
| Delete a connection | `DELETE /api/auth/sso/projects/:projectId/connections/:connectionId` | |

Both **OIDC and SAML** are supported protocols on a connection, configured through the same connection CRUD. There is also a domain-verification flow (a DNS TXT-record challenge) for binding a connection to an email domain, so mudbase can route a user straight to the right identity provider from their email address alone without asking them to pick one.

```typescript
// src/lib/sso.ts

const MUDBASE_URL = process.env.NEXT_PUBLIC_MUDBASE_URL ?? 'https://cloud.mudbase.dev'
const PROJECT_ID = process.env.NEXT_PUBLIC_MUDBASE_PROJECT_ID

export interface SsoDiscoveryResult {
  available: boolean
  connectionId?: string
  protocol?: 'oidc' | 'saml'
  displayName?: string
}

/** Call before rendering the login page's SSO button, to know whether to show it at all. */
export async function discoverSso(email?: string): Promise<SsoDiscoveryResult> {
  const qs = email ? `?email=${encodeURIComponent(email)}` : ''
  const res = await fetch(`${MUDBASE_URL}/api/auth/sso/discover${qs}`)
  if (!res.ok) return { available: false }
  return (await res.json()) as SsoDiscoveryResult
}

/** Redirect the browser into the identity provider's hosted login. */
export function startSsoLogin(): void {
  if (!PROJECT_ID) throw new Error('NEXT_PUBLIC_MUDBASE_PROJECT_ID must be set')
  const redirectUrl = `${window.location.origin}/auth/callback`
  window.location.href = `${MUDBASE_URL}/api/auth/sso/${PROJECT_ID}/start?redirect_url=${encodeURIComponent(redirectUrl)}`
}
```

The callback lands the browser back on your own `/auth/callback` route with a `token` query parameter, the same OAuth callback page shown in section 2 above handles it without changes: it reads `token`, calls `client.setToken()`, confirms with `getSession()`, and redirects. Build one callback page and reuse it for OAuth and SSO alike.

Admin connection management (creating and editing a connection's OIDC issuer/client credentials or SAML metadata) is a console/admin-plane operation, not something an end-user-facing app calls; treat it the same way as the API key management endpoints in `access-control.md`, behind an authenticated owner/admin screen, never from a public page.

## See also

- `sdk-client.md`, the `MudbaseClient` methods these hooks call, and the token-storage security checklist
- `access-control.md`, multi-role signup (role selection alongside these auth flows) and API keys
