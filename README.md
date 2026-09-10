# mudbase-skill

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](./LICENSE)

A [Claude Skill](https://docs.claude.com/en/docs/claude-code/skills) that teaches a coding agent everything it needs to build a client application on top of [Mudbase](https://mudbase.dev), a backend-as-a-service platform providing multi-tenant projects, JSON-schema collections, file storage, Socket.IO realtime, serverless functions, multi-chain crypto wallets, and more.

Drop `SKILL.md` (and its `references/` directory) into your project, or your global skills directory, and any coding agent that reads it will already know the SDK client shape, every auth flow (including SSO), the full Socket.IO event catalog, webhook signature verification, error handling conventions, and the newer platform surfaces (MCP server, GraphQL, add-ons, KYC, payment links, wallet broadcasting), without you having to paste API docs into every prompt.

`SKILL.md` itself is a short map: it points to the specific `references/*.md` file for whatever you're building, so an agent loads only what's relevant instead of one giant file.

## Installation

### Claude Code

Clone straight into your skills directory so it's picked up automatically:

```bash
git clone https://github.com/themudhaxk/mudbase-skill.git ~/.claude/skills/mudbase
```

Or clone it anywhere and reference the path from your project's `.claude/skills/` directory, a symlink works too:

```bash
git clone https://github.com/themudhaxk/mudbase-skill.git /path/to/mudbase-skill
ln -s /path/to/mudbase-skill ~/.claude/skills/mudbase
```

Claude Code loads any `SKILL.md` it finds under a skills directory and will surface this one automatically when you're building against Mudbase.

### Any other coding agent

This isn't Claude Code-specific. `SKILL.md` is a plain Markdown file, any agent or tool that can be pointed at project context works:

- **Cursor / Windsurf**: add it to your rules/context directory, or `@`-reference the file in a prompt.
- **Anything else**: paste the contents of `SKILL.md` directly into your system prompt or the start of a conversation, then paste the relevant `references/*.md` file(s) as needed. Nothing requires Claude Code specifically.

## What's covered

`SKILL.md` covers the SDK quick start, environment variables, and the security checklist, then points into these reference files:

1. `references/sdk-client.md`, the full typed REST client, types, and error class
2. `references/auth.md`, local auth, OAuth, magic link, OTP, anonymous auth, 2FA enrollment, and SSO (OIDC + SAML)
3. `references/collections-and-data.md`, schema design, the REST data contract, React Query hooks
4. `references/storage.md`, direct upload and presigned uploads
5. `references/realtime.md`, Socket.IO connection, live collection events, presence, custom channels, SSE
6. `references/messaging.md`, real-time chat, and the REST messaging API (email/SMS/push) with plan quotas
7. `references/functions.md`, serverless functions and the async execution model
8. `references/access-control.md`, the multi-role system and API key management
9. `references/wallets-and-payments.md`, wallet events, payment links, broadcasting, merchant fees, credit balance
10. `references/integrations-and-webhooks.md`, the integrations framework, calls/signaling, outbound webhooks
11. `references/error-handling-and-limits.md`, error codes, retry/backoff, rate limit buckets
12. `references/mcp-and-ai-agents.md`, the built-in MCP server and the standalone `mudbase-mcp-server` package
13. `references/addons-and-kyc.md`, the add-ons marketplace and identity verification (KYC/KYB)
14. `references/migration-import.md`, project sharing/cloning/forking, and the vendor migration importer
15. `references/graphql.md`, the auto-generated per-project GraphQL schema (read-only in this phase)

## Docs & product

- Full reference documentation: [docs.mudbase.dev](https://docs.mudbase.dev)
- Product / sign up: [mudbase.dev](https://mudbase.dev)

## Contributing

This skill is generated from, and kept in sync with, the real Mudbase API. If something in `SKILL.md` or `references/` looks wrong, out of date, or missing compared to what `docs.mudbase.dev` describes, please [open an issue](https://github.com/themudhaxk/mudbase-skill/issues). Pull requests fixing drift are welcome too.

## License

[MIT](./LICENSE)
