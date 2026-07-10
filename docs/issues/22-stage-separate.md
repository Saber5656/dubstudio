# Issue 22: Stage — separate

## Title

Implement separate stage: vocals/background split with skip semantics

## Summary

Implement `stages/separate.py` per DESIGN.md §5.2: run the separation provider on
`source.wav` producing `vocals.wav` + `background.wav`, honoring the
`separation.enabled=false` skip rule that reroutes downstream consumers.

## Context

Separation is what preserves BGM/SE in the final dub; but it is also the heaviest
optional dependency (torch), so the skip path must be first-class.

## Scope

In: stage class + skip semantics + tests (mock separation provider). Out: demucs
provider internals (20), mix bed behavior (28 — consumes the skip flag).

## Detailed Requirements

1. `Stage` with `name="separate"`, deps `["ingest"]`;
   `input_artifacts = [artifacts/ingest/source.wav]`; `config_subset` =
   `[separation]` (the `[separation]` table per issue 06 / DESIGN §8.2 — `enabled`
   drives the skip rule); provider stamp from the configured separation provider.
2. When `separation.enabled=false`: engine marks the stage `skipped` (issue 10 handles
   the transition); this stage module exports the single vocal-source accessor used
   by transcribe (23) and voice_ref (25):
   `vocals_source(store, manifest) -> Path` — exact status mapping:
   `completed` → `artifacts/separate/vocals.wav`; `skipped` →
   `artifacts/ingest/source.wav`; **any other status → `StageError DS-STAGE-001`**
   (dependency not satisfied — never silently fall back).
3. When enabled: call `SeparationProvider.separate(source.wav, artifacts/separate/)`;
   verify both outputs exist, are PCM s16le WAV at 44.1 kHz with the same channel
   count as `source.wav`, duration == source ±10 ms (drift → `DS-STAGE-004`).
4. Progress events forwarded from the provider callback (0.0–1.0 → stage_progress).
5. Failure of the provider surfaces its `ProviderError` unchanged (engine records it).
6. Cancellation: poll `ctx.cancelled` between provider chunks where the provider
   exposes hooks; otherwise document that separation cancels only between stages
   (acceptable v1; noted in docstring).

## Acceptance Criteria

- [ ] With mock provider: both stems written, format/channel/duration validated,
      stage completes; fingerprint includes provider stamp (changing provider name →
      stale) and input hash (modifying `source.wav` → stale).
- [ ] `separation.enabled=false` → stage `skipped`; `vocals_source()` returns
      `source.wav`; flipping to true re-plans the stage (engine test already covers the
      transition; this test asserts the accessor flips).
- [ ] `vocals_source()` with separate in `pending`/`failed` raises DS-STAGE-001.
- [ ] Duration-drift stem (mock returns short file) → DS-STAGE-004.
- [ ] Provider ProviderNotInstalled propagates with its install hint.

## Validation

`uv run pytest tests/stages/test_separate.py` (mock provider; no torch in CI).

## Dependencies

21, 20 (interface; tests use mock), 10 — ISSUE_PLAN row matches.

## Non-goals

Stem caching across projects, 4-stem workflows, noise reduction.

## Design References

DESIGN.md §5.2, §5.8 (bed interplay), §6.1 (skipped).
