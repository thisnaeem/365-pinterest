# 365 Pinterest

Pinterest automation: schedule pins, manage boards, write AI blogs and auto-pin them.

**Live site:** https://365pinterest.com

Connect one or more Pinterest accounts, plan pins on a 365-day calendar and queue, bulk-upload pins, and let auto-post publish them on schedule. Connect a WordPress site and Workers AI writes the blog post, publishes it to WordPress, and auto-pins it to Pinterest. Built-in analytics show impressions, pin clicks, outbound clicks, and saves. A REST API and MCP server (12 tools) let developers and AI assistants manage Pinterest programmatically.

## Features

- **Multi-account Pinterest** — connect several Pinterest accounts (OAuth), switch between them from the sidebar
- **Boards** — browse boards with pin covers and thumbnails
- **Create & schedule pins** — composer with a visual board picker, scheduling on a 365-day calendar and queue
- **Bulk upload** — upload many pins at once
- **Auto-post** — a scheduled worker publishes due pins every 5 minutes, with safe pacing
- **WordPress + AI blog** — connect a WordPress site (application password, encrypted at rest); Workers AI writes the post, publishes it, and auto-pins it
- **Analytics** — impressions, pin clicks, outbound clicks, saves per account
- **REST API** (`/api/v1/`) and **MCP server** (`/mcp`, 12 tools) — API keys at `/app/api-keys`; listed on the [MCP Registry](https://registry.modelcontextprotocol.io/v0/servers?search=365-pinterest)
- **Billing** — Polar checkout; see [Pricing](#pricing)

## Pricing

- **Free — $0:** 1 Pinterest account, 15 scheduled pins/month
- **Pro — $5/month or $50/year:** unlimited accounts, unlimited scheduled pins, bulk upload, WordPress AI blog, REST API + MCP access

## Get started

Sign up at https://365pinterest.com, connect your Pinterest account, and schedule your first pin. Developers: create an API key at https://365pinterest.com/app/api-keys and read the [REST API docs](docs/api.md) and [MCP docs](docs/mcp.md).

## Stack

- Next.js 15 (App Router) + TypeScript + Tailwind CSS 4
- Prisma 7 + Neon Postgres (HTTP driver on Workers)
- better-auth (email/password + Google + "Continue with Pinterest")
- Cloudinary (pin image hosting)
- Cloudflare Workers via OpenNext (`@opennextjs/cloudflare`)
- Polar (billing), Workers AI (blog writing + cover images)

## Docs

- [docs/api.md](docs/api.md) — REST API reference (`/api/v1/`)
- [docs/mcp.md](docs/mcp.md) — MCP server: endpoint, auth, and all 12 tools
- [docs/oauth-mcp.md](docs/oauth-mcp.md) — OAuth 2.0 server for the Claude Connectors Directory
- [mcp-registry/server.json](mcp-registry/server.json) — MCP Registry publication packet

## License

MIT — see [LICENSE](LICENSE).
