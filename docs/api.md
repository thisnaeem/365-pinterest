# 365 Pinterest — REST API

Base URL: `https://365pinterest.com/api/v1`

The REST API lets developers manage Pinterest pins, boards, queues, analytics, and WordPress blog posts programmatically. The same capabilities are also available through the [MCP server](mcp.md).

- **Auth:** `Authorization: Bearer pk_live_YOUR_KEY` (API key, Pro plan). Create a key at https://365pinterest.com/app/api-keys — keys are shown once.
- **Rate limit:** 60 requests/minute per key. `POST /pins` (create pin) is 30/minute per key.
- **Errors:** JSON `{ "error": "..." }` with the HTTP status. Unauthenticated requests return 401; unknown ids return 404; invalid state transitions (e.g. scheduling an already-published pin) return 409.
- **Full interactive docs:** https://365pinterest.com/developers

## Account

### `GET /api/v1/account`

Account status for the API-key owner.

Response: `{ connected, needsReconnect, username, pinterestUserId, scopes, boardCount }`

```bash
curl https://365pinterest.com/api/v1/account \
  -H "Authorization: Bearer pk_live_YOUR_KEY"
```

## Boards

### `GET /api/v1/boards`

List boards (cached copy when available, otherwise a live fetch). Optional `?account_id=` query parameter.

Response: `{ boards: [{ id, pinterestBoardId, name, description, privacy, pinCount }], cached }`

### `POST /api/v1/boards`

Create a board on Pinterest and cache it.

```bash
curl -X POST https://365pinterest.com/api/v1/boards \
  -H "Authorization: Bearer pk_live_YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{"name": "Weeknight dinners", "description": "Quick meals under 30 minutes", "privacy": "PUBLIC"}'
```

Body: `name` (required, max 180 characters), `description` (optional, max 500), `privacy` (optional: `PUBLIC`, `PROTECTED`, or `SECRET`). Response 201: `{ board }`.

## Pins

### `GET /api/v1/pins`

List pins. Query parameters: `?status=DRAFT|SCHEDULED|POSTING|PUBLISHED|FAILED`, `&boardId=<cached board id>`, `&limit=` (default 50, max 100), `&account_id=`.

Response: `{ pins, total, limit }`

### `POST /api/v1/pins`

Create a pin. Creation never publishes directly: a future `scheduledFor` makes it SCHEDULED (auto-post publishes it at the due time); otherwise it is a DRAFT.

JSON body:

```bash
curl -X POST https://365pinterest.com/api/v1/pins \
  -H "Authorization: Bearer pk_live_YOUR_KEY" \
  -H "Content-Type: application/json" \
  -d '{"title": "10 weeknight dinners", "description": "Quick meals under 30 minutes", "destinationUrl": "https://example.com/dinners", "boardId": "BOARD_ID", "imageUrl": "https://example.com/dinners.jpg", "scheduledFor": "2026-11-20T09:00:00Z"}'
```

Fields: `title` (required, max 100), `description` (optional, max 800), `destinationUrl` (optional), `boardId` or `pinterestBoardId` (one required), `imageUrl` (required for JSON), `scheduledFor` (optional ISO datetime).

Multipart form-data is also accepted: the same fields as text parts plus a `file` field (JPEG/PNG/WebP, max 10 MB) instead of `imageUrl`.

Response 201: `{ pin }`

### `POST /api/v1/pins/[id]/schedule`

Schedule a pin for a future time. Body: `{ scheduledFor }` (ISO datetime, required, must be in the future). Response: `{ pin }` with status SCHEDULED. Published or currently-publishing pins return 409.

### `POST /api/v1/pins/[id]/cancel`

Cancel a scheduled pin (SCHEDULED → DRAFT). No body needed. A pin that is not SCHEDULED returns 409. Response: `{ pin }` with status DRAFT and `scheduledFor` cleared.

## Queue

### `GET /api/v1/queue`

Up-next scheduled pins, ordered by `scheduledFor`. Query: `?limit=` (default 50, max 100), `&account_id=`.

Response: `{ queue: [{ ...pin, boardName }], total }`

## Analytics

### `GET /api/v1/analytics`

Account analytics for the last 30 days. Optional `?account_id=`.

Response: `{ connected, startDate, endDate, totals: { impressions, saves, clicks, outboundClicks }, data }` — `data` is the raw Pinterest payload (null when no Pinterest account is connected). Totals are honest zeros when Pinterest returns no data. A live Pinterest failure returns the Pinterest error (502), so it is distinguishable from "no data".

## WordPress

### `GET /api/v1/wordpress/sites`

List connected WordPress sites. App passwords are never returned. Response: `{ sites }`.

### `POST /api/v1/wordpress/sites`

Connect a site: `{ siteUrl, username, appPassword }`. The site is verified against the live WordPress REST API first; the app password is stored encrypted and is never returned. Connecting a site that is already connected updates its saved details. Response 201: `{ site }`.

### `DELETE /api/v1/wordpress/sites/[id]`

Disconnect a WordPress site and remove its blog posts. Linked Pinterest pins stay as normal pins. Unknown ids return 404.

### `GET /api/v1/wordpress/blog-posts`

List AI-written blog posts, newest first. Query: `?siteId=<site id>`, `&limit=` (default 50, max 100). List entries omit `contentHtml`. Response: `{ posts, limit }`.

### `POST /api/v1/wordpress/blog-posts`

Write a new draft with Workers AI. Body: `{ siteId, topic, keywords? (string[], max 10), tone? ("friendly" | "professional" | "storytelling"), length? ("short" | "medium" | "long") }`. Saves a DRAFT and returns it (with `contentHtml`). Response 201: `{ post }`.

### `POST /api/v1/wordpress/blog-posts/[id]/publish`

Publish a blog post to WordPress and auto-post its Pinterest pin. Body (all optional — defaults publish immediately): `{ boardId? | pinterestBoardId?, accountId?, status?: "publish" | "draft", imageUrl?, generateImage?, imagePrompt?, title?, contentHtml?, excerpt?, tags? }`. Edited draft fields (title, contentHtml, excerpt, tags) are validated and applied to the post before posting. The Pinterest account is resolved from the board when one is given. Without a board and an image the WordPress post still publishes but no pin is made. Response: `{ post, wordpressPostUrl, pin, pinterestPinId, pinterestPinUrl }`.
