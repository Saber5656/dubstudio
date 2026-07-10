# Issue 14: ASR provider — faster-whisper (local)

## Title

Implement local ASR provider backed by faster-whisper

## Summary

Implement `providers/asr/faster_whisper.py`: local Whisper transcription with word
timestamps, language auto-detection, device/compute-type selection, and model download
integrity posture, behind the `local-asr` extra.

## Context

Default ASR (research doc §4; DESIGN.md §7.5). Runs fully offline after first model
download — the local-first flagship path.

## Scope

In: provider class + registration + tests (mocked faster_whisper lib) + model cache
notes. Out: transcript post-processing (engine-side, issue 23), diarization (ADR-006).

## Detailed Requirements

1. Import policy: no top-level faster_whisper import (module stays importable without
   the extra); the **constructor performs the availability check** (importlib.
   util.find_spec) and raises `ProviderNotInstalled DS-PROVIDER-005` with hint
   `uv pip install 'dubstudio[local-asr]'`; the actual import happens on first use.
2. Config (issue 06 `[asr.faster_whisper]`):
   - `model`: (a) a known alias — `large-v3-turbo` (default), `large-v3`, `medium`,
     `small`, `tiny` — mapped to pinned `(repo_id, revision)` constants in this
     module. The repo ids are the official faster-whisper conversions on Hugging
     Face (`Systran/faster-whisper-<size>` family); **revisions must be immutable
     commit SHAs, not tags/branches**, resolved from the official repos at
     implementation time and committed with a dated comment (do not trust this
     issue or memory for the SHAs; the AC below enforces their presence and
     format); (b) an explicit `repo_id@revision` string (revision **required** for
     non-alias ids; missing → `ConfigError DS-CONFIG-001` with example); or (c) an
     existing local directory path (no download).
   - `device` (`auto|cpu|cuda`; `auto` = cuda if available else cpu — MPS unsupported
     by CTranslate2: on darwin `auto`→cpu with a one-line info log).
   - `compute_type`: `auto` (→ `int8` on cpu, `float16` on cuda) or explicit
     passthrough value from {`int8`, `int8_float16`, `float16`, `float32`}; anything
     else → `ConfigError DS-CONFIG-001` listing allowed values.
   Model integrity (§11.2 B4): downloads resolve through the pinned revision into
   `user_cache_dir("dubstudio")`; after first download, record
   `{"<repo>@<revision>": sha256(model.bin)}` in `<user_cache>/models.lock.json`
   (TOFU) and verify on every later resolve; mismatch → `ProviderError` telling the
   user to clear the cache entry deliberately. Local-path models skip TOFU.
3. `capabilities()`: `word_timestamps=True`, `languages="*"`.
4. `transcribe(audio, language, on_progress)`:
   - `WhisperModel(...).transcribe(str(audio), language=language or None,
     word_timestamps=True, vad_filter=True)`;
   - map segments/words to `RawTranscript` (ms ints, texts stripped; keep
     `avg_logprob`, `no_speech_prob`);
   - progress callback per segment: fraction `min(segment.end / info.duration, 1.0)`
     clamped monotonic (never decreasing), message `f"transcribed {mm:ss}"`; when
     `info.duration` is missing/0, emit 0.0 once at start and 1.0 at end only;
   - detected language + probability returned in `RawTranscript.language` /
     `.language_confidence`.
5. `ProviderInfo.version` format (exact): `faster-whisper/<lib_version>/<model-id>`
   where `<model-id>` is `<alias-or-repo>@<revision>` for downloaded models and
   `local:<sha256(model.bin)[:12]>` for local paths. Docstring + user-docs stub state
   the §11.2 B4 posture: official HF repos over TLS, pinned revisions, TOFU hash
   verification, no `trust_remote_code`, CTranslate2 weights (no pickle execution).
6. `healthcheck()`: imports lib, resolves device, loads model metadata (no inference),
   reports model resolved/downloaded state.
7. `estimate_cost` returns None (local/free).

## Acceptance Criteria

- [ ] Without the extra installed (find_spec mocked to None), constructing the
      provider raises DS-PROVIDER-005 with the exact install hint.
- [ ] Alias pin table exists with all five aliases, each revision matching
      `^[0-9a-f]{40}$` (commit SHA format test) and a dated source comment.
- [ ] Unit tests (faster_whisper module mocked): timestamp mapping s→ms exact;
      `language="auto"` passes None to the lib; vad_filter and word_timestamps flags
      set; darwin device fallback logs info and uses cpu; compute_type rejection for
      `int4`; non-alias model without `@revision` rejected; TOFU mismatch raises with
      the cache-clear guidance; version string format for all three model-source
      modes.
- [ ] Live-marked test (`-m live`, local model `tiny`): fixture speech WAV transcribes
      to non-empty segments with monotonic word timestamps.
- [ ] `providers list` row shows mode=local, extra status.

## Validation

`uv run pytest tests/providers/asr/test_faster_whisper.py`; optional
`-m live` run locally with `tiny` model documented in the test docstring.

## Dependencies

13, 07.

## Non-goals

WhisperX alignment, diarization, streaming, GPU benchmarking.

## Design References

DESIGN.md §7.5, §11.2 B4; research doc §4.
