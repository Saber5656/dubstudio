# ADR-004: Local-first security posture and localhost token-authenticated UI

- Status: accepted
- Date: 2026-07-10

## Context

dubstudio handles two sensitive asset classes on end-user machines: provider API keys
and the user's own media. It also exposes a local web UI, which historically (Jupyter,
various dev tools) is a real attack surface via CSRF/DNS-rebinding from arbitrary web
pages running in the same browser.

## Decision

1. **Secrets**: API keys only from env vars / untracked `.env`; TOML config rejects
   key-like fields; global redaction filter in all logs and errors (DESIGN.md §8.3, §12).
2. **UI**: bind 127.0.0.1 only (not configurable in v1); random per-launch bearer token
   exchanged for an HttpOnly SameSite=Strict cookie; CSRF double-submit header on
   mutations; Host-header allowlist; CSP `default-src 'self'`; no external assets; no
   client-supplied filesystem paths (id-based routes only) (DESIGN.md §10.2, §11.2 B3).
3. **Subprocess policy**: argv lists only, env allowlist, timeouts (DESIGN.md §11.3).
4. **Model downloads**: pinned sources, safetensors preferred, TOFU sha256 recording
   (§11.2 B4).
5. **No telemetry**; `plan` marks which stages send data off-machine (§11.5).

## Consequences

- Remote/LAN access to the UI is impossible in v1 (documented; SSH port-forward is the
  workaround). This is accepted to keep the threat model small.
- Some UI conveniences (external CDN assets) are forbidden by design.

## Alternatives considered

- Configurable bind address with password auth: expands threat model (TLS, credential
  storage) for marginal v1 value — deferred.
