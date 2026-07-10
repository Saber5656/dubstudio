# Issue 40: Test fixtures and golden E2E suite

## Title

Implement fixture generator and mock-provider golden end-to-end pipeline tests

## Summary

Implement `tests/fixtures/make_fixture.py` (deterministic ffmpeg-generated media, no
binaries committed) and the golden E2E suite running init → full pipeline → export with
mock providers, asserting the artifact tree, subtitle goldens, and output properties
per DESIGN.md §13.

## Context

The regression backstop for every stage and engine behavior; also the shared fixture
source earlier issues' media tests migrate to.

## Scope

In: fixture generator (+ pytest fixtures caching per-session), E2E tests, golden
files. Out: live provider tests (41), per-module unit tests (their own issues).

## Detailed Requirements

1. `make_fixture.py` (invokable as script and importable):
   - `video(path, duration_s=10, speech_spans=[(1.0,3.5),(4.0,6.0),(7.0,9.0)],
     with_bgm=True, audio_streams=1, resolution="640x360")` — testsrc2 video; "speech"
     = band-limited sawtooth bursts at the spans (distinct f0 per span), BGM = low
     sine bed; deterministic (fixed seeds/frequencies; two runs → identical streams,
     verified by stream md5);
   - variants used by earlier issues: `audio_only(path)`, `two_audio_streams(path)`,
     `long_silence(path)`;
   - registered as session-scoped pytest fixtures building into `tmp_path_factory`
     cache.
2. Mock-provider E2E (marker `media`; CI-run):
   - project init (targets `en`, source `ja`) → `run` (engine API, not subprocess) →
     assert: every §4.1 artifact exists; manifest all completed; export file probe
     (stream layout, tags incl. DUBSTUDIO tag, duration ±100 ms); mix LUFS within
     ±1 LU of target; fit report counts match mock durations (mock TTS's
     deterministic `len(text)×55 ms` makes expected fit results computable);
   - subtitle goldens: byte-exact SRT/VTT vs `tests/golden/` (mock MT output is
     deterministic);
   - **HITL round trip**: edit one segment via `segments import`, re-run → only that
     segment re-synthesized (mock TTS call count), export re-muxed, goldens updated
     copy asserted;
   - **resume**: kill after transcribe (cancel event), re-run → completes without
     re-calling ASR (mock call counts);
   - separation-disabled variant: full run with `separation.enabled=false` →
     no separate artifacts, mix has no bed, everything else completes.
3. CLI-level smoke: the same flow via `CliRunner` (`init`, `run --yes`, `status`)
   asserting exit codes and next-action line transitions.
4. Runtime budget: full module ≤ 120 s on CI (fixtures cached; ffmpeg ops on 10 s
   media are fast) — enforced with a marker note, not a hard timeout.
5. Document in `tests/README.md`: fixture philosophy (no committed binaries), how to
   regenerate goldens (`pytest --update-goldens` flag implemented here), when goldens
   may change (subtitle/wrap rule changes only).

## Acceptance Criteria

- [ ] Fixture determinism: two generations → identical audio stream md5.
- [ ] E2E green on Linux+macOS CI with all assertions above.
- [ ] Golden update flag regenerates and the diff is reviewable (test asserts flag
      exists and normal runs never write goldens).
- [ ] Mock call-count assertions prove: resume skips ASR; HITL re-synthesizes exactly
      one segment.
- [ ] Module wall time reported < 120 s in CI logs.

## Validation

`uv run pytest -m media tests/e2e/` in CI (issue 02 matrix); goldens reviewed in PR.

## Dependencies

13 (mocks), 10, 21–30, 31–33 (CLI smoke), 09.

## Non-goals

Real-model quality evaluation (U-02/U-04 are manual), performance benchmarking,
browser automation.

## Design References

DESIGN.md §13, §4.1, §5 (all), §6.
