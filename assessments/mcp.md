# MCP server — project assessment

Per-repo adoption for [../mcp.md](../mcp.md). Last verified: July 2026.

Three repos ship an MCP server; all serve `/mcp` over Streamable HTTP (stateless) and act as their own OAuth 2.1 AS scoped to `/mcp`.

| Repo | Stack | MCP library | Tools | OAuth AS | Client model | Access token |
|------|-------|-------------|-------|----------|--------------|--------------|
| **plugin-skill-hosting** | Go | `modelcontextprotocol/go-sdk` v1.6.1 (official) | 8 — plugin/skill read + edit | hand-rolled | static client | HS256 JWT, 1h |
| **irl-planner-pro** | Go | same v1.6.1 | 16 — events/RSVP/roster, user vs admin | hand-rolled (adapted from plugin) | static client | HS256 JWT, 1h |
| **deep-digest-rss** | Java `news-backend` | `spring-ai-starter-mcp-server-webmvc` (Spring AI 2.0.0) + `spring-ai-community:mcp-security` 0.1.13 | 4 — news, read-only | Spring Authorization Server | DCR (RFC 7591) | RS256 JWT (JWK), 30m |

**plugin-skill-hosting is the Go source of truth.** `irl-planner-pro`'s `oauth.go` is explicitly *"adapted from the plugin-skill-hosting reference, trimmed of its password/bcrypt and user-status concepts"* — the two share ~20 identically-named OAuth functions; only the tool surface is bespoke. A third Go MCP app should copy `plugin-skill-hosting` wholesale.

## Where each matches / deviates from the target

| Area | plugin-skill-hosting | irl-planner-pro | deep-digest-rss |
|------|----------------------|-----------------|-----------------|
| Transport `/mcp` Streamable HTTP stateless | ✅ | ✅ | ✅ |
| Own OAuth 2.1 AS, PKCE S256 | ✅ | ✅ | ✅ (Spring AS) |
| RFC 8414 + 9728 discovery | ✅ | ✅ | ✅ |
| Scope-confined MCP token | ✅ `typ:mcp_access` | ✅ `typ:mcp_access` | ✅ separate `/mcp/**` filter chain |
| Client registration | ⚠️ static only | ⚠️ static only | ✅ DCR (matches spec intent) |
| Config gate | ✅ `MCPEnabled()` | ✅ `mcp.enabled` toggle | ⚠️ always on (no toggle) |
| Prod identity `AUTH_MODE=oidc` | ✅ (bcrypt form for dev) | ✅ (**dev `password` mode does no credential check**) | ✅ Thymeleaf `/login` + Spring AS |
| Live tool-execution tests | ❌ | ❌ | ⚠️ integration test covers auth, not tools |

## Repo-specific notes

- **plugin-skill-hosting** — also accepts a **static API token** (bearer or HTTP Basic) as a non-OAuth alternative, encrypted at rest (AES-256-GCM, migration `0015`; auth lookup uses a separate SHA-256 hash column, so key rotation only affects UI re-display). This is an extra machine-client path beyond the target's OAuth flow — copy only if you need non-OAuth clients. OAuth tables in `0011_oauth.sql`. Ships a frontend `McpSection.vue` dev UI.
- **irl-planner-pro** — richest tool surface (16 tools split user vs admin via `requireMCPUser`/`requireMCPAdmin`). OAuth tables in `0004_oauth.sql`. **`AUTH_MODE=password` performs no credential check** — a local-dev bypass that must never reach a deployed profile. Helm `mcp.enabled` defaults off; redirect URIs include the Claude Code loopback.
- **deep-digest-rss** — the only DCR implementation. `LongLivedTokenRegisteredClientRepository` overrides Spring AS's 60-min default refresh TTL (which forced claude.ai to re-auth hourly) for every client. `JwtDecoder` validates against the in-process `JWKSource` (not over HTTP) to avoid a startup deadlock. JDBC-persisted client/authorization stores survive restart. Traefik `IngressRoute` routes `/mcp || /oauth2 || /connect || /.well-known || /login`. Deviates from the canonical Java backend on DB/auth/layout — copy the **MCP add-on only**, onto the canonical pattern; see [java-spring-backend.md](java-spring-backend.md).

## Discovery metadata & response shaping

Concrete facts behind the [../mcp.md](../mcp.md) "Discovery metadata" and "Tool responses" sections:

- **Metadata source.** Both Go apps hand-write the two JSON documents in `oauth.go` (`handleOAuthMeta`, `handleOAuthProtectedResource`), driven off `PUBLIC_BASE_URL` with `strings.TrimRight(base, "/")` at each use; the bodies are byte-identical between `plugin` and `irl`. Neither advertises `registration_endpoint` or `scopes_supported`. `deep-digest-rss` emits both documents from Spring Authorization Server + `spring-ai-community` mcp-security, supplying only `app.mcp.issuer-uri` — that one value feeds both `AuthorizationServerSettings.issuer` and the resource server's expected issuer (the explicit `iss`==metadata-issuer guarantee), and the framework additionally advertises `registration_endpoint`, `jwks_uri`, `scopes_supported`, `/.well-known/openid-configuration`.
- **`/mcp`-suffixed protected-resource path.** Go registers the same handler on both `/.well-known/oauth-protected-resource` and `…/mcp` (always in `plugin`; only when MCP-enabled in `irl`); Java covers both via a `/.well-known/oauth-protected-resource/**` matcher. All three point the `/mcp` 401 `WWW-Authenticate` `resource_metadata` at the suffixed URL.
- **Structured-vs-text response.** Both Go apps route every success through an `okResult` helper that embeds `json.MarshalIndent(out)` into the **text** content block *and* returns `out` as `structuredContent` — pinned by `TestOKResultEmbedsPayloadInText`, whose comment records the original bug: read tools returned only a summary header (e.g. `"8 plugin(s)"`) because the data was in `structuredContent` only, which the claude.ai connector ignores. The go-sdk infers an **output** schema from the generic `Out` type; `irl`'s `list_events` uses `Out = any` to return two shapes (admin vs user). Spring AI emits both channels automatically from the Java return type, so `deep-digest-rss` never hit the text-only-client bug.

## Disclosed gaps

1. **No live tool-execution tests** in either Go app — the OAuth surface is well covered (~18–25 cases in `oauth_test.go`), the tool handlers are not exercised end-to-end.
2. **`deep-digest-rss` MCP has no config gate** — it's always mounted, unlike the Go apps' `MCPEnabled()` / `mcp.enabled` toggle. Fine for a dedicated MCP app; add a toggle if MCP should be optional.
3. **`irl-planner-pro` dev `password` mode does no credential check.** Keep it strictly local; production is `oidc`.

## Safe copy sources

| You need… | Copy from |
|-----------|-----------|
| Go MCP server + OAuth 2.1 AS | `plugin-skill-hosting` (`mcp.go` + `oauth.go` + `auth.go` MCP gate, `0011_oauth.sql`) |
| Go MCP with user/admin tool split | `irl-planner-pro` |
| Java Spring MCP + Spring Authorization Server + DCR | `deep-digest-rss/news-backend` — MCP add-on only, onto the canonical Java pattern |
| Non-OAuth static API token (encrypted at rest) | `plugin-skill-hosting` (migration `0015`) |
