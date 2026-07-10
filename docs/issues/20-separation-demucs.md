# Issue 20: Separation provider — Demucs

## Title

Implement Demucs two-stem vocal/background separation provider

## Summary

Implement `providers/sep/demucs.py`: local two-stem separation (vocals /
accompaniment) using demucs `htdemucs`, behind the `separate` extra, with chunked
processing for long inputs.

## Context

Background preservation is what makes dubbed videos sound professional (DESIGN.md §5.2,
§5.8). Demucs is the only v1 separation backend (research doc §5).

## Scope

In: provider + tests (library mocked; one `-m live` cpu test). Out: the separate stage
wiring (22), mixing (28).

## Detailed Requirements

1. Lazy import; missing → DS-PROVIDER-005 hint `uv pip install 'dubstudio[separate]'`.
2. Config `[separation]`: `enabled=True` (stage-level), `model="htdemucs"`,
   `device="auto"` (cuda → mps → cpu), `segment_s: int = 0` (0 = library default;
   set >0 to bound memory on long files — hint surfaced on OOM).
3. `separate(audio, out_dir, on_progress)`: run demucs `--two-stems=vocals`
   equivalent via its Python API; outputs normalized to
   `vocals.wav` / `background.wav`, 44.1 kHz, source channel count; assert output
   duration == input ±10 ms (`StageError DS-STAGE-004` on drift).
4. Invoke through the library API in-process (not subprocess) but wrap in
   try/except mapping `torch.cuda.OutOfMemoryError`/`RuntimeError(OOM)` →
   `ProviderError DS-PROVIDER-010` with hint to set `separation.segment_s = 30` or
   `device="cpu"`.
5. Progress callback from demucs' per-chunk hooks (fallback: coarse 3-step progress).
6. Model weights via official demucs download path into user cache (§11.2 B4 note in
   docstring); `ProviderInfo.version` = demucs version + model name.
7. `healthcheck()`: import, device resolve, weights presence.
8. `estimate_cost` → None.

## Acceptance Criteria

- [ ] Mocked tests: two-stem invocation args, output naming/normalization, duration
      assert, OOM mapping with hint.
- [ ] Missing extra → DS-PROVIDER-005.
- [ ] `-m live` cpu test on 10 s fixture: produces both stems, vocals+background
      RMS sum ≈ source RMS (±3 dB sanity), duration match.
- [ ] `providers list` row correct (local, extra status).

## Validation

`uv run pytest tests/providers/sep/test_demucs.py`; live test locally.

## Dependencies

13, 07.

## Non-goals

4-stem output, cloud separation, karaoke features, denoising.

## Design References

DESIGN.md §5.2, §7.5, §11.2 B4; research doc §5.
