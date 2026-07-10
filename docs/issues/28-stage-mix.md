# Issue 28: Stage — mix

## Title

Implement mix stage: timeline placement, background bed, loudness normalization

## Summary

Implement `stages/mix.py` per DESIGN.md §5.8: place fitted segments at their slot
positions on a full-length timeline, mix with the background stem when present, and
two-pass-loudnorm the result to `mix/<lang>/dubbed_audio.wav`.

## Context

Final audio assembly; the duration guard here is the last defense against cumulative
timing bugs before mux.

## Scope

In: stage + timeline assembly + tests. Out: loudnorm/ffmpeg primitives (07), export
(29).

## Detailed Requirements

1. `Stage` `name="mix"`, per-lang, deps `["fit:<lang>"]` (+ background availability
   read from `separate` status); `config_subset` = `[mix]`.
2. Timeline assembly via issue 07's `place_on_timeline` builder exclusively (which
   owns the >200-inputs batching rule: intermediate **full-duration** timelines
   combined with `amix`, never concat). Timing join (exact): for every segment id in
   the transcript, `start_ms` comes from `transcript.json` and the audio path from
   the matching `fit_report.json` entry (via `segment_art`); a transcript id with no
   fit entry, or a fit entry with no transcript id, → `StageError DS-STAGE-006`
   naming the ids (never silently skipped). All ffmpeg work goes through issue 07
   helpers/`procs.run` (argv lists, §11.3); every input/output path resolves through
   the store under the project root (§11.2 B6).
3. Bed: when `separate` completed → include `background.wav` at
   `mix.background_gain_db` (default −3 dB); when skipped → no bed (§5.2/§5.8; the
   docstring notes the output contains synthesized speech only).
4. Loudness: two-pass `loudnorm_two_pass` (issue 07) to `mix.loudness_lufs`
   (default −16 LUFS) / `mix.true_peak_db` (default −1.5 dBTP); output 48 kHz stereo
   PCM WAV.
5. Duration guard: output duration vs `media_duration_ms` drift > 100 ms →
   `DS-STAGE-004` (bug guard; never auto-trim).
6. Progress: 3 coarse steps (assemble, bed-mix, loudnorm) with events.
7. Determinism: identical inputs → byte-identical output not required (float DSP), but
   duration and integrated LUFS (±0.5 LU) must be stable — asserted in tests.

## Acceptance Criteria

- [ ] Media test (objective thresholds): 3 fitted tone segments at known offsets on a
      30 s timeline, bed disabled → per-100 ms RMS scan shows mean in-window level ≥
      −30 dBFS and out-of-window level ≤ −55 dBFS, window edges ±30 ms tolerance
      (first/last 50 ms of each window excluded from the check).
- [ ] Bed case: background sine present across full duration at reduced gain;
      integrated LUFS within ±0.5 LU of target.
- [ ] Mismatched ids (fit entry missing / orphan) → DS-STAGE-006 naming them.
- [ ] Duration always equals source ±100 ms; crafted drift (bad fixture) →
      DS-STAGE-004.
- [ ] >200-segment chunked path exercised with generated micro-segments (fast tones).
- [ ] Stale propagation: replacing one fitted WAV re-runs mix (fingerprint test).

## Validation

`uv run pytest -m media tests/stages/test_mix.py`.

## Dependencies

27 (22 for the bed) — matches the ISSUE_PLAN row; 07/10 are transitive
implementation facilities.

## Non-goals

Ducking/sidechain (v2), per-segment gain rides, multi-channel (5.1) output.

## Design References

DESIGN.md §5.8, §5.2, §15.
