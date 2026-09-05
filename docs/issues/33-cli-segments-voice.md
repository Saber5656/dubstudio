# Issue 33: CLI — segments round-trip, voice, invalidate

## Title

Implement `segments export/import`, `voice set/auto/show`, and `invalidate` commands

## Summary

Implement the human-in-the-loop file editing commands per DESIGN.md §9: exporting
translations for external editing, importing them back with validation and precise
staleness, managing the reference voice, and manual invalidation (including cloud
voice cleanup).

## Context

The file-based HITL loop is the CLI counterpart of the review UI (P2). Import
validation is a trust boundary (B6): edited files are untrusted input.

## Scope

In: five commands + import validation. Out: translation stage merge policy (24),
voice_ref stage (25), UI PATCH equivalent (36).

## Detailed Requirements

1. `segments export --lang L [--to F]` (default `segments_review.<lang>.json`, CWD):
   emits `{schema_version: 1, project_id, lang, segments: [{id, start_ms, end_ms,
   source_text, text, status, char_budget, over_budget: bool}]}` — readable, ordered,
   with timing context for editors. Refuses when `translate:<lang>` not completed
   (`DS-STAGE-001` "run translate first").
2. `segments import --lang L --from F [--dry-run]`:
   - validates: parses via model, project_id match, lang match, text non-empty,
     status ∈ enum. Id rules (exact): an id in the file that does **not** exist in
     the current translation doc → rejected (itemized `ConfigError DS-CONFIG-006`);
     a duplicate id within the file → rejected (itemized); a current-doc id
     **omitted** from the file → allowed, means "unchanged";
   - diff report: per changed segment old→new text (truncated 60 chars) and status
     changes; `--dry-run` stops here;
   - apply: changed text → status `edited` (unless file explicitly sets `approved`),
     unchanged text with status change honored; write translation doc atomically under
     lock; call engine per-segment invalidation so exactly the changed ids re-run in
     synthesize/fit (§6.2);
   - summary: N changed, M approved, downstream stages now stale.
3. `voice show`: current mode, reference file/provenance summary from
   `reference.json` (spans count, total seconds, built_at) or "not built".
4. `voice set --ref F`: validates file per §5.5 rules (via voice_ref validation
   helper), **resolves F to a canonical absolute path at command time**
   (`Path(F).expanduser().resolve()`) before storing
   `manifest.voice = {mode:"user", user_ref_path:<absolute>}` (a relative path
   stored from an arbitrary CWD would resolve differently later — §11.2 B6),
   invalidates `voice_ref` (cascade). `voice auto`: sets mode auto + invalidate.
5. `invalidate --stage S [--lang L] [--cascade/--no-cascade] | --input`
   (**default `--cascade`**, matching issue 10's `cascade=True`; `--no-cascade`
   marks only the selected stage):
   maps to engine APIs (10); when target includes `synthesize` and the last synth doc
   holds a `provider_voice_id`, call the TTS provider's `cleanup_voice` (best-effort:
   failures log a warning with manual cleanup hint, never block) before marking stale
   (§5.6/issue 17). `--input` re-hashes and reports changed/unchanged.
6. All mutating commands take the project lock; all support `--project`.

## Acceptance Criteria

- [ ] Export→edit→import round trip: edit 2 texts, approve 1 → statuses correct,
      exactly 2 segments stale in synthesize (engine assertion), diff shown.
- [ ] Import rejections: wrong project_id, unknown id, empty text, bad status —
      itemized errors, file untouched.
- [ ] `--dry-run` import changes nothing (byte-compare doc).
- [ ] `voice set` with invalid file → DS-STAGE-003 variant, manifest unchanged;
      valid file → voice_ref + descendants stale.
- [ ] `invalidate --stage synthesize --lang en` triggers `cleanup_voice` exactly once
      (mock TTS asserts) and tolerates cleanup failure with warning.
- [ ] Concurrent import while a run holds the lock → exit 13.

## Validation

`uv run pytest tests/cli/test_segments.py test_voice.py test_invalidate.py`.

## Dependencies

24, 25, 10, 26 (synth.json header contract for voice cleanup), 31 (plumbing) —
matches the ISSUE_PLAN row; `cleanup_voice` interface comes from 13 (17 mocked in
tests).

## Non-goals

CSV/xlsx export formats, in-terminal editing, transcript (source-text) editing in v1
(transcript edits happen by re-running transcribe or v2 tooling).

## Design References

DESIGN.md §9, §6.2, §5.5, §5.6, §11.2 B6.
