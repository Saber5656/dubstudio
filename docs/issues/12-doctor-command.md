# Issue 12: doctor command

## Title

Implement `dubstudio doctor` environment diagnostics

## Summary

Implement `cli/doctor.py`: a read-only diagnostic command reporting ffmpeg presence and
version, API-key presence (names only), optional-extra importability, provider
healthchecks, GPU/MPS availability, disk space, and config validity — human table and
`--json`.

## Context

Setup friction is the #1 support cost for local-first audio tools (DESIGN.md §1.2, §16);
doctor turns it into a checklist. It also surfaces the `watermark_builtin` and consent
status so the safety posture is visible.

## Scope

In: the command + check framework. Out: fixing anything automatically; network calls
beyond provider `healthcheck()` (which must be opt-in via `--network`).

## Detailed Requirements

1. Check framework: `Check` dataclass (id, title, status: ok|warn|fail|skip, detail,
   hint); checks run isolated (one crash → that check `fail`, others continue).
2. All binary probes (`ffmpeg -version`, `ffprobe -version`) go through the issue 07
   safe runner (`procs.run`: argv list, env allowlist, timeout 10 s) — never
   `shell=True`, never inheriting API-key env (§11.3).
3. Checks (exact ids):
   `ffmpeg.present`, `ffmpeg.version` (≥ 6 ok; < 6 warn), `ffprobe.present`,
   `config.valid` (loads merged config; DS-CONFIG errors → fail with message),
   `project.detected` (skip when not in a project),
   `keys.openai` / `keys.elevenlabs` (env presence only — **never** print values;
   skip when provider unselected),
   `extras.local-asr` / `extras.separate` / `extras.local-tts` (importable?),
   `gpu.cuda` / `gpu.mps` (torch-based detection only when torch importable; else skip),
   `disk.free` (drive of the detected project ≥ 10 GB ok, ≥ 2 GB warn, else fail;
   **skips with reason when not inside a project**),
   `consent.status` (accepted/outdated/missing — informational warn when missing),
   `providers.selected` (each selected provider resolvable in registry, capability
   check for configured languages — plan-time check reuse),
   `providers.health.<name>` — one entry **per selected provider** always present:
   status `skip` (reason "requires --network") in offline runs, real
   `healthcheck()` result (timeout 10 s) with `--network`.
4. Output: rich table grouped by section with ✓/!/✗; exit code 0 when no `fail`
   (warns allowed), else 12. `--json`: list of Check dicts (stable ids for scripting).
5. `--network` flag gates any outbound request; default fully offline.

## Acceptance Criteria

- [ ] On a machine without ffmpeg (PATH stripped in test), doctor exits 12 and the
      ffmpeg check carries the install hint.
- [ ] `--json` output parses and contains every static check id above plus one
      `providers.health.<name>` entry per selected provider (`skip` when offline).
- [ ] No API key value ever appears in output (test sets a key and greps output).
- [ ] Without `--network`, no socket is opened (respx/socket-guard test).
- [ ] With mock provider registry, `providers.selected` fails when config names an
      unknown provider and hints `providers list`.

## Validation

`uv run pytest tests/cli/test_doctor.py`; manual run on macOS with/without extras.

## Dependencies

06, 07, 11 (consent status read), 13 (registry + healthcheck interface) — all hard;
ISSUE_PLAN dependency row matches this list.

## Non-goals

Auto-fix, model downloads, benchmark measurements (§15 validation is U-04, manual).

## Design References

DESIGN.md §9 (doctor row, exit 12), §16, §7.1 (healthcheck), §11.5.
