# MCP OAuth — Keycloak

<p>
  <a href="https://github.com/mcp-use/mcp-use">Built with <b>mcp-use</b></a>
  &nbsp;
  <a href="https://github.com/mcp-use/mcp-use">
    <img src="https://img.shields.io/github/stars/mcp-use/mcp-use?style=social" alt="mcp-use stars">
  </a>
</p>

An MCP server template that delegates authentication to a Keycloak realm using **Dynamic Client Registration (RFC 7591)**. MCP clients register themselves with Keycloak on first use, complete a PKCE authorization flow, and send the resulting access token as a bearer token on MCP requests. The MCP server only verifies the JWT against Keycloak's JWKS — it never proxies OAuth traffic.

## Features

- **Dynamic Client Registration (RFC 7591)** — clients self-register against Keycloak
- **PKCE authorization** — secure flow for public clients
- **JWT verification via JWKS** — Keycloak's published keys
- **Realm roles + scopes in `ctx.auth`** — enforce per-tool authorization

## Prerequisites

- **Node.js 20+** (22 recommended)
- **pnpm 10+**
- A reachable **Keycloak realm** with:
  - **Dynamic Client Registration** enabled (Keycloak's OIDC client registration service is on by default at `/realms/{realm}/clients-registrations/openid-connect`)
  - An anonymous DCR policy permitting the `redirect_uris` your MCP client will send. The default `Trusted Hosts` policy only allows `localhost` / `127.0.0.1` — loosen or tighten to match your deployment.
  - On Keycloak **26.6+** with browser-based MCP clients (e.g. the inspector), configure the `Allowed Registration Web Origins` policy with the client's origin — otherwise Keycloak returns `403 Invalid origin` on the DCR POST.
- A **test user** in the realm so you can log in during the PKCE flow.

## Setup

### 1. Configure environment variables

Copy `.env.example` to `.env` and point it at your Keycloak:

```bash
cp .env.example .env
```

```bash
MCP_USE_OAUTH_KEYCLOAK_SERVER_URL=http://localhost:8080
MCP_USE_OAUTH_KEYCLOAK_REALM=demo
# Optional — requires an Audience protocol mapper in Keycloak
# MCP_USE_OAUTH_KEYCLOAK_AUDIENCE=http://localhost:3000
```

### 2. Install and run

```bash
pnpm install
pnpm dev
```

The server starts on port **3000** with the inspector at <http://localhost:3000/inspector>.

## Try it out

1. Open <http://localhost:3000/inspector>
2. Connect to `http://localhost:3000/mcp`
3. The inspector discovers Keycloak via `/.well-known/oauth-authorization-server`, registers itself via DCR, and redirects you to the Keycloak login page
4. Log in with a user from your realm
5. Back in the inspector, call:
   - `get-user-info` — claims lifted from the JWT (`sub`, `preferred_username`, realm roles, scopes…)
   - `get-keycloak-userinfo` — full OIDC userinfo document fetched from Keycloak

## Flow

```
MCP Client ──(1) GET /.well-known/oauth-protected-resource ─▶ MCP Server
MCP Client ──(2) GET /.well-known/oauth-authorization-server ─▶ MCP Server ─▶ Keycloak
MCP Client ──(3) POST /clients-registrations/openid-connect ─▶ Keycloak      (DCR)
MCP Client ──(4) GET  /protocol/openid-connect/auth ─────────▶ Keycloak      (PKCE)
MCP Client ──(5) POST /protocol/openid-connect/token ────────▶ Keycloak
MCP Client ──(6) MCP request + Bearer <token> ──────────────▶ MCP Server    (verifies JWT via JWKS)
```

Step 2 is a passthrough from the MCP server back to Keycloak's metadata — it's what tells the client where to register and where to send the user for login. Everything else goes directly to Keycloak.

## Notes

- **Audience**: Keycloak doesn't set `aud` to the resource server by default. To enforce `aud`, add an *Audience* protocol mapper on the client scope in Keycloak and set `MCP_USE_OAUTH_KEYCLOAK_AUDIENCE` to the matching value.
- **Anonymous DCR**: The default `Trusted Hosts` policy enforces that registration `redirect_uris` use an allowed hostname. For non-localhost redirects, either extend that policy's trusted-hosts list or mint an Initial Access Token and have the client pass it on the registration request.
- **Browser clients**: On Keycloak 26.6+, add the `Allowed Registration Web Origins` client-registration policy (provider id `registration-web-origins`, config key `web-origins`) listing every origin your inspector / client will run from. Without it, browser DCR is blocked by CORS.
- **Production**: Turn off anonymous DCR, require Initial Access Tokens, serve everything over HTTPS, and set `MCP_USE_OAUTH_KEYCLOAK_AUDIENCE`.

## Deploy

```bash
npx mcp-use deploy
```

## Learn more

- [Keycloak Client Registration](https://www.keycloak.org/docs/latest/securing_apps/index.html#_client_registration)
- [RFC 7591 — Dynamic Client Registration](https://www.rfc-editor.org/rfc/rfc7591)
- [mcp-use docs](https://mcp-use.com/docs)
- [MCP Authorization spec](https://modelcontextprotocol.io/specification/2025-11-25/basic/authorization)

## License

MIT
