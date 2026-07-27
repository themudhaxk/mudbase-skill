# mudbase-skill

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](./LICENSE)

A [Claude Skill](https://docs.claude.com/en/docs/claude-code/skills) that teaches a coding agent everything it needs to build a client application on top of [Mudbase](https://mudbase.dev) — a backend-as-a-service platform providing multi-tenant projects, JSON-schema collections, file storage, Socket.IO realtime, serverless functions, multi-chain crypto wallets, and more.

Drop `SKILL.md` into your project (or your global skills directory) and any coding agent that reads it will already know the SDK client shape, every auth flow, the full Socket.IO event catalog, webhook signature verification, error handling conventions, and the newer platform surfaces (MCP server, add-ons, KYC, payment links, wallet broadcasting) — without you having to paste API docs into every prompt.

## Installation

### Claude Code

Clone straight into your skills directory so it's picked up automatically:

```bash
git clone https://github.com/themudhaxk/mudbase-skill.git ~/.claude/skills/mudbase
```

Or clone it anywhere and reference the path from your project's `.claude/skills/` directory — a symlink works too:

```bash
git clone https://github.com/themudhaxk/mudbase-skill.git /path/to/mudbase-skill
ln -s /path/to/mudbase-skill ~/.claude/skills/mudbase
```

Claude Code loads any `SKILL.md` it finds under a skills directory and will surface this one automatically when you're building against Mudbase.

### Any other coding agent

This isn't Claude Code-specific. `SKILL.md` is a plain Markdown file — any agent or tool that can be pointed at project context works:

- **Cursor / Windsurf**: add it to your rules/context directory, or `@`-reference the file in a prompt.
- **Anything else**: paste the contents of `SKILL.md` directly into your system prompt or the start of a conversation. It's self-contained — no external references required.

## What's covered

1. mudbase SDK Integration
2. Authentication Patterns
3. Collections & Data
4. File Storage
5. Real-time WebSocket Events
6. Serverless Functions
7. Multi-Role System
8. API Keys Management
9. Environment Variables
10. Security Checklist
11. Socket.IO — Connection & Setup
12. Database Events (Socket.IO — Expanded)
13. Chat & Messaging
14. Presence
15. Wallets & Blockchain
16. Integrations Framework
17. Custom Channels
18. Calls & Signaling
19. Server-Sent Events (SSE)
20. Webhooks (Outbound — mudbase → your server)
21. Background Processing — what actually runs where
22. Error Handling & Retry Logic
23. Rate Limiting — Client-Side Strategies
24. MCP Server (for AI coding agents)
25. Add-ons Marketplace
26. Identity Verification (KYC / KYB)
27. Project Sharing, Cloning & Forking
28. Vendor Migration Importer
29. Payment Links
30. Wallets — Broadcasting Transactions
31. In-App Credit Balance

## Docs & product

- Full reference documentation: [docs.mudbase.dev](https://docs.mudbase.dev)
- Product / sign up: [mudbase.dev](https://mudbase.dev)

## Contributing

This skill is generated from, and kept in sync with, the real Mudbase API. If something in `SKILL.md` looks wrong, out of date, or missing compared to what `docs.mudbase.dev` describes, please [open an issue](https://github.com/themudhaxk/mudbase-skill/issues) — pull requests fixing drift are welcome too.

## License

[MIT](./LICENSE)
