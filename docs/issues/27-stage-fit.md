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
2. Pure planner `plan_fit(slot_ms, bleed_room_ms, synth_ms, cfg) -> FitDecision`
   implementing §5.7 exactly. The **caller** computes `bleed_room_ms`: for non-final
   segments, `next.start_ms − end_ms`; for the final segment,
   `media_duration_ms − end_ms − 200`. `FitDecision` = frozen dataclass
   `{action: Literal["keep","atempo","shorten_candidate","overflow"],
   atempo: float, pad_ms: int, overrun_ms: int}`. Rules, with
   `available_ms = slot_ms + min(max(bleed_room_ms, 0) × 0.8, cfg.max_bleed_ms)` and
   `ratio = synth_ms / available_ms`:
   - **short audio** (ratio ≤ 1): `atempo = clamp(ratio, cfg.atempo_min, 1.0)` —
     very short synth is slowed at most to `atempo_min` (default 0.9, ≈ 1.11×
     stretch) toward the slot; when `0.98 ≤ ratio ≤ 1.0` skip the tempo filter
     entirely (imperceptible; avoids a pointless re-encode). Remainder padded with
     tail silence to slot_ms; action `keep` when no tempo applied, else `atempo`;
   - 1 < ratio ≤ `cfg.atempo_max` (default 1.15) → action `atempo` with
     `atempo = ratio`;
   - ratio > atempo_max → action `shorten_candidate` (projected at `atempo_max`;
     the planner knows nothing of translation status or auto_shorten config — see
     req 3);
   - `overrun_ms` has **one meaning everywhere**:
     `max(projected_fitted_ms − slot_ms, 0)` where `projected_fitted_ms =
     synth_ms / atempo_of_the_branch` (`atempo_max` for `shorten_candidate`) — this
     is the value persisted in `FitEntry.overrun_ms` and used for collision
     warnings;
   - the planner never truncates audio.
3. Shorten/overflow resolution (stage code, not planner): on `shorten_candidate`,
   if `fit.auto_shorten=true` (default) **and** the segment's translation status ==
   `draft` → one re-translate pass via `retranslate_segment(ctx, lang, id,
   shorten_budget)` where `shorten_budget = max(12, floor(char_budget × 0.8))`
   (issue 24) → single-segment re-synthesis via issue 26's exported
   `synthesize_segment(ctx, lang, id)` (cache-aware; overwrites the synth entry) →
   re-plan **once**; if the re-plan still yields `shorten_candidate`, or the
   segment was never eligible → overflow handling: apply `atempo_max`, allow
   overrun into the bleed room, result `warn_overflow` (fitted end colliding with
   the next segment's start → collision warning with overlap ms).
   Eligible-and-improved path records result `shortened`; translation doc updated
   with the shortened text (status stays `draft`), synth doc updated.
4. Apply decisions with issue 07 primitives only (`atempo`, pad via
   `silence`+`concat` — list-argv subprocesses, no shell, §11.3); all input synth
   paths and fitted outputs resolve through the store registry
   (`segment_art`, ids validated `^seg_\d{4}$`, everything under
   `fit/<lang>/segments/` — §11.2 B6); per-segment cache key =
   synth entry hash + fit-relevant config + neighbor bleed_room (re-fit only what
   changed).
5. Collision detection: fitted end (slot start + fitted duration) > next segment's
   start_ms → warning event `data={overlap_ms}` (§5.7).
6. Edge rules: final-segment bleed room per the planner input rule (req 2);
   segment 0 start preserved.
7. `fit_report.json` per issue 08 `FitEntry` for every segment (including clean `ok`
   ones), plus summary counts in the stage-completed event
   (`{ok, shortened, warn_overflow}`).

## Acceptance Criteria

- [ ] `plan_fit` table-driven tests hit every branch incl. boundary ratios (exactly
      1.0, exactly atempo_max, 0.98 no-op band, ratio below atempo_min → slow-down
      floored at atempo_min + padding, negative bleed_room clamped to 0) and bleed
      capping; final-segment bleed rule tested via the caller-computed input;
      property test: fitted duration ≤ available_ms + 1 ms for non-overflow results.
- [ ] Auto-shorten happy path (mock MT returns shorter text): result `shortened`,
      translation doc updated, synth cache invalidated for that id only.
- [ ] `edited` and `approved` segments are never auto-shortened (both go straight to
      overflow when over — separate test cases); shorten_budget floor test
      (char_budget 10 → budget 12).
- [ ] Atempo'd audio duration matches synth_ms/ratio ±20 ms (media test).
- [ ] Collision warning fires with correct overlap_ms on a crafted pair.
- [ ] Report contains every segment id; counts in the completion event match.

## Validation

`uv run pytest tests/stages/test_fit.py` (+ `-m media` subset for real atempo/pad).

## Dependencies

26, 24 (retranslate helper), 07 — matches the ISSUE_PLAN row (10 transitive).

## Non-goals

Time-stretching with formant preservation (rubberband — U-03 candidate), global
retiming/elastic timeline (v2), gap redistribution beyond single-neighbor bleed.

## Design References

DESIGN.md §5.7, §4.3 (FitReport), §6.2; ISSUE_PLAN U-03, U-08.
