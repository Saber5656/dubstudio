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
3. Load prior `translation.json` when present; classify segments:
   - missing/`draft` → send to provider;
   - `edited`/`approved` → keep verbatim (unless CLI passed
     `--force-retranslate`, plumbed as stage option, which demotes everything to
     draft first — destructive, requires the confirm flow in issue 32/33);
   - segments whose **source_text changed** vs current transcript (id present but text
     hash differs, i.e. transcript was re-run) → re-translate and reset status to
     `draft` regardless of prior status, logging a warning listing affected ids.
4. Window construction: consecutive to-translate segments in windows ≤
   `max_window_segments`; each request includes `context_summary` = concatenation of
   the final translations of up to the 3 preceding segments (any status) truncated to
   400 chars — deterministic (no LLM summarization in v1 despite §5.4's "rolling
   summary" wording; this is the concrete v1 realization).
5. Merge results by id; provider `overruns` → warning events per segment
   (`warning`, data `{chars_over}`).
6. `retranslate_segment(store, lang, segment_id, char_budget_override) -> str`:
   exported helper running a single-segment window (used by fit auto-shorten, §5.7);
   respects edited/approved protection (fit only calls it for `draft`).
7. Output `TranslationDoc` sorted by id, statuses per above, `char_budget` persisted
   per segment.

## Acceptance Criteria

- [ ] Budget table test incl. override and unknown-lang warning; slot 3.6 s en →
      budget 54 (matches DESIGN §4.3 example).
- [ ] Merge policy table-driven test: draft re-translated; edited/approved preserved
      byte-identical; changed-source segment re-translated + demoted with warning.
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

23, 16, 10.

## Non-goals

Glossaries, style presets beyond the v1 prompt, cross-language reuse.

## Design References

DESIGN.md §5.4, §4.3 (TranslationDoc), §5.7 (auto-shorten contract), §6.2.
