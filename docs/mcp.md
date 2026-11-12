# 365 Pinterest — MCP Server

365 Pinterest exposes a **Model Context Protocol (MCP) server** at `https://365pinterest.com/mcp`, so AI assistants (Claude, and any MCP client) can manage Pinterest for you: list boards, create and schedule pins, check the queue and analytics, and write and publish WordPress blog posts that auto-pin to Pinterest.

- **Endpoint:** `https://365pinterest.com/mcp` (streamable HTTP)
- **Registry:** listed on the [MCP Registry](https://registry.modelcontextprotocol.io/v0/servers?search=365-pinterest) as `com.365pinterest/365-pinterest`
- **Auth:** API key (Pro plan) or OAuth 2.0 — see [docs/oauth-mcp.md](oauth-mcp.md) for the OAuth flow
- **Full interactive docs:** https://365pinterest.com/developers

## Authentication

Create an API key in the dashboard at https://365pinterest.com/app/api-keys (Pro plan). Keys look like `pk_live_...` and are shown once — store yours safely. Send it as a Bearer token:

```bash
curl -X POST https://365pinterest.com/mcp \
  -H "Authorization: Bearer pk_live_YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list","params":{}}'
```

Rate limits: 60 requests/minute per key. `create_pin` is 30/minute per key and `connect_wordpress` is 10/minute per key.

## Claude Desktop

Add this to your Claude Desktop config (`claude_desktop_config.json`):

```json
{
  "mcpServers": {
    "365-pinterest": {
      "command": "npx",
      "args": [
        "mcp-remote",
        "https://365pinterest.com/mcp",
        "--header",
        "Authorization: Bearer pk_live_YOUR_KEY"
      ]
    }
  }
}
```

## Claude Code

```bash
claude mcp add --transport http 365-pinterest https://365pinterest.com/mcp --header "Authorization: Bearer pk_live_YOUR_KEY"
```

## Tools (12)

| Tool | Access | What it does |
| --- | --- | --- |
| `account_status` | Read | Connection status, username, scopes, board count. Optional `account_id`. |
| `list_boards` | Read | List boards (id, name, description, privacy, pin count). Optional `account_id`, `limit` (default 100, max 250). |
| `create_pin` | Write | Create a draft pin, or a scheduled pin with `scheduledFor`. Requires `title` and `imageUrl`; `boardId` or `pinterestBoardId` picks the board. Optional `account_id`. |
| `schedule_pin` | Write | Schedule an existing draft pin for a future time. Requires `pinId` and `scheduledFor` (ISO datetime). |
| `list_pins` | Read | List pins, filtered by `status` (DRAFT, SCHEDULED, POSTING, PUBLISHED, FAILED) and/or `boardId`. Optional `account_id`, `limit` (default 50, max 100). |
| `get_queue` | Read | Up-next scheduled pins, ordered by scheduled time. Optional `account_id`, `limit` (default 50, max 100). |
| `get_analytics` | Read | Last-30-days totals: impressions, saves, clicks, outbound clicks. Optional `account_id`. |
| `wordpress_sites` | Read | List connected WordPress sites. App passwords are never returned. |
| `connect_wordpress` | Write | Connect a WordPress site with `siteUrl`, `username`, and `appPassword`. Verified against the live WordPress API first. |
| `write_blog_post` | Write | Generate a blog draft with Workers AI (Llama 3.3 70B) for a connected site. `siteId` and `topic` required; `keywords`, `tone` (friendly, professional, storytelling), `length` (short, medium, long) optional. |
| `publish_blog_post` | Write | Publish a draft to WordPress and create its Pinterest pin. `postId` required; `boardId` or `pinterestBoardId`, `status` (publish or draft), `imageUrl`, and edited draft fields optional. No image is generated through this tool — pass `imageUrl`, or use the draft's stored cover image. |
| `list_blog_posts` | Read | List blog posts, newest first. Optional `siteId` and `limit` (default 50, max 100). |

Every tool accepts an optional `account_id` (a connected Pinterest account id). Omit it to use the active account.

## Notes

- Creating a pin never publishes it directly: a future `scheduledFor` makes it SCHEDULED (auto-post publishes it at the due time); otherwise it is a DRAFT.
- `publish_blog_post` without a board and an image still publishes the WordPress post, but no pin is made.
- Unauthenticated requests to `/mcp` return 401 with a `WWW-Authenticate` header pointing at the OAuth protected-resource metadata.
