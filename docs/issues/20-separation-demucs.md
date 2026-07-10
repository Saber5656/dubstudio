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

1. Import policy mirrors issue 14: constructor availability check
   (`find_spec("demucs")`) → DS-PROVIDER-005 hint `uv pip install
   'dubstudio[separate]'`; import on first use.
2. Config `[separation]` (single table, exact keys per issue 06 — `enabled` is
   consumed by the **stage** for skip logic, the rest by this provider):
   `enabled=True`, `model="htdemucs"`, `device="auto"` (cuda → mps → cpu),
   `segment_s: int = 0` (0 = library default; >0 bounds memory — hint surfaced on
   OOM). DESIGN.md §5.2 references these same `separation.*` keys.
3. Library contract (verify symbol names against the pinned demucs version at
   implementation time; docs-first rule on drift): `from demucs.api import
   Separator`; `Separator(model=cfg.model, device=<resolved>, segment=cfg.segment_s
   or None)`; `origin, stems = separator.separate_audio_file(str(audio))`. Two-stem
   composition: `vocals = stems["vocals"]`; `background = sum(all other stems)`
   (float32 sample-accurate sum; if peak > 1.0, scale the background by 1/peak with
   a NOTICE log). Write both via issue 07 helpers → `vocals.wav` /
   `background.wav`, 44.1 kHz, source channel count; assert output duration == input
   ±10 ms (`StageError DS-STAGE-004` on drift).
4. Wrap inference in try/except mapping `torch.cuda.OutOfMemoryError` /
   `RuntimeError("out of memory")` → `ResourceExhausted DS-PROVIDER-010` with hint
   to set `separation.segment_s = 30` or `device="cpu"`.
5. Progress callback from demucs' per-chunk hooks when available (fallback: coarse
   3-step progress).
6. Model integrity (§11.2 B4): the model name maps to demucs' official remote files;
   pin the demucs package version (weights checksums are embedded in demucs's own
   remote-file manifest — document this in the docstring as the integrity mechanism,
   plus TOFU sha256 recording of the downloaded checkpoint in
   `<user_cache>/models.lock.json` for defense in depth). No `trust_remote_code`
   equivalent path exists; state it.
   `ProviderInfo.version` = `demucs/<package_version>/<model>`.
7. `healthcheck()`: import, device resolve, weights presence.
8. `estimate_cost` → None.

## Acceptance Criteria

- [ ] Mocked tests: Separator constructed with model/device/segment args; background
      = sum of non-vocal stems with peak-guard scaling; output naming/normalization;
      duration assert; OOM mapping with hint.
- [ ] Missing extra (find_spec mocked) → DS-PROVIDER-005 at construction.
- [ ] `-m live` cpu test on 10 s fixture: produces both stems, vocals+background
      RMS sum ≈ source RMS (±3 dB sanity), duration match.
- [ ] Registry metadata correct (kind=separation, mode=local, extra status) — CLI
      row rendering is issue 31's concern.

## Validation

`uv run pytest tests/providers/sep/test_demucs.py`; live test locally.

## Dependencies

13, 07.

## Non-goals

4-stem output, cloud separation, karaoke features, denoising.

## Design References

DESIGN.md §5.2, §7.5, §11.2 B4; research doc §5.
