# Claude Code OAuth Reverse Engineering Spec

> Source: `@anthropic-ai/claude-code@2.1.88` (npm pack + minified cli.js analysis)
> Date: 2026-03-31

## Overview

Claude Code uses OAuth 2.0 Authorization Code Flow with PKCE (S256) to authenticate Pro/Max subscription users. The OAuth token is then used as a Bearer token to call `api.anthropic.com/v1/messages`.

Architecture for a proxy:

```
Device A (claude code) ──┐
Device B (claude code) ──┼──→ claude-proxy ──→ api.anthropic.com
Device C (headless)    ──┘    (holds tokens,
                               auto-refreshes)
```

---

## 1. Endpoints

| Purpose | URL | Method |
|---------|-----|--------|
| Client Metadata (RFC 7591) | `https://claude.ai/oauth/claude-code-client-metadata` | GET |
| Authorize | `https://claude.com/cai/oauth/authorize` | Browser redirect |
| Token Exchange | `https://platform.claude.com/v1/oauth/token` | POST |
| Token Refresh | `https://platform.claude.com/v1/oauth/token` | POST |
| Create API Key | `https://api.anthropic.com/api/oauth/claude_cli/create_api_key` | POST |
| Get Roles | `https://api.anthropic.com/api/oauth/claude_cli/roles` | POST |
| Messages API | `https://api.anthropic.com/v1/messages` | POST |

## 2. Client Registration

Public client (no client_secret):

```json
{
  "client_id": "https://claude.ai/oauth/claude-code-client-metadata",
  "client_name": "Claude Code",
  "client_uri": "https://claude.ai",
  "redirect_uris": ["http://localhost/callback", "http://127.0.0.1/callback"],
  "grant_types": ["authorization_code", "refresh_token"],
  "response_types": ["code"],
  "token_endpoint_auth_method": "none"
}
```

Production `CLIENT_ID` (hardcoded): `9d1c250a-e61b-44d9-88ed-5944d1962f5e`

> Note: The `client_id` in the metadata response is the metadata URL itself (RFC 7591 dynamic client).
> The hardcoded `CLIENT_ID` is used in actual token requests.

## 3. Scopes

```
user:inference           -- call the Messages API
user:profile             -- read user profile
user:sessions:claude_code -- manage CLI sessions
user:mcp_servers         -- access MCP servers
user:file_upload         -- upload files
org:create_api_key       -- create org-scoped API keys (admin scope)
```

Default scopes requested during login:
```
user:profile user:inference user:sessions:claude_code user:mcp_servers user:file_upload org:create_api_key
```

OAuth version identifier: `oauth-2025-04-20`

## 4. Authorization Flow (PKCE)

### Step 1: Generate PKCE Pair

```typescript
import { randomBytes, createHash } from 'crypto';

const codeVerifier = randomBytes(32).toString('base64url');
const codeChallenge = createHash('sha256')
  .update(codeVerifier)
  .digest('base64url');
```

### Step 2: Build Authorize URL

```
https://claude.com/cai/oauth/authorize
  ?response_type=code
  &client_id=9d1c250a-e61b-44d9-88ed-5944d1962f5e
  &code_challenge={codeChallenge}
  &code_challenge_method=S256
  &redirect_uri=http://localhost:{port}/callback
  &scope=user:profile user:inference user:sessions:claude_code user:mcp_servers user:file_upload org:create_api_key
  &state={random_state}
```

Optional params:
- `orgUUID` - target organization
- `login_hint` - pre-fill email
- `login_method` - force login method

### Step 3: Local HTTP Server

Start a local HTTP server on a random port to receive the callback:

```
GET http://localhost:{port}/callback?code={auth_code}&state={state}
```

### Step 4: Token Exchange

```http
POST https://platform.claude.com/v1/oauth/token
Content-Type: application/json

{
  "grant_type": "authorization_code",
  "code": "{auth_code}",
  "redirect_uri": "http://localhost:{port}/callback",
  "client_id": "9d1c250a-e61b-44d9-88ed-5944d1962f5e",
  "code_verifier": "{codeVerifier}",
  "state": "{state}"
}
```

Optional: `"expires_in": {seconds}` to request specific TTL.

Timeout: 15 seconds.

### Step 5: Token Response

```json
{
  "access_token": "sk-ant-oat-...",
  "refresh_token": "sk-ant-ort-...",
  "expires_in": 86400
}
```

## 5. Token Refresh

```http
POST https://platform.claude.com/v1/oauth/token
Content-Type: application/json

{
  "grant_type": "refresh_token",
  "refresh_token": "{refresh_token}",
  "client_id": "9d1c250a-e61b-44d9-88ed-5944d1962f5e",
  "scope": "user:profile user:inference user:sessions:claude_code user:mcp_servers user:file_upload"
}
```

Response: new `access_token` + optionally rotated `refresh_token`.

If the response includes a new `refresh_token`, it replaces the old one. If omitted, the old one remains valid.

Timeout: 15 seconds.

## 6. Using the Token

### As Bearer Token (direct API call)

```http
POST https://api.anthropic.com/v1/messages
Authorization: Bearer {access_token}
Content-Type: application/json
```

### As authToken in SDK

The Anthropic SDK supports `authToken` param (separate from `apiKey`):

```typescript
import Anthropic from '@anthropic-ai/sdk';

const client = new Anthropic({
  authToken: accessToken,  // OAuth token
  // apiKey is NOT needed when using authToken
});
```

When `authToken` is set, the SDK sends `Authorization: Bearer {token}` instead of `X-Api-Key`.

## 7. Credential Storage (macOS)

Stored in macOS Keychain:

| Field | Value |
|-------|-------|
| Service | `Claude Code-credentials` |
| Account | `{unix_username}` |
| Format | JSON |

If `CLAUDE_CONFIG_DIR` is set, a SHA256 hash suffix is appended to the service name:
`Claude Code-credentials-{sha256(configDir).slice(0,8)}`

### Keychain JSON Structure

```json
{
  "claudeAiOauth": {
    "accessToken": "sk-ant-oat-...",
    "refreshToken": "sk-ant-ort-...",
    "expiresAt": 1774969485563,
    "scopes": [
      "user:file_upload",
      "user:inference",
      "user:mcp_servers",
      "user:profile",
      "user:sessions:claude_code"
    ],
    "subscriptionType": "max",
    "rateLimitTier": "default_..._20x"
  }
}
```

### Read from Keychain (macOS)

```bash
security find-generic-password -s "Claude Code-credentials" -a "$(whoami)" -w
```

### Write to Keychain (macOS)

```bash
security add-generic-password -U -s "Claude Code-credentials" -a "$(whoami)" -w '{json}'
```

### Non-macOS

Falls back to `~/.claude/.credentials.json` (plaintext).

## 8. Environment Variable Overrides

Claude Code checks these env vars (in order of priority):

| Variable | Purpose |
|----------|---------|
| `CLAUDE_CODE_OAUTH_TOKEN` | Skip keychain, use this access token directly |
| `CLAUDE_CODE_OAUTH_REFRESH_TOKEN` | Provide refresh token via env |
| `CLAUDE_CODE_OAUTH_TOKEN_FILE_DESCRIPTOR` | Read token from file descriptor |
| `CLAUDE_CODE_OAUTH_SCOPES` | Override default scopes |
| `CLAUDE_CODE_CUSTOM_OAUTH_URL` | Custom OAuth provider URL |
| `CLAUDE_CODE_OAUTH_CLIENT_ID` | Override the hardcoded client ID |
| `ANTHROPIC_AUTH_TOKEN` | Used by Anthropic SDK as Bearer token |

**Key insight for proxy**: Setting `CLAUDE_CODE_OAUTH_TOKEN` on any device bypasses the entire login flow. The proxy just needs to serve fresh tokens.

## 9. 401 Auto-Refresh Logic

When the API returns 401:

1. Read latest token from keychain (may have been refreshed by another process)
2. If keychain token differs from the one that got 401 → retry with new token
3. If same → call `refresh_token` grant → store new tokens → retry
4. On `403` with `forceRefreshOnFailure` → also triggers refresh

## 10. Proxy Design Notes

### Minimal Proxy Architecture

```
┌─────────────────────────────────────────┐
│  claude-proxy (runs on main machine)    │
│                                         │
│  ┌─────────────┐  ┌──────────────────┐  │
│  │ Token Store  │  │ Refresh Worker   │  │
│  │ (in-memory)  │  │ (periodic, or    │  │
│  │              │  │  on 401)         │  │
│  └──────┬───────┘  └────────┬─────────┘  │
│         │                   │            │
│  ┌──────┴───────────────────┴─────────┐  │
│  │        HTTP Reverse Proxy          │  │
│  │  localhost:8300 → api.anthropic.com │  │
│  │  Injects: Authorization: Bearer    │  │
│  └────────────────────────────────────┘  │
└─────────────────────────────────────────┘
```

### Client Configuration (on each device)

```bash
# Point Claude Code at the proxy
export ANTHROPIC_BASE_URL=http://{proxy-ip}:8300
export CLAUDE_CODE_OAUTH_TOKEN=dummy  # bypass local login check
# or
export ANTHROPIC_AUTH_TOKEN=dummy     # SDK-level bypass
```

Or, simpler — proxy periodically syncs the token and devices just read it:

```bash
# On each device, cron job or systemd timer
export CLAUDE_CODE_OAUTH_TOKEN=$(curl -s http://{proxy-ip}:8300/token)
```

### Token Lifecycle

| Event | Action |
|-------|--------|
| Startup | Read from keychain, store in memory |
| Every ~12h (or before expiry) | Refresh token proactively |
| On 401 from upstream | Refresh immediately, retry |
| Refresh token rotated | Update keychain + in-memory store |

### Security Considerations

- Proxy should only listen on trusted network (LAN / tailscale)
- Add a simple bearer token or mTLS for proxy-to-client auth
- Never expose the proxy to the public internet
- Refresh tokens are long-lived — treat them like passwords

## 11. Rate Limits

The `rateLimitTier` in the credential indicates the subscription tier. Rate limits are enforced server-side per account, not per device. Multiple devices sharing one account share the same rate limit pool.

## 12. Related Endpoints

| Endpoint | Purpose |
|----------|---------|
| `POST /api/oauth/claude_cli/create_api_key` | OAuth token → short-lived API key |
| `POST /api/oauth/claude_cli/roles` | Get org/workspace roles for the OAuth user |
| `GET /api/oauth/profile` | Get user profile info |
| `POST /api/claude_cli_feedback` | Submit feedback |
| `POST /api/claude_code/metrics` | Usage metrics |

## Appendix A: Quick Token Extraction (macOS)

```bash
#!/bin/bash
# Extract current access token from keychain
security find-generic-password -s "Claude Code-credentials" -a "$(whoami)" -w \
  | python3 -c "import sys,json; print(json.loads(sys.stdin.read())['claudeAiOauth']['accessToken'])"
```

## Appendix B: Quick Token Refresh (standalone)

```bash
#!/bin/bash
REFRESH_TOKEN="$1"
CLIENT_ID="9d1c250a-e61b-44d9-88ed-5944d1962f5e"
TOKEN_URL="https://platform.claude.com/v1/oauth/token"

curl -s -X POST "$TOKEN_URL" \
  -H "Content-Type: application/json" \
  -d "{
    \"grant_type\": \"refresh_token\",
    \"refresh_token\": \"$REFRESH_TOKEN\",
    \"client_id\": \"$CLIENT_ID\",
    \"scope\": \"user:profile user:inference user:sessions:claude_code user:mcp_servers user:file_upload\"
  }"
```
