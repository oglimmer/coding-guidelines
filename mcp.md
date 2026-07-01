# MCP server

How we expose an app's data and actions to LLM clients (the **claude.ai remote connector**, **Claude Code**) over the **Model Context Protocol**. There is one way to build it here: an MCP server is an **add-on mounted inside the existing backend** at `/mcp` — never a standalone service — speaking **Streamable HTTP**, and authenticating callers through the app's **own OAuth 2.1 authorization server** scoped to `/mcp`, so a remote client can discover it and connect with no pre-shared secret.

This doc is the target architecture. It layers onto the backend patterns in [go-backend.md](go-backend.md) / [java-spring-backend.md](java-spring-backend.md); chart/ingress wiring is [helm.md](helm.md); tool metrics feed [observability.md](observability.md). The same architecture is realized two ways — Go (official SDK + a small hand-rolled OAuth layer) and Java (Spring AI + Spring Authorization Server); where the stacks force different code, both realizations are given. Per-repo adoption and gaps live in [assessments/mcp.md](assessments/mcp.md).

> **MCP is a gated add-on.** It mounts only when its OAuth client is configured; a backend without MCP config exposes no `/mcp`, no OAuth routes, no discovery. It is a parallel, self-contained surface — never fold MCP concerns into the human `/api`.

## Philosophy

- **Mounted, not separate.** The MCP server is a route group inside the same process that serves the SPA API. One deploy, one image, one auth backend — tools call the same stores the HTTP handlers do.
- **Streamable HTTP, stateless.** `mcp.NewStreamableHTTPHandler(..., {Stateless: true})` (Go) / `protocol: STREAMABLE` (Java). No per-session server state, so a backend restart doesn't 404 a connected client. Not SSE, not stdio.
- **The app is its own OAuth 2.1 AS, scoped to `/mcp`.** Authorization Code + **mandatory PKCE (S256)**, with RFC 8414 + RFC 9728 discovery so claude.ai / Claude Code auto-negotiate the flow. The AS protects `/mcp` only; it is not the SPA's login.
- **MCP tokens can't touch `/api`.** An MCP access token is confined to the MCP surface — a `typ` claim rejected off `/mcp` (Go), or a separate path-matched filter chain (Java). A token handed to Claude cannot drive the human API.
- **Tools carry the caller's identity; authorize per-tool in the handler.** The token subject is the acting user; each tool checks its own permission (`requireMCPUser` / `requireMCPAdmin` in Go; `SecurityContextHolder` in Java). Advertise the whole tool list to everyone; reject in the handler with a message naming what the caller *can* do.
- **Every tool is instrumented.** Wrap each handler so tool calls land in Prometheus — the "MCP tool calls" domain counter in [observability.md](observability.md).
- **The client is Claude.** Redirect URIs default to `https://claude.ai/api/mcp/auth_callback`; add the Claude Code loopback `http://localhost:8765/callback` where CLI use is expected.

## Transport & mount

Mount MCP at `/mcp` over Streamable HTTP, stateless, behind the config gate. In Go, a `chi` group wraps `app.mcpTokenGateMiddleware` then `r.Mount("/mcp", app.mcpHandler())`, mounted only when MCP is enabled. In Java, the `spring-ai-starter-mcp-server-webmvc` starter serves `/mcp` with `spring.ai.mcp.server.protocol: STREAMABLE`. CORS must allow the MCP headers `Mcp-Session-Id`, `Mcp-Protocol-Version`, `Last-Event-ID` and expose `Mcp-Session-Id`.

## Auth — the OAuth 2.1 authorization server

The heart of MCP hosting. A remote client hits `/mcp` unauthenticated, gets a `401` pointing at discovery, walks the flow, and returns with a bearer token.

**401 challenge.** Unauthenticated `/mcp` returns `WWW-Authenticate: Bearer realm="…", resource_metadata="…/.well-known/oauth-protected-resource/mcp"` (RFC 9728). That header is how the client finds the AS.

**Discovery + endpoints:**

| Purpose | RFC | Go | Java (Spring AS) |
|---------|-----|----|------------------|
| AS metadata | 8414 | `/.well-known/oauth-authorization-server` | same |
| Protected-resource metadata | 9728 | `/.well-known/oauth-protected-resource[/mcp]` | same |
| Authorize | — | `GET/POST /oauth/authorize` | `/oauth2/authorize` → `/login` |
| Token | — | `POST /oauth/token` | `/oauth2/token` |

**Client registration.** The baseline is a **single static client** for the Claude connector — `MCP_OAUTH_CLIENT_ID`/`SECRET` with an **exact-match redirect-URI allowlist** and mandatory PKCE. This is the tightest surface and is all the Claude connector needs. Dynamic Client Registration (RFC 7591, `POST /oauth2/register`) is available in the Spring stack and acceptable to leave enabled there; reach for it only when you must onboard clients you can't pre-register.

**Token model** (the rules are identical; only the signing primitive differs by stack):

| | Rule | Go | Java |
|---|------|----|------|
| Access token | short-lived JWT, scope-confined to `/mcp` | HS256, `JWT_SECRET`, 1h, `typ: mcp_access` + `ver` revocation counter | RS256, RSA **JWK**, 30m |
| Refresh token | rotating (delete-old-return-new), **hashed at rest**, long TTL | opaque hex, SHA-256, 30d | 180d |
| Auth code | hashed, short TTL (~10 min) | ✅ | Spring AS default |
| PKCE | **S256 mandatory** — reject missing / `plain` | ✅ | public client, `token_endpoint_auth_method: none` |

**User-identity leg.** Production is `AUTH_MODE=oidc`: the authorize step chains through the real OIDC provider — OAuth params are stashed (an `oauth_pending` row in Go) and resumed in the OIDC callback, which mints the auth code. Java renders a themed `/login` and lets Spring AS drive consent against the same provider. `AUTH_MODE=password` is a **local-dev shortcut only** — never enable it in a deployed profile.

**Signing keys.** Go signs with `JWT_SECRET` (the app secret; min 32 chars, never the in-repo default). Java signs with an RSA **JWK** from `MCP_JWK_PRIVATE_KEY`/`PUBLIC_KEY` — set a stable key in prod; an unset key generates an **ephemeral** RSA key that invalidates every token on restart. Give MCP its own OAuth/resource-server filter chain(s), separate from the SPA chain, so MCP rules never bleed into `/api` (in Spring: `@Order(1)` AS + `@Order(2)` resource server on `/mcp/**` + `@Order(3)` form login).

**Housekeeping.** Sweep expired auth codes / refresh tokens / pending rows on a schedule (`StartOAuthGC`, hourly). Rate-limit `/oauth/token` per IP ([go-backend.md](go-backend.md)).

## Discovery metadata

This is where MCP onboarding most often breaks. A client is handed only the `/mcp` URL; everything else — where to authorize, where to get a token, what PKCE method to use — it learns by walking two metadata documents. Get one field or one route wrong and the client fails to connect with no useful error.

**The discovery chain**, in the order the client walks it:

1. `POST /mcp` with no token → `401` carrying `WWW-Authenticate: Bearer resource_metadata="<base>/.well-known/oauth-protected-resource/mcp"`.
2. `GET` that URL → **protected-resource metadata** (RFC 9728): which AS protects this resource.
3. `GET <as>/.well-known/oauth-authorization-server` → **authorization-server metadata** (RFC 8414): the authorize/token endpoints + supported PKCE/grants.
4. Run authorize → token, then retry `/mcp` with the bearer token.

Both documents are **unauthenticated** and both must be reachable through the ingress. The two target bodies:

```json
// GET /.well-known/oauth-authorization-server   (RFC 8414)
{
  "issuer": "<base>",
  "authorization_endpoint": "<base>/oauth/authorize",
  "token_endpoint": "<base>/oauth/token",
  "response_types_supported": ["code"],
  "grant_types_supported": ["authorization_code", "refresh_token"],
  "code_challenge_methods_supported": ["S256"],
  "token_endpoint_auth_methods_supported": ["client_secret_basic", "client_secret_post"]
}
// GET /.well-known/oauth-protected-resource[/mcp]   (RFC 9728)
{
  "resource": "<base>/mcp",
  "authorization_servers": ["<base>"],
  "bearer_methods_supported": ["header"]
}
```

Four things determine whether this works — each one has cost real debugging time:

- **Serve protected-resource metadata at BOTH `/.well-known/oauth-protected-resource` and `/.well-known/oauth-protected-resource/mcp`.** RFC 9728 metadata is *resource-path-specific*: a client that derives the URL from the resource identifier (`<base>/mcp`) asks for the **`/mcp`-suffixed** path, while a client that reads the `resource_metadata` pointer from the 401 fetches whatever that pointer names. Point the challenge at the suffixed path and register the **same handler** on both. Go registers one handler on both literal routes; Spring covers both with a `/.well-known/oauth-protected-resource/**` matcher. Miss the suffixed variant and the Claude connector 404s mid-discovery — the single most common failure.
- **`issuer` = the externally-reachable base URL, and it must exactly equal the token `iss`.** Derive it from config (`PUBLIC_BASE_URL` / `MCP_ISSUER_URI`) — **never from the request `Host` header** — and feed the *same* value into (a) the metadata `issuer`, (b) the endpoint URLs in the metadata, and (c) the `iss` claim minted at the token endpoint. If the metadata advertises `https://app.example.com` but tokens carry `http://localhost:8080`, the resource server rejects every token and all tool calls 401. In Spring, one `app.mcp.issuer-uri` feeds both `AuthorizationServerSettings.issuer` and the resource server's expected issuer — mirror that single-source discipline in Go.
- **Trim the trailing slash.** `<base> + "/mcp"` must yield `https://app/mcp`, not `https://app//mcp`. Normalize the configured base (`strings.TrimRight(base, "/")`) at every URL you build; a stray slash in the env var otherwise corrupts `resource` and the endpoint URLs.
- **Publish every `.well-known` route on the ingress.** The two metadata paths (plus the `/mcp`-suffixed one) are the most-forgotten ingress entries. If discovery works locally but not in the cluster, check the ingress before the code.

**Generate it, or let the framework generate it.** In Go these are two small hand-written JSON handlers driven entirely off the one base-URL config — keep them dumb. In Spring, the `spring-ai-community` mcp-security configurers + Spring Authorization Server emit both documents from `AuthorizationServerSettings.issuer` + `mcp.authorizationServer(issuerUri)`; you supply only the issuer, and the framework additionally advertises `registration_endpoint` (DCR), `jwks_uri`, `scopes_supported`, and `/.well-known/openid-configuration`. The hand-rolled Go path advertises none of those — no DCR, and HS256 needs no JWKS.

## Tools

- **Register in one place** — `registerMCPTools` (Go) or `@McpTool` methods in a dedicated `mcp/` package (Java) — and wrap every handler for metrics.
- **Authorize inside the handler**, off the token subject — not in middleware. A non-admin calling an admin tool gets an error naming its allowed tools; nothing is silently hidden except genuinely gated data.
- **Read-mostly by default.** Mark read tools `readOnlyHint` / `destructiveHint=false`. Expose mutations deliberately; don't expose destructive deletes via MCP.
- **Shape the response so every client can read it** — see the next section; this is the second thing that reliably breaks.

## Tool responses — structured & unstructured

An MCP tool result carries its payload in two parallel channels, and the gap between them bites hard:

- **`content`** — an array of blocks (text, image, blob) meant for the model to read.
- **`structuredContent`** — a typed JSON object, validated against the tool's output schema, for programmatic clients.

**The trap: some clients render only `content` and ignore `structuredContent`.** The claude.ai connector is one of them. A tool that puts its data solely in `structuredContent` with just a one-line summary in `content` shows the model *only* the summary (`"8 plugin(s)"`) and none of the data — the result looks empty. Treat the **text block as the source of truth** and `structuredContent` as a bonus for clients that read it.

**House rule: render the full payload into a text block *and* let the structured channel carry it too.** In Go a single `okResult(summary, out)` helper does both — the text block is `summary + "\n\n" + json.MarshalIndent(out)`, while the SDK returns `out` as `structuredContent`. Route every success path through it:

```go
func okResult[T any](text string, out T) (*mcp.CallToolResult, T, error) {
    body, _ := json.MarshalIndent(out, "", "  ")
    return &mcp.CallToolResult{Content: toolText(text + "\n\n" + string(body))}, out, nil
}
```

This works around a subtle SDK behavior worth knowing: the go-sdk auto-embeds the JSON into a text block **only when `Content` is left nil**. The moment you set a human-readable summary line, that fallback is suppressed and the data would vanish from `content` — so setting both explicitly is deliberate, not redundant. In **Spring AI this is automatic**: return a record / `List<record>` / `Map` from an `@McpTool` method and the framework emits **both** a `structuredContent` object and a text serialization — which is exactly why the Java app never hit the text-only-client bug. The principle is the same; only Go has to do it by hand.

**Output schemas are inferred from the return type — mind them.** The go-sdk derives an *output* schema (not just the input schema) from the handler's generic `Out` type and validates it at registration — a malformed `jsonschema:"…"` tag panics on startup (pin a registration smoke test). So:

- Use a **concrete `Out` struct** so the schema is meaningful; put field docs in `json:` / `jsonschema:` tags.
- When one tool legitimately returns **two shapes** (e.g. an admin view vs a user view), type it `Out = any` — the SDK then omits the output schema instead of forcing one struct. Spring AI derives the output schema from the Java return type and the input schema from `@McpToolParam`.

**Errors are a plain error return.** Return a Go `error` (`fmt.Errorf("plugin %q not found", name)`) and the SDK packs it into an `IsError` text result — no separate error-result helper, and never smuggle a failure into a success payload.

**Binary rides inside the JSON payload.** There's no special MCP content type in use for tool output — a binary file is an ordinary struct field: a base64 string + an `isBinary` flag + a size, marshaled into the text block like everything else. Keep the write side symmetric (accept base64 when `isBinary` is set).

## Config

**Go** (`internal/config`):

| Env | Meaning |
|-----|---------|
| `MCP_OAUTH_CLIENT_ID` / `MCP_OAUTH_CLIENT_SECRET` | The static client. **Both or neither** — MCP is enabled only when both are set; startup fails if only one is. Empty = `/mcp` and OAuth routes 404. |
| `MCP_OAUTH_REDIRECT_URIS` | Comma list; exact-match allowlist. Default `https://claude.ai/api/mcp/auth_callback`. |
| `PUBLIC_BASE_URL` | OAuth `issuer` + endpoint base. Must be the externally-reachable URL. |
| `AUTH_MODE` | `oidc` (prod) or `password` (dev only). |
| `JWT_SECRET` | Signs MCP access tokens (also the app session secret). |

**Java** (`app.mcp.*` in `application.yml`):

| Env | Property | Default |
|-----|----------|---------|
| `MCP_ISSUER_URI` | `app.mcp.issuer-uri` | `http://localhost:8080` — set to the public URL |
| `MCP_CORS_ALLOWED_ORIGINS` | `app.mcp.cors-allowed-origins` | pin to `https://claude.ai` |
| `MCP_DCR_ENABLED` | `app.mcp.dynamic-client-registration-enabled` | `true` |
| `MCP_ACCESS_TOKEN_TTL` / `MCP_REFRESH_TOKEN_TTL` | `app.mcp.token.*` | `30m` / `180d` |
| `MCP_JWK_PRIVATE_KEY` / `MCP_JWK_PUBLIC_KEY` | `app.mcp.jwk.*` | empty → ephemeral key (dev) |

## Helm & ingress

- **Gate on a values toggle.** An `mcp.enabled` / `mcpOAuth` block drives the OAuth env injection and the ingress paths ([helm.md](helm.md)); the client secret / JWK ride the SealedSecret.
- **Publish the whole surface.** Route to the backend: `/mcp`, the OAuth endpoints (`/oauth` in Go; `/oauth2` + `/connect` + `/login` in Java), and **both** `/.well-known/oauth-authorization-server` and `/.well-known/oauth-protected-resource` (including the `/mcp`-suffixed variant). Miss a `.well-known` route and the client can't discover the AS.
- **Streamable HTTP needs a long-lived stream.** nginx-ingress: `proxy-buffering: off`, `proxy-read-timeout` / `proxy-send-timeout: 3600`. Traefik: one IngressRoute rule covering `/mcp` + the OAuth/discovery paths. Without buffering off + long timeouts the stream gets cut.

## Tests

- **OAuth surface, no DB:** PKCE verification, exact-match redirect allowlist (reject prefix/suffix/query variants), discovery metadata (200 enabled / 404 disabled), the `WWW-Authenticate` challenge, authorize param validation (bad `client_id`/`redirect_uri` → 400 with no redirect; unsupported `response_type` / missing challenge / non-S256 → redirect error), token endpoint errors + `Cache-Control: no-store`, **scope confinement** (an MCP token is rejected on `/api`), constant-time client validation, `/oauth/token` rate limiting.
- **MCP surface:** tool registration builds; the full-payload-in-text rendering is pinned. Integration test (Testcontainers): public discovery metadata, `401 + WWW-Authenticate` on `/mcp`, and the authorize→token round-trip.

## New-repo checklist

1. **Gate + mount.** Add the MCP config keys; mount `/mcp` over Streamable HTTP, stateless, only when MCP is enabled.
2. **Stand up the OAuth AS.** Discovery (RFC 8414 + 9728) served at both protected-resource paths (bare + `/mcp`-suffixed), authorize + token with **mandatory PKCE S256**, scope-confined access tokens, rotating hashed refresh tokens, and the `401` `WWW-Authenticate` challenge pointing at the suffixed metadata URL. Static client + redirect allowlist as the baseline. Feed one config value into the metadata `issuer`, the endpoint URLs, and the token `iss`.
3. **Wire identity.** `AUTH_MODE=oidc` in prod, chaining the authorize step through the app's OIDC provider; `password` mode dev-only.
4. **Set the externally-reachable issuer** (`PUBLIC_BASE_URL` / `MCP_ISSUER_URI`), the Claude redirect URI, and a real signing key (`JWT_SECRET` / RSA JWK — never the ephemeral dev key).
5. **Register tools in one place**, authorize per-tool off the token subject, return the full payload in a **text** content block, and wrap each handler for Prometheus.
6. **Chart it.** Gate on a values toggle, publish `/mcp` + OAuth + **both** `.well-known` routes, set buffering-off + long proxy timeouts, carry secrets in the SealedSecret.
7. **Verify** a real client (claude.ai connector or the `mcp` CLI) can discover, authorize, and call a tool. Rate-limit `/oauth/token`; GC expired codes/tokens.

## Anti-patterns

| Do not | Do instead |
|--------|------------|
| Ship MCP as a separate service | Mount it in the existing backend at `/mcp` |
| Run a stateful / SSE-session transport | Streamable HTTP, `Stateless: true` — survives restarts |
| Let an MCP token hit `/api` | Scope-confine it (`typ` claim / separate filter chain) |
| Accept `code_challenge_method=plain` or skip PKCE | Require S256; reject missing / other |
| Redirect to any `redirect_uri` | Exact-match allowlist (or DCR with validation) |
| Derive the issuer from the request `Host`, or let metadata `issuer` and token `iss` diverge | Pin one config base URL; use it for metadata, endpoints, and `iss` |
| Serve protected-resource metadata only at the bare path | Also serve `/.well-known/oauth-protected-resource/mcp` (same handler) |
| Leave a trailing slash on the base URL | `TrimRight(base, "/")` before building every URL |
| Enable `AUTH_MODE=password` in a deployed profile | `oidc` in prod; password is dev-only |
| Ship without a stable signing key | Real `JWT_SECRET` / RSA JWK — the ephemeral key invalidates all tokens on restart |
| Forget a `.well-known` ingress route | Publish both AS + protected-resource metadata paths |
| Buffer the MCP stream / short proxy timeout | `proxy-buffering off` + long read/send timeouts |
| Return only `StructuredContent` from a tool | Embed the full payload in a text content block |
| Authorize MCP tools in middleware only | Check per-tool in the handler off the token subject |
| Store refresh tokens / auth codes in plaintext | Hash at rest; rotate refresh tokens |
