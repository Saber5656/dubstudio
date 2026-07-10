# Issue 27: Stage — fit

## Title

Implement fit stage: timing adaptation with auto-shorten loop and fit report

## Summary

Implement `stages/fit.py` per DESIGN.md §5.7: adapt each synthesized segment to its
original time slot via padding/atempo, trigger the single-shot auto-shorten
(re-translate + re-synthesize) path for overflowing draft segments, and write
`fit_report.json` + fitted WAVs.

## Context

Fit quality is the audible difference between "AI slop" and a watchable dub. The rules
are deliberately deterministic and fully reported so the UI can surface every
compromise made.

## Scope

In: stage + fit math (pure functions) + tests. Out: translation/synthesis provider
work (24/26 expose `retranslate_segment` / single-segment synth used here), mixing
(28).

## Detailed Requirements

1. `Stage` `name="fit"`, per-lang, deps `["synthesize:<lang>"]`; `config_subset` =
   `[fit]`.
2. Pure planner `plan_fit(slot_ms, gap_to_next_ms, synth_ms, cfg) -> FitDecision`
   implementing §5.7 exactly:
   - `available_ms = slot_ms + min(gap_to_next_ms × 0.8, cfg.max_bleed_ms)`;
   - ratio ≤ 1 → `keep` (pad tail to slot_ms; if synth > slot but ≤ available:
     no pad, overrun recorded);
   - 1 < ratio ≤ atempo_max → `atempo(ratio)`;
   - ratio > atempo_max → `shorten` (eligible) else `overflow`
     (atempo_max applied, `overrun_ms = synth_ms/atempo_max − available_ms`).
3. Auto-shorten path (once per segment, only when `fit.auto_shorten` and translation
   status == `draft`): call `retranslate_segment(..., char_budget × 0.8)` (issue 24) →
   single-segment synthesize via the synthesize stage's exported helper (cache-aware) →
   re-plan; if still over → overflow handling; translation doc updated with the
   shortened text (status stays `draft`), synth doc updated; result `shortened`.
4. Apply decisions with issue 07 primitives (`atempo`, pad via `silence`+`concat`);
   fitted output per segment under `fit/<lang>/segments/`; per-segment cache key =
   synth entry hash + fit-relevant config + neighbor gap (re-fit only what changed).
5. Collision detection: fitted end (slot start + fitted duration) > next segment's
   start_ms → warning event `data={overlap_ms}` (§5.7).
6. Edge rules: last segment may bleed to `media_duration − 200 ms`; segment 0 start
   preserved.
7. `fit_report.json` per issue 08 `FitEntry` for every segment (including clean `ok`
   ones), plus summary counts in the stage-completed event
   (`{ok, shortened, warn_overflow}`).

## Acceptance Criteria

- [ ] `plan_fit` table-driven tests hit every branch incl. boundary ratios (exactly
      1.0, exactly atempo_max) and bleed capping; property test: fitted duration ≤
      available_ms + 1 ms for non-overflow results.
- [ ] Auto-shorten happy path (mock MT returns shorter text): result `shortened`,
      translation doc updated, synth cache invalidated for that id only.
- [ ] `edited` segment never auto-shortened (goes straight to overflow when over).
- [ ] Atempo'd audio duration matches synth_ms/ratio ±20 ms (media test).
- [ ] Collision warning fires with correct overlap_ms on a crafted pair.
- [ ] Report contains every segment id; counts in the completion event match.

## Validation

`uv run pytest tests/stages/test_fit.py` (+ `-m media` subset for real atempo/pad).

## Dependencies

26 (24 for retranslate helper), 07, 10.

## Non-goals

Time-stretching with formant preservation (rubberband — U-03 candidate), global
retiming/elastic timeline (v2), gap redistribution beyond single-neighbor bleed.

## Design References

DESIGN.md §5.7, §4.3 (FitReport), §6.2; ISSUE_PLAN U-03, U-08.
