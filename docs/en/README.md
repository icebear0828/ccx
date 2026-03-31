# ccx

**Claude Code Exchange** — an OAuth reverse proxy that lets multiple devices share a single Claude Code subscription.

[中文](../../README.md)

```
Device A (claude code) ──┐
Device B (claude code) ──┼──→ ccx ──→ api.anthropic.com
Device C (headless)    ──┘   (token management
                              + auto-refresh)
```

## What it does

- Holds OAuth tokens centrally on one machine
- Auto-refreshes tokens before expiry (or on 401)
- Reverse-proxies `api.anthropic.com` with injected `Authorization: Bearer` header
- Clients just point `ANTHROPIC_BASE_URL` at ccx — no per-device login needed

## Client setup

On each device:

```bash
export ANTHROPIC_BASE_URL=http://<ccx-host>:8300
export CLAUDE_CODE_OAUTH_TOKEN=dummy  # bypass local login
```

## Security

- Only run ccx on a trusted network (LAN / Tailscale)
- Never expose to the public internet
- Refresh tokens are long-lived — treat them like passwords

## License

MIT
