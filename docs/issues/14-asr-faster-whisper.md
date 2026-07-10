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

1. Lazy import faster_whisper inside methods; `ImportError` →
   `ProviderNotInstalled DS-PROVIDER-005` hint `uv pip install 'dubstudio[local-asr]'`.
2. Config (issue 06 `[asr.faster_whisper]`): `model` (default `large-v3-turbo`;
   accept any CTranslate2 model id/path), `device` (`auto|cpu|cuda`; `auto` = cuda if
   available else cpu — MPS unsupported by CTranslate2: on darwin `auto`→cpu with a
   one-line info log), `compute_type` (`auto` → `int8` on cpu, `float16` on cuda).
3. `capabilities()`: `word_timestamps=True`, `languages="*"`.
4. `transcribe(audio, language, on_progress)`:
   - `WhisperModel(...).transcribe(str(audio), language=language or None,
     word_timestamps=True, vad_filter=True)`;
   - map segments/words to `RawTranscript` (ms ints, texts stripped; keep
     `avg_logprob`, `no_speech_prob`);
   - progress callback per segment using `info.duration`;
   - detected language + probability returned in `RawTranscript.language` /
     `.language_confidence`.
5. Model download: goes through the library's HF cache; provider records
   `model_dir` + revision into `ProviderInfo.version`; document (docstring +
   user-docs stub) that models come from official HF repos over TLS into
   `user_cache_dir` per DESIGN.md §11.2 B4 — no `trust_remote_code`, weights are
   CTranslate2 format (no pickle execution).
6. `healthcheck()`: imports lib, resolves device, loads model metadata (no inference),
   reports model resolved/downloaded state.
7. `estimate_cost` returns None (local/free).

## Acceptance Criteria

- [ ] Without the extra installed, constructing the provider raises DS-PROVIDER-005
      with the exact install hint.
- [ ] Unit tests (faster_whisper module mocked): timestamp mapping s→ms exact;
      `language="auto"` passes None to the lib; vad_filter and word_timestamps flags
      set; darwin device fallback logs info and uses cpu.
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
