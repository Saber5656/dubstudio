# Issue 39: CLI — ui command

## Title

Implement `dubstudio ui`: launch the review server with token URL and browser open

## Summary

Implement `cli/ui_cmd.py` per DESIGN.md §9/§10.2: start the UI server for the current
project, print the tokenized URL exactly once, open the browser (unless
`--no-browser`), and shut down cleanly on Ctrl-C.

## Context

The glue between CLI and UI surfaces; owns the token lifecycle start (34 owns its
verification).

## Scope

In: the command + lifecycle. Out: server internals (34–38).

## Detailed Requirements

1. `dubstudio ui [--port N] [--no-browser] [--project PATH]`:
   - opens the project store (must be a valid project → else DS-CONFIG errors as
     usual);
   - generates `secrets.token_urlsafe(32)`; builds `create_app` (34); starts uvicorn
     on `127.0.0.1:<port|ui.port|0>`; on port-in-use with explicit `--port` →
     `EnvMissingError`-style clear error (exit 12) naming the port; port 0 → resolve
     actual;
   - prints exactly one line containing the URL
     `http://127.0.0.1:<port>/?token=<token>` plus a note that the token grants
     access to this project for this session (the only place the token is ever
     printed; NOT written to logs — assert via redaction-style test that the JSONL
     log contains no token);
   - `--no-browser` skips `webbrowser.open`; otherwise open after the server is
     accepting connections (readiness probe on the socket, ≤ 5 s);
   - runs until SIGINT/SIGTERM → graceful uvicorn shutdown (in-flight requests ≤ 5 s),
     exit 0; a second SIGINT forces exit 10.
2. Server lifetime notes printed at start: single project, localhost-only, how to
   stop. `--json` not applicable (interactive command; flag absent by design).
3. Mutating job execution inside the server takes the project lock (10/36); `ui`
   itself does not hold the lock while idle — CLI runs in another terminal are
   possible when no job is active (document in command help).
4. Token regeneration: every launch = new token; old URLs die with the process.

## Acceptance Criteria

- [ ] Subprocess test: launch with `--no-browser --port 0`, parse the printed URL,
      complete the handshake (`/?token=…` → 303 with Set-Cookie, then `/` → 200 with
      the client following redirects), hit a served route 200, SIGINT → exit 0,
      port released.
- [ ] SIGTERM → graceful shutdown exit 0 within the 5 s grace window; SIGINT twice
      in quick succession → forced exit 10.
- [ ] Token appears exactly once in stdout and never in the JSONL log (scan test).
- [ ] Explicit busy port → exit 12 with the named port; port 0 works.
- [ ] `--no-browser` verified (webbrowser mocked in unit test; subprocess test uses
      the flag anyway).
- [ ] Launch outside a project → the standard not-a-project error (exit 4).

## Validation

`uv run pytest tests/cli/test_ui_cmd.py` (unit + one subprocess lifecycle test).

## Dependencies

34, 31 (CLI plumbing: `--project`, error rendering) — matches the ISSUE_PLAN row;
35–38 make the UI useful but are not build-order prerequisites.

## Non-goals

Daemon mode, multiple projects, TLS, custom bind address (ADR-004).

## Design References

DESIGN.md §9, §10.2; ADR-004.
