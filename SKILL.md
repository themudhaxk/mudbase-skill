---
name: mudbase
description: Complete integration guide for building client apps on Mudbase (backend as a service). Covers the typed SDK client, every auth flow (local, OAuth, magic link, OTP, anonymous, 2FA, SSO), collections CRUD with React Query, file storage, Socket.IO realtime (chat, presence, calls), GraphQL, serverless functions, webhooks, the MCP server for AI agents, add-ons, KYC, payment links, and merchant payment processing.
---

# Mudbase

Mudbase (`mudbase.dev`) is a backend-as-a-service platform: multi-tenant projects, JSON-schema collections with role-based permissions, file storage, Socket.IO realtime, serverless functions, an auto-generated read-only GraphQL schema, merchant payment processing and stablecoin payment links, and a REST API at `cloud.mudbase.dev`. This skill teaches an AI coding agent, or any developer, everything needed to build a client application against an existing Mudbase project.

Full reference documentation lives at https://docs.mudbase.dev.

## How this skill is organized

This file is a map, not the manual. Read it first, then open the specific `references/*.md` file(s) that cover what you're building. Do not try to hold the whole platform in context at once, load only the reference relevant to the current task.

| File | Covers |
|---|---|
| `references/sdk-client.md` | The full typed REST client (`MudbaseClient`), request/response shapes, base types, error class |
| `references/auth.md` | Local auth, OAuth, magic link, OTP, anonymous auth, 2FA/TOTP enrollment, and SSO (OIDC + SAML) |
| `references/collections-and-data.md` | Collection schema design, the REST data contract, React Query hooks, why `$regex` is blocked |
| `references/storage.md` | Direct file upload, presigned uploads, the two storage surfaces and when to use each |
| `references/realtime.md` | Socket.IO connection/rooms, live collection events, presence, custom channels, SSE |
| `references/messaging.md` | Real-time chat over Socket.IO, and the REST messaging API (email, SMS, push) plus plan quotas |
| `references/functions.md` | Serverless functions, the async execution/polling model, function vs. client-side logic |
| `references/access-control.md` | The multi-role system and API key management, including scoped least-privilege keys |
| `references/wallets-and-payments.md` | Payment links (stablecoin checkout), merchant payment processing fees, in-app credit balance. Mudbase's own crypto wallets and transaction broadcasting were archived to MudChain, a separate product, and are not covered here |
| `references/integrations-and-webhooks.md` | The integrations framework, calls/signaling, and outbound webhooks (config, signature verification, retries) |
| `references/error-handling-and-limits.md` | HTTP error codes, retry/backoff, rate limit buckets, client-side request queuing |
| `references/mcp-and-ai-agents.md` | The backend's built-in MCP server (38 tools) and the standalone `mudbase-mcp-server` package |
| `references/addons-and-kyc.md` | The add-ons marketplace (metered utilities) and identity verification (platform KYC + white-label KYC/KYB) |
| `references/migration-import.md` | Project sharing/cloning/forking, and the vendor migration importer (what relationship import actually supports) |
| `references/graphql.md` | The auto-generated per-project GraphQL schema (read-only in this phase), query shape, and hardening limits |

## Quick start: the SDK client

Mudbase has no official npm SDK yet. Use the REST API directly with a typed client, install one dependency (or use native `fetch`, no dependency needed):

```bash
npm install axios
# or use native fetch, no external dep needed
```

The full client, `MudbaseClient`, its types, and the `MudbaseError` class are in `references/sdk-client.md`. Start there before any other reference file, everything else assumes that client (or an equivalent) exists.

## Environment variables

```bash
# .env.local

# mudbase
NEXT_PUBLIC_MUDBASE_PROJECT_ID=your_project_id
NEXT_PUBLIC_MUDBASE_URL=https://cloud.mudbase.dev

# Server-side only (never NEXT_PUBLIC_)
MUDBASE_URL=https://cloud.mudbase.dev
MUDBASE_PROJECT_ID=your_project_id
MUDBASE_API_KEY=ak_...                            # project API key for server-side reads (multi-role config, etc.)
MUDBASE_ADMIN_TOKEN=your_org_owner_bearer_token   # only needed for API-key/billing management calls
MUDBASE_WEBHOOK_SECRET=your_webhook_signing_secret
```

## Security checklist

- API keys and service account tokens never in the client bundle, only in server-side env vars.
- **Token storage, pick deliberately.** Mudbase gives you no cookie option: every auth endpoint returns the bearer token in a JSON body, there is no `Set-Cookie`. Three honest choices:
  - **Browser, default:** `sessionStorage`, not `localStorage`. Both are XSS-readable, but `sessionStorage` clears on tab close and is not shared across tabs. Reach for `localStorage` only if "stay signed in across browser restarts" is a real requirement.
  - **Browser, higher assurance:** proxy authentication through your own backend. Your server calls the mudbase auth endpoint, keeps the bearer token server-side, and sets your *own* `httpOnly; Secure; SameSite=Lax` cookie on the browser. The mudbase token never reaches client JavaScript. This is the right default for anything handling money, health data, or admin capability.
  - **Mobile (React Native / Expo):** `expo-secure-store` or `react-native-keychain`. Never `AsyncStorage`, it is unencrypted plaintext on disk.
- Anonymous sessions: always convert before storing sensitive user data (PII, payment info). See `references/auth.md`.
- File buckets: set access policy correctly, private for user uploads, public only for truly public assets. See `references/storage.md`.
- Real-time channels: always scope to the authenticated user's own data. Subscribe to `user:{userId}:*` or filter by your own userId, never a bare `collection:{id}` room for all users. See `references/realtime.md`.
- OAuth and SSO: save the return path before redirecting, validate the session server-side after callback, never trust a token from the URL alone. See `references/auth.md`.
- Functions: validate every payload field on the function side, never trust client-provided IDs or roles. See `references/functions.md`.
- Never expose `projectId` in combination with an admin-level API key, use scoped keys with the minimum required permissions. See `references/access-control.md`.
- Password reset and magic link tokens are one-time use; project-scoped resets are OTP-based (6 digits, 10-minute TTL), not link-token based. See `references/auth.md`.
- **2FA:** mudbase's login endpoint does not gate on 2FA, it returns a usable token regardless of `twoFactorEnabled`. If your product promises a second factor at login, enforce it in your own backend proxy, do not present `user.twoFactorEnabled` to users as though the server were blocking on it. See `references/auth.md`.
- Rate limit auth endpoints on the server, mudbase handles this internally, but add client-side debounce on forms. See `references/error-handling-and-limits.md`.
- Content Security Policy: if loading files from mudbase storage via signed URLs, add the storage domain to your CSP `img-src` / `media-src` directives.
- Webhooks: always verify the delivery signature before trusting a payload, and check the signature length before comparing to avoid a timing side channel. See `references/integrations-and-webhooks.md`.
- Never expose which third-party service backs a Mudbase capability (payments, email/SMS delivery, AI features, storage/CDN) in anything user-facing, present it as a first-party Mudbase capability.

## What's out of scope for a client app

A few areas of the platform are **management-plane**, called from an admin console or provisioning script, not from an application you build on top of a project:

- Project sharing, cloning, forking, and the vendor migration importer, see `references/migration-import.md`.
- Org-level billing and API-key issuance beyond what a signed-in admin does in your own admin UI, see `references/access-control.md`.

If you find yourself reaching for these from inside a normal end-user-facing app, stop and check whether the operation actually belongs in your own internal tooling instead.

## Keeping this skill accurate

This skill is generated from, and should be kept in sync with, the real Mudbase API. Two features are commonly overclaimed, watch for stale claims about either:

- GraphQL is read-only in this phase, no mutations or subscriptions (`references/graphql.md`).
- The vendor migration importer only converts foreign keys into real Mudbase relationships for Postgres, MySQL, and Supabase sources, not MongoDB, Firebase, Clerk, or Appwrite (`references/migration-import.md`).
