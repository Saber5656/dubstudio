# Issue 24: Stage — translate

## Title

Implement translate stage: windowed, budget-aware, edit-preserving translation

## Summary

Implement `stages/translate.py` per DESIGN.md §5.4: compute char budgets, drive the
translation provider in context windows, and merge results into
`translate/<lang>/translation.json` without ever overwriting human-edited segments.

## Context

This stage owns the HITL contract: `edited`/`approved` segments are sacred. It also
seeds the char budgets the fit stage's auto-shorten loop relies on.

## Scope

In: per-language stage + budget calc + merge policy + tests (mock MT). Out: provider
prompt/JSON handling (16), single-segment re-translation helper used by fit (built
here, consumed by 27).

## Detailed Requirements

1. `Stage` `name="translate"`, per-lang instances `translate:<lang>`, deps
   `["transcribe"]`; `config_subset` = `[translation.*]` + budget table + target lang.
2. Char budgets: `char_budget = round(slot_seconds × cps[target])` with built-in table
   `{en: 15, ja: 9, zh: 7, ko: 9, de: 14, fr: 15, es: 16}` (constant; config
   `[translation] cps.<lang>` overrides; unknown lang → 14 + warning once). Minimum
   budget 12 chars.
3. Document loading (§11.2 B6 — the prior `translation.json` is user-editable,
   i.e. untrusted): read via the store's `read_json`/`parse_document` path; ids must
   match `^seg_\d{4}$` and be unique (violations → `StageError DS-STAGE-006` naming
   the file); entries whose id no longer exists in the current transcript are
   dropped with a warning listing them. Then classify segments:
   - missing/`draft` → send to provider;
   - `edited`/`approved` with unchanged source → keep verbatim;
   - `edited`/`approved` whose **source_text changed** vs current transcript →
     **preserved** (never auto-overwritten, per §5.4), but `notes` is set to
     `"source text changed after edit"` and a warning event lists the ids for
     manual review;
   - `draft` whose source changed → re-translate normally;
   - `--force-retranslate` (stage option, wired with the confirm flow in issues
     32/33) demotes **everything** to draft first (destructive), after which all
     segments re-translate.
4. Window construction (the **stage** owns windowing; the provider receives exactly
   one pre-windowed `TranslateRequest` per call, issue 16): consecutive
   to-translate segments in windows ≤ the selected provider's
   `max_window_segments` config (default 20); each request includes
   `context_summary` = concatenation of the final translations of up to the 3
   preceding segments (any status) truncated to 400 chars — deterministic (no LLM
   summarization in v1; this realizes §5.4's context block).
5. Merge results by id; provider `overruns` → warning events per segment
   (`warning`, data `{chars_over}`).
6. `retranslate_segment(ctx: StageContext, lang, segment_id,
   char_budget_override) -> str`: exported helper running a single-segment window
   through the same provider/config/event plumbing as the stage (persists the
   updated doc atomically under the project lock and returns the new text). Calling
   it for a non-`draft` segment raises an internal error (programming-error guard —
   fit only calls it for `draft`, §5.7).
7. Output `TranslationDoc` sorted by id, statuses per above, `char_budget` persisted
   per segment.

## Acceptance Criteria

- [ ] Budget table test incl. override and unknown-lang warning; slot 3.6 s en →
      budget 54 (matches DESIGN §4.3 example).
- [ ] Merge policy table-driven test: draft re-translated; edited/approved preserved
      byte-identical (unchanged source); changed-source edited/approved preserved
      with the notes marker + warning; changed-source draft re-translated;
      `--force-retranslate` demotes and re-translates everything.
- [ ] Untrusted-doc tests: duplicate id / bad id format → DS-STAGE-006; obsolete id
      dropped with warning.
- [ ] Window packing: 45 draft segments, max 20 → 3 requests with correct context
      summaries (mock MT records calls).
- [ ] `retranslate_segment` sends exactly one segment with the override budget.
- [ ] Fingerprint: changing target cps or prompt version (provider stamp) → stale;
      editing translation.json by hand does NOT mark translate stale (it is this
      stage's own output) but does mark synthesize:<lang> stale (engine test hook).
- [ ] Overrun warnings emitted with ids.

## Validation

`uv run pytest tests/stages/test_translate.py` (mock MT provider).

## Dependencies

23, 16 (10 transitively via 23 — plan row lists hard deps only).

## Non-goals

Glossaries, style presets beyond the v1 prompt, cross-language reuse.

## Design References

DESIGN.md §5.4, §4.3 (TranslationDoc), §5.7 (auto-shorten contract), §6.2.
