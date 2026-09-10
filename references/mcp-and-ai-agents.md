# MCP Server (for AI coding agents)

Two separate MCP surfaces exist. Do not conflate them, they have different transports, different auth, and different tool sets.

| | Backend built-in `/mcp` | Standalone `mudbase-mcp-server` package |
|---|---|---|
| What it is | An endpoint on the mudbase backend itself | A separate npm package, run locally via `npx` |
| Transport | Streamable HTTP, stateless | stdio |
| Auth | Project API key (`X-API-Key` / `Bearer ak_...`) | `MUDBASE_API_KEY` environment variable |
| Plan gate | Paid plans only | None documented, follows whatever the API key can do |
| Tool count | 38 tools, full platform surface (collections, functions, webhooks, messaging, users, add-ons) | About 14 tools, narrower: collections (read-only schema), documents CRUD, search, storage |
| Best for | An agent building against a live mudbase project through the console-integrated flow | A minimal, dependency-light MCP client setup, e.g. wiring mudbase into a generic MCP host without the backend's own connection contract |

Neither is what your **application** calls at runtime, that is the REST/Socket.IO client in `sdk-client.md`. Both MCP surfaces are for an AI coding agent (Claude Code, Cursor, any Model Context Protocol client) connecting while *building* the application, so it can create collections, inspect schemas, deploy functions, and read data directly instead of guessing at your backend.

## 1. Backend built-in `/mcp`

### Connection contract

| | |
|---|---|
| Endpoint | `POST /mcp` on the org's API host (shared platform host, or the dedicated host for orgs with dedicated infrastructure) |
| Transport | Streamable HTTP, **stateless**, a fresh server and transport per request. `GET /mcp` and `DELETE /mcp` return `405`; there are no long-lived session streams |
| Auth | A project **API key**, sent as `X-API-Key: ak_...` or `Authorization: Bearer ak_...`. Not a user JWT |
| Plan gate | Paid plans only (`MCP_ALLOWED_PLANS`, default `starter,growth,scale,enterprise`). A free-tier key gets `402` with `code: "PLAN_REQUIRED"` |
| Scope | The key resolves to exactly one project and org. Write-capable tools are checked against the key's `permissions` grant (see `access-control.md`, section 2.2), so a read-only key cannot be talked into writing |
| Metering | Every MCP request is metered against the org's `apiCalls` quota, same as a REST call |

### The full tool list (38 tools)

Grouped by area, this is the exact, verified list of tool names the backend registers:

**Project**
- `get_project_info`, `get_project_usage`

**Collections & schema**
- `list_collections`, `get_collection_schema`, `create_collection`, `update_collection`, `delete_collection`

**Documents**
- `query_documents`, `get_document`, `create_document`, `update_document`, `delete_document`, `search_documents`

**Relationships** (read-only)
- `list_relationships`, `get_relationship`

**Storage**
- `list_files`, `upload_file`, `get_file_url`, `delete_file`, `list_buckets`, `create_bucket`

**Functions**
- `list_functions`, `invoke_function`, `get_function`, `create_function`, `update_function`, `delete_function`, `get_function_logs`

**Webhooks** (read-only)
- `get_webhook_config`, `list_webhook_deliveries`

**Messaging**
- `send_email`, `list_email_templates`, `send_sms`, `get_message_history`

**Users**
- `list_users`, `get_user`

**Add-ons**
- `list_addons`, `generate_addon`

This list is served live by `GET /mcp/config` as well; treat that endpoint as the source of truth if the platform adds tools after this file was last updated, this document reflects a real point-in-time snapshot, not a live feed.

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

The `endpoint` it returns is the URL to hand to the agent, resolve it rather than hardcoding `cloud.mudbase.dev`, because dedicated-infrastructure orgs run on their own host.

### Wiring an agent up

```bash
# Claude Code, scope the key to what the agent actually needs (see access-control.md, section 2.3)
claude mcp add --transport http mudbase https://cloud.mudbase.dev/mcp \
  --header "X-API-Key: ak_your_project_key"
```

Security note: an API key handed to an agent is a standing credential in that agent's config. Mint a dedicated key for it, grant the narrowest useful permission set, and rotate it (`POST /api/api-keys/:id/regenerate`) when the agent's access should end.

## 2. Standalone `mudbase-mcp-server` package

A separate, published npm package rather than a backend route. Run it with `npx`, no local install required:

```bash
npx mudbase-mcp-server
```

Configuration is entirely via environment variable, `MUDBASE_API_KEY`, no header wiring or endpoint URL needed, the package resolves the platform host itself.

### Tool list (about 14 tools)

A narrower, read-and-basic-write surface focused on collections and storage, not the full platform:

**Collections** (read-only schema)
- `mudbase_list_collections`, `mudbase_get_collection`

**Documents**
- `mudbase_list_documents`, `mudbase_get_document`, `mudbase_create_document`, `mudbase_update_document`, `mudbase_delete_document`, `mudbase_search_documents`

**Storage**
- `mudbase_list_buckets`, `mudbase_list_files`, `mudbase_get_file`, `mudbase_upload_file`, `mudbase_delete_file`, `mudbase_get_file_download_url`

Notably absent versus the backend's own 38-tool surface: functions, webhooks, messaging, users, add-ons, and collection create/update/delete (schema is read-only here). Reach for the backend built-in `/mcp` surface above when an agent needs any of those.

### Choosing between the two

- Building inside Claude Code or another agent that already speaks Streamable HTTP MCP and you want the full platform surface (functions, webhooks, messaging): use the backend's `/mcp`.
- Wiring mudbase into a simpler or stdio-only MCP host, or you only need collections/documents/storage: the standalone package is the lighter integration.
- Both authenticate with the same kind of project API key; scope it the same way regardless of which surface you point it at (see `access-control.md`, section 2.3, `AGENT_GRANT`).

## See also

- `access-control.md`, API key permission grants and the least-privilege pattern both MCP surfaces rely on
- `sdk-client.md`, the runtime client your application uses, distinct from either MCP surface above
