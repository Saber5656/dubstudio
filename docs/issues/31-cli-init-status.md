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
   - lang args validated (BCP-47 primary subtag); source==target → UsageError;
   - delegates to `ProjectStore.create` (09); prints a next-steps block: consent
     status hint (if missing), provider/key readiness summary (from registry
     `list_all`), and `dubstudio run`;
   - warns (not fails) when selected providers lack keys/extras.
3. `status [--json]`: renders the manifest stage table — rows per stage (per-lang
   grouped), status glyphs, fingerprint freshness (calls engine `evaluate`, marking
   would-be-stale rows), last error code+message, warnings summary from the latest
   run log (fit overflow count, overrun chars), and a "next action" line (first
   pending/stale stage → suggested command). `--json`: statuses + evaluation + next
   action in a stable schema.
4. `config show [--json]`: effective merged config (issue 06) with provenance
   annotation per key group (default/user/project/env) in human mode; secrets never
   present by construction (§8.3).
5. `providers list [--json]`: registry `list_all` rows (name, kind, mode,
   builtin/plugin, enabled, key set?, extra installed?, watermark for tts, languages
   summary).

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

`uv run pytest tests/cli/test_init.py test_status.py test_config_show.py
test_providers_list.py` via `typer.testing.CliRunner`.

## Dependencies

09, 07, 13, 06, 10 (evaluate for status).

## Non-goals

Interactive init wizard, project migration, editing config via CLI.

## Design References

DESIGN.md §9, §4.1, §8, §12.
