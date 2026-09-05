# Issue 32: CLI — run and plan orchestration UX

## Title

Implement `run`/`plan` commands: progress rendering, cost confirmation, filters

## Summary

Implement `cli/run.py` per DESIGN.md §9/§6.4/§7.6: the primary pipeline command with
`--until/--only/--from/--target/--dry-run/--yes/--require-approved`, rich progress
bound to engine events, the paid-work confirmation gate, cloud badges, and SIGINT
cancellation.

## Context

This is the product's main interaction. The engine (10) does the work; this issue is
the UX contract: what users see, when they're asked, how interruption behaves.

## Scope

In: run/plan commands + progress renderer + confirmation flow + SIGINT wiring.
Out: engine semantics (10), stage behavior (21–30).

## Detailed Requirements

1. Argument mapping to engine planner: `--until S`, `--only S`, `--from S`
   (mutually exclusive trio → UsageError), `--target L` (repeatable; must ⊆ manifest
   targets), `--dry-run`. `plan` command = `run --dry-run` alias.
   Stage arguments accept **base stage names only** (`ingest`, `separate`,
   `transcribe`, `translate`, `voice_ref`, `synthesize`, `fit`, `mix`, `export`,
   `subtitles`) — never keyed forms like `translate:en`; languages are selected via
   `--target`. Unknown stage → UsageError listing valid names. Planner semantics are
   exactly issue 10's filter rules (e.g. `run --until translate --target en` plans
   ingest→separate→transcribe→translate:en; `run --only synthesize` on a
   two-target project plans synthesize:en and synthesize:de, erroring per issue 10
   when deps are unmet).
2. Plan rendering (dry-run and pre-run header): ordered stage list with reason
   (pending/stale/failed), per-stage `[cloud]` badge (§11.5), per-stage cache summary
   ("12/40 segments cached"), aggregate cost estimate with breakdown lines and the
   "approximate" label (§7.6).
3. Confirmation gate: when estimate > `cost.confirm_over_usd` → TTY prompt
   `Proceed? [y/N]` (default No); non-TTY without `--yes` →
   `CostConfirmationRequired DS-COST-001` exit 11; `--yes` skips. Estimate `None`
   (unknown) counts as 0 but renders "unknown" and never triggers the gate (documented
   behavior).
4. `--require-approved`: before executing, for every planned `synthesize:<lang>`,
   check that language's translation doc: missing doc → fail (exit 9,
   `DS-STAGE-009`) with "translations not generated yet — run `dubstudio run
   --until translate` and review first"; existing doc with non-`approved` segments →
   fail listing those ids per language (§4.3). The check runs pre-execution so
   nothing partial happens.
5. Progress: rich `Progress` — one task line per running stage (percentage from
   `stage_progress` events; segment counters for synthesize/fit from
   `segment_completed`), warnings streamed beneath (overflow, overrun, collisions),
   quiet mode = final summary only. Non-TTY: line-per-event plain logs (no control
   codes).
6. Run summary block on completion/failure: per-stage durations, segments synthesized
   (cached/new), actual usage and fit result counts from the `run_completed` /
   `run_failed` event's `data` payload (issue 10's runner aggregates them there —
   §6.5/§7.6; no separate "usage" event type exists), output paths
   (export/subtitles), and — on failure — the §12-rendered error + next action.
7. SIGINT: first Ctrl-C → `runner.cancel()` + "finishing current segment…" notice;
   second → hard exit 10 (after best-effort lock release via context managers).
8. Interactive consent trigger: when a planned stage requires consent and status is
   missing, the run prompts (via issue 11 flow) **before** starting any stage, so
   long pipelines never die mid-way on the gate (plan-time detection).

## Acceptance Criteria

- [ ] CliRunner tests with stub engine: filter trio exclusivity; target validation;
      dry-run prints plan and exits 0 without side effects (engine `execute` not
      called).
- [ ] Cost gate matrix: TTY-yes / TTY-no (abort, exit 11, nothing ran) / non-TTY
      without `--yes` (exit 11) / with `--yes` (runs) — stub estimates.
- [ ] `--require-approved` failure lists exact ids (fixture translation doc).
- [ ] SIGINT test (subprocess, real signal): exit 10, manifest shows rolled-back
      stage, partial artifacts on disk, lock released.
- [ ] Consent prompted at plan time (stub: consent missing + synthesize planned →
      prompt happens before ingest starts).
- [ ] Non-TTY output contains no ANSI control sequences (regex check).

## Validation

`uv run pytest tests/cli/test_run.py` (stub engine + one subprocess SIGINT test);
manual full run against mock providers.

## Dependencies

10, 13, 11 (plan-time consent), 31 (shared plumbing) — hard; behavioral completeness
with 21–30. ISSUE_PLAN row matches.

## Non-goals

Parallel multi-language execution, watch mode, TUI dashboards.

## Design References

DESIGN.md §9, §6.4–6.5, §7.6, §11.5, §12.
