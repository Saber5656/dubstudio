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
2. Timeline assembly: build one ffmpeg filtergraph (single invocation) using `adelay`
   per fitted segment at `start_ms` over a generated silent base of exactly
   `media_duration_ms` (from probe.json), `amix`-ing inputs with
   `normalize=0`; segments count > ffmpeg input limit (~900) → chunked assembly in
   batches of 200 with intermediate concat (deterministic, documented).
3. Bed: when `separate` completed → include `background.wav` at
   `mix.background_gain_db` (default −3 dB); when skipped → no bed (§5.2/§5.8; the
   docstring notes the output contains synthesized speech only).
4. Loudness: two-pass `loudnorm_two_pass` (issue 07) to `mix.loudness_lufs` /
   `mix.true_peak_db`; output 48 kHz stereo PCM WAV.
5. Duration guard: output duration vs `media_duration_ms` drift > 100 ms →
   `DS-STAGE-004` (bug guard; never auto-trim).
6. Progress: 3 coarse steps (assemble, bed-mix, loudnorm) with events.
7. Determinism: identical inputs → byte-identical output not required (float DSP), but
   duration and integrated LUFS (±0.5 LU) must be stable — asserted in tests.

## Acceptance Criteria

- [ ] Media test: 3 fitted tone segments at known offsets on a 30 s timeline → energy
      detected exactly at those windows (RMS scan via issue 07), silence elsewhere
      (bed disabled case).
- [ ] Bed case: background sine present across full duration at reduced gain;
      integrated LUFS within ±0.5 LU of target.
- [ ] Duration always equals source ±100 ms; crafted drift (bad fixture) →
      DS-STAGE-004.
- [ ] >200-segment chunked path exercised with generated micro-segments (fast tones).
- [ ] Stale propagation: replacing one fitted WAV re-runs mix (fingerprint test).

## Validation

`uv run pytest -m media tests/stages/test_mix.py`.

## Dependencies

27, 07, 10 (22 informs bed presence).

## Non-goals

Ducking/sidechain (v2), per-segment gain rides, multi-channel (5.1) output.

## Design References

DESIGN.md §5.8, §5.2, §15.
