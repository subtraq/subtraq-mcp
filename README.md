# Subtraq MCP server

Remote [Model Context Protocol](https://modelcontextprotocol.io) server for [Subtraq](https://subtraq.co):
short links, click tracking and sales attribution. Your AI agent creates links, reads attribution and
records sales — eight tools backed by the Subtraq API.

| | |
|---|---|
| **Endpoint** | `https://subtraq.co/api/mcp` |
| **Transport** | Streamable HTTP (JSON-RPC 2.0) |
| **Authentication** | `Authorization: Bearer <Subtraq API key>` — keys look like `stq_sk_…` |
| **MCP Registry name** | [`co.subtraq/subtraq`](https://registry.modelcontextprotocol.io/v0/servers?search=co.subtraq/subtraq) |
| **Documentation** | [subtraq.co/en/mcp](https://subtraq.co/en/mcp) · [subtraq.co/en/developers](https://subtraq.co/en/developers) |

It is a hosted server: there is nothing to install, no SDK, no local process. This repository holds the
server's public description (`server.json`, OpenAPI). The Subtraq application itself is not open source.

## Get an API key

1. Create an account at [subtraq.co/en/signup](https://subtraq.co/en/signup) — free plan, no credit card.
2. In Subtraq, open **Settings → API Keys** and create a key. It is shown once; Subtraq only keeps its
   SHA-256 fingerprint.
3. Give the key only the scopes your agent needs:

| Scope | Allows |
|---|---|
| `links:read` | List workspaces and links, detail a link with its final destination and its clicks over thirty days. |
| `links:write` | Create a workspace, a parent link or a placement, update a destination or a label, archive. |
| `analytics:read` | Read clicks, leads, sales, attributed and unattributed revenue, placement by placement. |
| `events:write` | Record a sale. The only scope that touches money, and it can only be exercised from a server. |

A key can also carry `*`, which opens all four. Revoking a key is immediate and applies to both the API
and the MCP server.

## Connect your agent

Any MCP client that supports remote streamable-HTTP servers with a custom header works.

**Claude Code**

```bash
claude mcp add --transport http subtraq https://subtraq.co/api/mcp \
  --header "Authorization: Bearer stq_sk_YOUR_KEY"
```

**JSON configuration** (for example a project `.mcp.json`)

```json
{
  "mcpServers": {
    "subtraq": {
      "type": "http",
      "url": "https://subtraq.co/api/mcp",
      "headers": { "Authorization": "Bearer stq_sk_YOUR_KEY" }
    }
  }
}
```

**Check it by hand**

```bash
curl -s https://subtraq.co/api/mcp \
  -H "Authorization: Bearer stq_sk_YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
```

The server speaks standard MCP — `initialize`, `tools/list`, `tools/call` — with no proprietary extension.

## The eight tools

| Tool | What it does | HTTP equivalent | Scope |
|---|---|---|---|
| `list_spaces` | Lists the account's workspaces. One workspace = one client, brand or project; everything else (links, clicks, sales) lives inside it. | `GET /api/v1/spaces` | `links:read` |
| `create_space` | Creates a client workspace. Refused with code `space_limit_reached` when the plan is full. | `POST /api/v1/spaces` | `links:write` |
| `list_links` | Lists links. A parent link carries the destination; a placement (sublink) carries the UTMs of one specific post and inherits the rest. | `GET /api/v1/links` | `links:read` |
| `create_link` | Creates a short link. Without `parentId` it is a parent link and `destination` is required. With `parentId` it is a placement: it inherits the parent's destination and carries its own UTMs. Returns `shortUrl`, ready to publish. | `POST /api/v1/links` | `links:write` |
| `get_link` | Details a link: its final destination (UTMs included, exactly as the landing site receives it) and its clicks over 30 days. | `GET /api/v1/links/{id}` | `links:read` |
| `update_link` | Changes a link's destination, label or address (`domain` and/or `slug`), or archives it. The old address keeps redirecting and its clicks count on this link (`previousShortUrls`). A link is never deleted: archived, it stops redirecting but its history stays. | `PATCH /api/v1/links/{id}` | `links:write` |
| `get_analytics` | A workspace's numbers: clicks, leads, sales, attributed and unattributed revenue, plus the detail placement by placement. `model` picks how attribution is read; the response always says which one was used. | `GET /api/v1/analytics` | `analytics:read` |
| `track_sale` | Records a sale and ties it back to the person's originating placement. `invoiceId` makes the call idempotent: replaying it never bills twice. | `POST /api/v1/sales` | `events:write` |

The MCP server and the v1 API share the same tool registry: what one exposes, the other exposes too.
The descriptions returned by `tools/list` are written in French — they are a contract shared with the API
and the OpenAPI file, so they do not change with the reader's language. The English summaries above are the
ones published on [subtraq.co/en/mcp](https://subtraq.co/en/mcp).

## Rules worth knowing

- **Amounts are in minor units** (cents): $43 is written `4300`. Subtraq does not convert currencies — if
  a workspace holds several, `mixedCurrencies` is `true` and totals only cover `currency`.
- **Lists use a cursor**: follow `nextCursor` until it is `null`.
- **Writes accept an `Idempotency-Key` header**; for a sale, `invoiceId` plays that role and is required.
- **Sales only come in from a server**, never from a browser.
- **An agent does exactly what its key permits**, within that key's workspace. No mass deletion is exposed:
  the tools create, read and update.

## Files in this repository

| File | Content |
|---|---|
| [`server.json`](server.json) | The server's entry in the official MCP Registry. |
| [`openapi.json`](openapi.json) | A copy of the OpenAPI 3.1 description of the v1 API. The live file at [`subtraq.co/api/v1/openapi.json`](https://subtraq.co/api/v1/openapi.json) is generated from the same tool registry and is the reference; its descriptions are in French. |

## Support

Questions or problems: [contact@subtraq.co](mailto:contact@subtraq.co).

## License

The contents of this repository are released under the [MIT License](LICENSE).
