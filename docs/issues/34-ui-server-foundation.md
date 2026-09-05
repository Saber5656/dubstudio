# Issue 34: UI server foundation

## Title

Implement FastAPI UI server foundation: token auth, CSRF, security headers, static

## Summary

Implement `ui/server.py`, `ui/auth.py`: the localhost-only FastAPI application with
the §10.2 token→cookie handshake, CSRF double-submit enforcement, Host-header
allowlist, security headers, and static frontend serving — with every §11.2 B3 control
covered by negative tests.

## Context

This is the product's only network listener. ADR-004 commits us to a Jupyter-grade
localhost threat model; all API routers (35/36) mount onto this foundation.

## Scope

In: app factory, auth/session, middleware, static mount, uvicorn runner wiring.
Out: business routes (35/36), browser launch UX (39).

## Detailed Requirements

1. `create_app(store: ProjectStore, token: str) -> FastAPI` (the caller — issue 39 —
   opens the store and reads `ui.port` from config), plus the runner helper this
   issue also owns: `run_ui_server(app, port: int) -> BoundServer` — builds a
   uvicorn `Server` bound to **`127.0.0.1` only** (host not parameterizable —
   ADR-004), binds the socket eagerly (port 0 = ephemeral) and exposes
   `BoundServer.port` (the resolved port), `.serve()` (blocking) and
   `.shutdown(grace_s=5)`.
2. Handshake (§10.2): `GET /?token=<t>` — constant-time compare; valid → set cookie
   `dubstudio_session=<t>` (HttpOnly, SameSite=Strict, Path=/; no Secure flag —
   http://127.0.0.1) → 303 redirect to `/`; invalid/absent token without cookie →
   401 minimal HTML ("launch via `dubstudio ui`"), no token echo.
3. Auth middleware: every route except the handshake requires the valid cookie;
   mutating methods (POST/PATCH/DELETE) additionally require header
   `X-Dubstudio-Csrf: <token>` equal to the session token → else 403
   (JSON error, stable shape `{error: {code, message}}`).
4. Host guard middleware: `Host` ∉ {`127.0.0.1[:port]`, `localhost[:port]`} → 421.
   Applies to every request including the handshake.
5. Security headers on all responses: `Content-Security-Policy: default-src 'self';
   img-src 'self' data:; media-src 'self' blob:`, `X-Content-Type-Options: nosniff`,
   `Referrer-Policy: no-referrer`. Caching: `Cache-Control: no-store` on `/api/*`;
   `Cache-Control: no-cache` (revalidate) on `/` and `/static/*`. No CORS headers
   ever.
6. Static serving: `/` (index.html) and `/static/*` from packaged
   `dubstudio/ui/static/` (importlib.resources); correct content types; no
   directory listing; unknown paths → 404 (single-page app, no SPA fallback).
   `index.html` is served as a template with the placeholder `__CSRF__` replaced by
   the session token at response time, rendered as `<meta name="csrf"
   content="…">` — this is the documented hook the frontend (issues 37/38) reads to
   set `X-Dubstudio-Csrf` (the cookie itself is HttpOnly and unreadable by JS).
7. Error shape: all HTTP errors JSON `{error: {code, message}}` except the 401
   handshake HTML; unexpected exceptions → 500 with generic message (details only in
   server log, redaction filter active).
8. Single-project invariant: app state carries the `ProjectStore` opened at launch;
   no route accepts a path/project parameter (B6 by construction).
9. App logs through issue 05 logging (uvicorn access logs at DEBUG only).

## Acceptance Criteria

Auth/CSRF matrix is exercised against a **test-only probe router** (`GET/POST
/api/_probe`, mounted only inside `tests/ui/test_server_foundation.py` via
`create_app`'s returned app — business routes arrive in issues 35/36):
- [ ] No cookie → 401 on `/api/_probe`; wrong token in handshake → 401 without echo.
- [ ] Valid cookie but missing/wrong CSRF header on POST → 403; GET without CSRF → 200.
- [ ] `Host: evil.example` → 421 even with valid cookie (rebinding proof).
- [ ] CSP/nosniff headers on `/` and `/api/_probe`; `no-store` on `/api/_probe`,
      `no-cache` on `/`; no `Access-Control-Allow-*` anywhere.
- [ ] `/static/../pyproject.toml` style traversal → 404 (packaged-resources access
      cannot escape).
- [ ] Cookie flags: HttpOnly + SameSite=Strict asserted.
- [ ] Served `/` contains `<meta name="csrf" content="<token>">` exactly once.
Positive:
- [ ] Handshake sets cookie + redirects; subsequent probe GET 200.
- [ ] `run_ui_server` with port 0 exposes the real bound port; bind address is
      127.0.0.1 (socket assertion).

## Validation

`uv run pytest tests/ui/test_server_foundation.py` (FastAPI TestClient; one uvicorn
socket test asserting the bind address is 127.0.0.1).

## Dependencies

05, 06, 09 (store type in the app state) — matches the ISSUE_PLAN row.

## Non-goals

HTTPS, remote access, multi-project, websockets (SSE only, issue 36), auth beyond the
token model.

## Design References

DESIGN.md §10.1–10.2, §11.2 B3;
`docs/decisions/ADR-004-local-first-security-posture.md`.
