# Risk Analysis — Account Ban Factors

[中文](../../RISK.md)

> Understanding what triggers account restrictions when proxying Claude Code OAuth tokens.

## Core Issue

Claude Pro/Max is a per-seat subscription. A proxy that shares one account across multiple devices (or users) fundamentally conflicts with the Terms of Service. Everything below is a detection vector for this behavior.

---

## High Risk

### 1. Concurrent Requests from Multiple IPs

The upstream server sees a single OAuth token issuing requests from different source IPs simultaneously. This is the most obvious signal of token sharing and is trivial to detect server-side.

**Mitigation:** Route ALL traffic through the proxy's single egress IP. Never let clients call `api.anthropic.com` directly.

### 2. Token Sharing Violates ToS

Regardless of technical detection, sharing a subscription seat across devices/users is a policy violation. If Anthropic reviews the account for any reason, the usage pattern speaks for itself.

**Mitigation:** None — this is an inherent risk of the architecture.

### 3. Refresh Token Storm

When multiple clients hit 401 simultaneously, naive implementations will fire multiple refresh requests in parallel. A burst of `grant_type=refresh_token` calls in a short window is abnormal and triggers rate limiting or fraud detection.

**Mitigation:** Implement a refresh lock — only one refresh request at a time. Queue other callers until the refresh completes.

---

## Medium Risk

### 4. Abnormal Request Volume

Even within the `rateLimitTier` hard limits, sustained high-frequency usage (aggregated from multiple devices) can exceed what a single human reasonably produces. Soft limits and manual reviews catch this.

**Mitigation:** Add a rate limiter at the proxy layer. Keep aggregate throughput within single-user norms.

### 5. IP Geolocation Jumps

If clients are on different networks (home LAN, office, cloud server), the same token appears from geographically distant IPs within short intervals — similar to "impossible travel" detection used in fraud systems.

**Mitigation:** All traffic through the proxy (single egress). If clients are remote, use a VPN/Tailscale tunnel back to the proxy rather than forwarding tokens.

### 6. Inconsistent Client Fingerprints

Different devices send different `User-Agent` strings, OS identifiers, and Claude Code versions. A single account rapidly alternating between `darwin/arm64` and `linux/x86_64` is suspicious.

**Mitigation:** Have the proxy normalize outbound headers — rewrite `User-Agent` and strip or unify client-identifying headers.

---

## Low Risk

### 7. Unusual Session Patterns

Claude Code creates sessions (`user:sessions:claude_code` scope). Multiple overlapping sessions with different characteristics (different working directories, simultaneous tool calls) may be flagged.

### 8. Token Lifetime Anomalies

If the proxy aggressively refreshes tokens well before expiry (e.g., every hour when TTL is 24h), the refresh pattern deviates from the official client's behavior.

**Mitigation:** Mirror the official client's refresh strategy — refresh at ~50-75% of TTL or on 401, not eagerly.

### 9. Missing Telemetry

Claude Code sends metrics to `/api/claude_code/metrics` and feedback to `/api/claude_cli_feedback`. If the proxy blocks or doesn't forward these, the account shows API usage with zero telemetry — a gap that could flag automated/proxy usage.

**Mitigation:** Pass telemetry requests through transparently.

---

## Summary: Defense in Depth

| Layer | Action |
|-------|--------|
| Network | Single egress IP, all traffic through proxy |
| Headers | Normalize User-Agent and client fingerprints |
| Refresh | Mutex lock, one refresh at a time |
| Rate | Limit aggregate requests to single-user norms |
| Telemetry | Forward all metrics/feedback transparently |
| Behavior | Avoid true parallel requests from multiple clients |

The safest configuration: all devices on the same LAN (or Tailscale network), proxy as the sole egress point, with request queuing to serialize upstream calls.
