# Issue 31: CLI — init, status, config show, providers list

## Title

Implement `init`, `status`, `config show`, and `providers list` commands

## Summary

Implement the project-lifecycle and introspection commands per DESIGN.md §9, wiring
store/config/registry into user-facing Typer commands with `--json` machine output and
the CLI conventions (non-TTY fails closed, exit codes, error rendering).

## Context

`init` is the first command every user runs; its validation quality and the template
`dubstudio.toml` it writes define the onboarding experience.

## Scope

In: four commands + shared CLI plumbing (`--project`, `--json`, `--quiet/-v`, error
rendering via issue 05 `render_error`, exit-code mapping at the Typer app boundary).
Out: run/plan (32), segments/voice/invalidate (33), doctor (12), consent (11), ui (39).

## Detailed Requirements

1. Shared plumbing in `cli/app.py`: global options; a single top-level exception
   handler mapping `DubstudioError.exit_code`, printing via `render_error`; unexpected
   exceptions → §12 crash policy (short message + log path, `-v` traceback, exit 9).
2. `init [DIR] --input F --source-lang S --target-lang T (repeatable) [--copy-input]`:
   - DIR default = input basename stem; refuses existing non-empty dir
     (DS-CONFIG-002);
   - validates input via probe before creating anything (fail → nothing written);
   - lang args validated and normalized **via issue 06's `normalize_lang` semantics
     exactly** (so `EN` → `en`, `en-US` accepted); `source == target` compared
     after normalization → UsageError;
   - delegates to `ProjectStore.create` (09); prints a next-steps block: consent
     status hint (if missing), provider/key readiness summary (from registry
     `list_all`), and `dubstudio run`;
   - warns (not fails) when selected providers lack keys/extras.
3. `status [--json]`: renders the manifest stage table — rows per stage (per-lang
   grouped), status glyphs, fingerprint freshness (calls engine `evaluate` per
   DESIGN §6.1–6.2, marking would-be-stale rows), last error code+message, a
   warnings summary, and a "next action" line (first pending/stale stage →
   suggested command). Warnings are derived **from artifacts, not run logs**
   (deterministic): fit overflow/shortened counts from each `fit_report.json`;
   over-budget counts computed from each translation doc (`len(text) >
   char_budget`). Missing artifacts → that summary line is omitted.
   `--json` schema (exact):
   `{"project": str, "languages": {"source": str, "targets": [str]},
   "stages": [{"key": str, "status": str, "evaluated": "up_to_date|pending|stale|
   failed|blocked|skipped", "finished_at": str|null, "error": {"code": str,
   "message": str}|null}], "warnings": {"fit_overflow": {"<lang>": int},
   "over_budget": {"<lang>": int}}, "next_action": str|null}`.
4. `config show [--json]`: effective merged config (issue 06) with provenance
   annotation per key group (default/user/project/env) in human mode; secrets never
   present by construction (§8.3). `--json`: `{"config": <effective model dump>,
   "provenance": {"<table>": "default|user|project|env|mixed"}}`.
5. `providers list [--json]`: registry `list_all` rows. `--json`: `{"providers":
   [{"name": str, "kind": str, "mode": "local|cloud", "origin": "builtin|plugin",
   "enabled": bool, "selected": bool, "key_env_set": bool|null, "extra_installed":
   bool|null, "watermark_builtin": bool|null, "languages": "…"|null}]}` (null =
   not applicable / unknown-until-loaded for disabled plugins).

## Acceptance Criteria

- [ ] `init` happy path creates the §4.1 tree; CliRunner golden output includes
      next-steps; non-empty dir → exit 4; bad lang → exit 2; probe failure → exit 5
      and **no** directory left behind.
- [ ] `status` on fresh project shows all pending + next action `dubstudio run`;
      after hand-marking a stage completed with a stale fingerprint (test fixture),
      the row shows stale.
- [ ] `status --json` schema snapshot test.
- [ ] `config show` provenance correct across the 4 layers (fixture envs);
      `providers list` reflects a fake plugin as discovered-but-disabled.
- [ ] All commands honor `--project PATH` from outside the project dir.
- [ ] Exit-code mapping test: each DubstudioError subclass raised inside a command
      exits with its §9 code.

## Validation

`uv run pytest tests/cli/test_init.py tests/cli/test_status.py
tests/cli/test_config_show.py tests/cli/test_providers_list.py` via
`typer.testing.CliRunner`.

## Dependencies

06, 07, 09, 10 (`evaluate` for status), 13 — matches the ISSUE_PLAN row.

## Non-goals

Interactive init wizard, project migration, editing config via CLI.

## Design References

DESIGN.md §9, §6.1–6.2 (status semantics), §4.1, §8, §12.
