# MCP OAuth 2.0 — Claude Connectors Directory backend

Implemented 2026-11-12. **Path B (hand-rolled, SDK v1 kept)** — see
"Why Path B" below. Do NOT deploy from this doc alone: the coordinator
deploys once at the end, after applying migration 003 and setting the
`OAUTH_SIGNING_KEY` Worker secret.

## What Claude requires (verified vs live Anthropic docs, 12 Nov 2026)

- OAuth 2.0 authorization code + PKCE S256 for servers acting on a
  user's account. API-key auth alone is NOT acceptable for the
  directory — but API keys KEEP working for REST + direct MCP clients.
- Unauthenticated `/mcp` returns 401 WITH `WWW-Authenticate` pointing
  at the RFC 9728 protected-resource metadata; its `resource` is exactly
  `https://365pinterest.com/mcp`.
- RFC 8414 authorization-server metadata, reachable from Anthropic
  egress 160.79.104.0/21, advertising `code_challenge_methods_supported:
  ["S256"]` and `offline_access` in `scopes_supported`.
- Redirect URIs: `https://claude.ai/api/mcp/auth_callback` (hosted) and
  loopback `http://localhost/callback` + `http://127.0.0.1/callback`
  (Claude Code, port-agnostic). Consent shows the redirect hostname.
- Token endpoint accepts `application/x-www-form-urlencoded` (code +
  refresh); refresh failures return RFC 6749 codes (`invalid_grant`);
  refresh tokens ROTATE; `client_credentials` is NOT supported.
- Client identity: CIMD preferred (advertise
  `client_id_metadata_document_supported: true` AND `"none"` in
  `token_endpoint_auth_methods_supported`), DCR (RFC 7591) as fallback.

## Why Path B (not Path A)

Path A (install `@better-auth/mcp` + `@better-auth/cimd`, migrate the
MCP handler to SDK v2 `createMcpHandler`) was rejected:

- The better-auth OAuth Provider doc warns the plugin "is in active
  development and may not be suitable for production use yet".
- It forces a rewrite of the working 12-tool SDK-v1.32.2 server to SDK
  v2 (`@modelcontextprotocol/server`), which is not installed.
- `@better-auth/cimd/node`'s fetch transport is Node-only; Workers
  needs a custom transport anyway (and Workers has no DNS API, so true
  resolve-once/IP-pinning is impossible — see CIMD note below).

Path B keeps SDK v1.32.2 + the existing transport, uses `jose` 6.2.17
(WebCrypto, Workers-compatible) for ES256 JWTs, and implements the
small OAuth surface directly: authorize / consent / token / register /
revoke / userinfo + RFC 8414 / RFC 9728 / JWKS discovery. Treat the
first deploy as experimental and test heavily.

## Endpoints

| Route | Purpose |
| --- | --- |
| `GET /.well-known/oauth-authorization-server` | RFC 8414 metadata (issuer, endpoints, S256, CIMD flags) |
| `GET /.well-known/oauth-protected-resource/mcp` | RFC 9728 metadata (`resource` = `https://365pinterest.com/mcp`) |
| `GET /.well-known/jwks.json` | Public ES256 key (`kid` = JWK thumbprint) |
| `GET /oauth/authorize` | Code flow start; signed-out users go to `/sign-in?next=...` |
| `GET /consent?req=...` | Consent screen (client name/logo, scopes in plain language, redirect hostname) |
| `POST /oauth/consent` | Approve/Deny; approve mints a single-use code (10 min) |
| `POST /oauth/token` | Form-urlencoded: `authorization_code` (+PKCE S256) and `refresh_token` (rotating) |
| `POST /oauth/register` | DCR (RFC 7591), open registration, 201 + `client_secret` once |
| `POST /oauth/revoke` | RFC 7009 refresh-token revocation (always 200) |
| `GET /oauth/userinfo` | OIDC userinfo (`sub`, `name`, `email`) for `openid`/`profile` |
| `POST /mcp` | Accepts `Bearer pk_live_…` OR OAuth JWT; 401 + `WWW-Authenticate` when unauthenticated |

## Scopes (v1 — explicit, least-privilege; no `mcp:tools` umbrella)

`openid`, `profile`, `offline_access` (refresh token iff requested),
`pins:read`, `pins:write`, `boards:read`, `wordpress:read`,
`wordpress:write`. No scope requested → all except `offline_access`.
Per-tool enforcement in `app/mcp/route.ts` (`TOOL_SCOPES`):
read-only tools need the matching `:read` scope, mutating tools the
`:write` scope; `account_status` accepts `pins:read` or `boards:read`.

## Tokens & storage

- Access token: stateless ES256 (ECDSA P-256) JWT, 1h TTL.
  `iss=https://365pinterest.com`, `aud=https://365pinterest.com/mcp`,
  `sub`=user id, `scope`, `client_id`. Verified in `/mcp` against the
  key from `OAUTH_SIGNING_KEY` (never stored in the DB).
- Authorization codes: single-use, 10 min, SHA-256 hash stored,
  PKCE-bound, deleted before tokens are issued.
- Refresh tokens: opaque, 30 days, SHA-256 hash stored; ROTATE on
  every use (old row deleted, new token returned in the same response).
- DCR client secrets: SHA-256 hash stored; shown once at registration.
- CIMD clients (client_id = metadata URL) are NOT stored; the document
  is fetched + validated per use (5-min in-memory cache).
- **CIMD fetch hardening (Workers caveat):** Workers has no DNS API, so
  resolve-once/IP-pinning is not implemented. Mitigations: https only,
  no credentials/fragment, localhost / `.local` / `.internal` /
  private / loopback / link-local hosts rejected (hostname + IP-literal
  forms), `redirect: "manual"`, 5s timeout, 16KB cap, strict document
  validation. A failed fetch fails closed; DCR is the fallback.

## Secrets (set on the Worker BEFORE deploy)

- `OAUTH_SIGNING_KEY` — **new, required**. Generate:
  `npx tsx scripts/generate-oauth-key.mjs`, then
  `wrangler secret put OAUTH_SIGNING_KEY` and paste the printed
  single-line JWK JSON. Never commit it, never print it in chat.
  Accepts a JWK JSON or a PKCS#8 PEM.
- `MIGRATE_TOKEN` — one-time secret for the migration route; delete it
  (`wrangler secret delete MIGRATE_TOKEN`) after migration 003 is
  confirmed, and remove `app/api/admin/migrate-oauth-003/` later.
- Reused, unchanged: `BETTER_AUTH_SECRET` (consent-request JWT HMAC),
  `TOKEN_ENCRYPTION_KEY` (not used by OAuth — codes/tokens are hashed,
  not encrypted), `DATABASE_URL`.

## Migration 003 — apply steps (coordinator, at deploy time)

1. Deploy the code (migration route ships with it).
2. `wrangler secret put MIGRATE_TOKEN` (any random value; keep it).
3. `curl -X POST https://365pinterest.com/api/admin/migrate-oauth-003 \
   -H "x-migrate-token: <MIGRATE_TOKEN>"` → expect
   `{"applied":true,"statements":11,"tables":["oauth_authorization_codes","oauth_clients","oauth_refresh_tokens"]}`.
   Re-running returns `{"alreadyApplied":true,...}`.
   (Local equivalent, only if a DATABASE_URL is available:
   `DATABASE_URL=... npx tsx scripts/apply-sql-migration.ts scripts/migration-003-oauth.sql`.)
4. `wrangler secret delete MIGRATE_TOKEN`.
5. Verify: `GET /.well-known/oauth-authorization-server` and
   `GET /.well-known/jwks.json` return 200; `GET /mcp` without auth
   returns 401 with a `WWW-Authenticate` header.

## Testing the OAuth flow (post-deploy)

```bash
# 1. Discovery
curl https://365pinterest.com/.well-known/oauth-authorization-server
curl https://365pinterest.com/.well-known/oauth-protected-resource/mcp
curl https://365pinterest.com/.well-known/jwks.json
curl -i https://365pinterest.com/mcp   # 401 + WWW-Authenticate header

# 2. Register a test public client (DCR)
curl -X POST https://365pinterest.com/oauth/register \
  -H 'Content-Type: application/json' -d '{
    "redirect_uris": ["http://127.0.0.1/callback"],
    "client_name": "OAuth test client",
    "token_endpoint_auth_method": "none",
    "scope": "openid profile offline_access pins:read pins:write boards:read wordpress:read wordpress:write"
  }'
# -> {"client_id": "mcp_oauth_...", ...}

# 3. PKCE pair (any S256 pair; e.g. python: base64url(sha256(verifier)))
# 4. Authorize in a browser (signed in as a 365 Pinterest user):
# https://365pinterest.com/oauth/authorize?response_type=code
#   &client_id=mcp_oauth_...&redirect_uri=http%3A%2F%2F127.0.0.1%2Fcallback
#   &scope=openid%20profile%20offline_access%20pins%3Aread%20pins%3Awrite%20boards%3Aread%20wordpress%3Aread%20wordpress%3Awrite
#   &state=test123&code_challenge=<CHALLENGE>&code_challenge_method=S256
#    -> consent screen -> Approve -> redirect with ?code=...&state=test123
#    (loopback redirect will not load in a browser; copy the code from the URL bar)

# 5. Exchange the code
curl -X POST https://365pinterest.com/oauth/token \
  -H 'Content-Type: application/x-www-form-urlencoded' -d \
  'grant_type=authorization_code&code=<CODE>&redirect_uri=http%3A%2F%2F127.0.0.1%2Fcallback&code_verifier=<VERIFIER>&client_id=mcp_oauth_...'
# -> {"access_token": "eyJ...", "token_type": "Bearer", "expires_in": 3600,
#     "refresh_token": "...", "scope": "..."}

# 6. Call MCP with the OAuth token (initialize, then tools/list)
curl -X POST https://365pinterest.com/mcp \
  -H "Authorization: Bearer <ACCESS_TOKEN>" \
  -H 'Content-Type: application/json' -d \
  '{"jsonrpc":"2.0","id":1,"method":"tools/list","params":{}}'
# -> 12 tools, each with title + annotations (readOnlyHint/destructiveHint)

# 7. Refresh (rotates: old refresh token must then fail with invalid_grant)
curl -X POST https://365pinterest.com/oauth/token \
  -H 'Content-Type: application/x-www-form-urlencoded' -d \
  'grant_type=refresh_token&refresh_token=<REFRESH_TOKEN>&client_id=mcp_oauth_...'

# 8. Negative checks: reuse the code (invalid_grant), wrong verifier
#    (invalid_grant), client_credentials grant (unsupported_grant_type),
#    GET /mcp with no auth (401 + WWW-Authenticate).
```

## Changes in this batch (2026-11-12)

- `lib/oauth.ts` (new): constants, scopes, PKCE, redirect validation,
  ES256 key handling, JWT mint/verify, consent-request JWT.
- `lib/oauth-clients.ts` (new): DCR register/lookup, CIMD hardened
  fetch + validation, token-endpoint client auth.
- `app/.well-known/oauth-authorization-server/route.ts`,
  `app/.well-known/oauth-protected-resource/mcp/route.ts`,
  `app/.well-known/jwks.json/route.ts` (new): discovery.
- `app/oauth/{authorize,consent,token,register,revoke,userinfo}/route.ts`
  (new) + `app/consent/page.tsx` (new): the AS + consent screen.
- `app/mcp/route.ts`: accepts API key OR OAuth JWT; 401 + RFC 9728
  `WWW-Authenticate`; GET 401 when unauthenticated (405 when authed);
  per-tool scope gates; annotations on all 12 tools
  (read-only: `readOnlyHint` + `idempotentHint`; mutating:
  `destructiveHint`; `openWorldHint` never set); `publish_blog_post`
  no longer accepts `generateImage`/`imagePrompt` (no AI image
  generation via MCP — REST + dashboard keep it); server version
  0.2.0 → 0.3.0.
- `app/sign-in/page.tsx`: honours `?next=` (same-origin relative
  only) so a signed-out authorize flow returns to `/oauth/authorize`
  after sign-in.
- `prisma/schema.prisma`: `OAuthClient`, `OAuthAuthorizationCode`,
  `OAuthRefreshToken` models.
- `scripts/migration-003-oauth.sql` (new) +
  `app/api/admin/migrate-oauth-003/route.ts` (new): one-time migration.
- `scripts/generate-oauth-key.mjs` (new): ES256 (ECDSA P-256) key generator.
- `package.json` / `package-lock.json`: `jose@^6.2.17` as a direct
  dependency (was transitive only).

## What WAS verified locally (12 Nov 2026, dev server + tsx harness)

- `tsc --noEmit` clean; full `next build` succeeds (dummy DATABASE_URL).
- Built bundle contains all 12 tool titles + annotations (7×
  `readOnlyHint:true` + `idempotentHint:true`, 5× `destructiveHint:true`,
  `openWorldHint` never set, `generateImage` absent from the MCP bundle).
- Live over HTTP (`next dev`, test ES256 key via env, dummy DB):
  AS metadata, PR metadata (`resource` exact), JWKS (EC P-256, ES256,
  kid = thumbprint) all 200; GET/POST `/mcp` without auth → 401 +
  `WWW-Authenticate` (JSON-RPC -32001 on POST); garbage Bearer → 401.
- Real minted OAuth JWT → `tools/list` returns all 12 tools with
  correct annotations; `publish_blog_post` inputSchema has no
  `generateImage`/`imagePrompt`.
- Scope gate live: read-only token calling `create_pin` →
  "Insufficient scope" error; same token calling `list_pins` passes
  the gate (fails only on the missing local DB).
- Crypto unit checks (tsx): mint → verify round-trip, tampered/garbage
  tokens rejected, PKCE challenge stable, consent-request JWT
  round-trip + tamper rejection.
- Two real bugs caught + fixed by local testing: (1) jose imports keys
  non-extractable — `exportJWK(privateKey)` throws; the public JWK is
  now taken from the input JWK / an extractable PEM import. (2) RFC
  7638 thumbprint for EC keys requires `y` — it was omitted, so
  `calculateJwkThumbprint` threw; `y` is now included. Also corrected:
  ES256 is ECDSA P-256, not Ed25519 (comments/docs said Ed25519).

## What was NOT verified locally

- No local Postgres is available in this environment, so the DB-backed
  half of the flow could NOT be exercised end-to-end locally:
  DCR registration, authorize → consent → code minting, code exchange,
  refresh rotation, and revoke all need the live DB (migration 003).
  `/oauth/register` returned 500 locally for exactly that reason (no
  DB), and `/oauth/authorize` with an unknown client correctly returned
  400 before touching the DB. Run the §Testing checklist above against
  the deployed Worker before submitting to the directory.
- CIMD fetch was not exercised against a real metadata document
  (no public CIMD URL exists yet); DCR is the tested path.
- Claude's own client behaviour (which scopes it requests, CIMD vs
  DCR choice) can only be confirmed in the Claude Connectors Directory
  submission flow.

## Open questions

- Should consent be remembered per client (skip the screen on repeat
  authorizations)? v1 always asks — safer, and matches "experimental".
- Key rotation procedure is manual (new secret + redeploy); consider a
  second served key + `kid` selection if rotation becomes frequent.
- The MCP Registry server-card (`mcp-registry/server-card.json`,
  version 0.2.0) still says 0.2.0 while the server now reports 0.3.0 —
  update the card at the next registry publish.
