# Issue 15: ASR provider — OpenAI transcription API

## Title

Implement cloud ASR provider using the OpenAI transcription API

## Summary

Implement `providers/asr/openai_asr.py`: transcription via OpenAI's API
(default `gpt-4o-mini-transcribe`) with long-audio chunking at silence boundaries,
timestamp normalization, retry/backoff, and strict key handling.

## Context

Cloud alternative when local compute is weak (DESIGN.md §7.5). Known unknown U-01:
timestamp granularity for `gpt-4o-*-transcribe` vs `whisper-1` must be verified during
implementation and the fallback rule below applied.

## Scope

In: provider + chunker + tests (respx-mocked HTTP). Out: engine post-processing (23).

## Detailed Requirements

1. httpx client; base URL `https://api.openai.com/v1` (config-overridable
   `base_url` for compatible gateways); auth via `resolve_api_key("openai")`; missing
   key → `ProviderAuthError DS-PROVIDER-002` naming `OPENAI_API_KEY` (never echoing
   values).
2. Config `[asr.openai]`: `model` default `gpt-4o-mini-transcribe`; `timestamps`
   `auto|segment|word`.
3. **Timestamp rule (U-01)**: at implementation time, verify current API support.
   Required behavior: request the finest granularity the configured model supports;
   if the model cannot return at least segment-level timestamps, the provider must
   transparently use `whisper-1` with `response_format=verbose_json` +
   `timestamp_granularities=["word","segment"]` for the timing layer (config
   `timestamps="auto"`), documenting the substitution in `ProviderInfo.version` and a
   NOTICE log. Capabilities reflect reality (`word_timestamps` True only when truly
   available).
4. Chunking: files > 20 MB or > 20 min are split via ffmpeg silence-scan
   (`silencedetect`, threshold −35 dB, min 400 ms) into ≤ 15-min chunks cut at the
   nearest silence; per-chunk offsets re-applied to timestamps; chunk boundaries never
   split inside detected speech (fallback hard cut at 15 min when no silence found,
   warning logged).
5. Segment mapping to `RawTranscript` (ms ints); language passthrough/auto-detect.
6. Retry/backoff via issue 13 `retry_policy`; 401→auth, 429→quota (honor
   `Retry-After`), 5xx→remote.
7. `estimate_cost`: audio minutes × price-table rate.
8. `healthcheck()` (network-gated by doctor): GET `/models/<model>`.
9. Privacy note in docstring + user-docs stub: this provider uploads project audio to
   OpenAI (DESIGN.md §11.2 B2 disclosure duty).

## Acceptance Criteria

- [ ] respx tests: auth header set exactly once; 401/429/500 map to the right error
      classes; Retry-After honored (fake clock).
- [ ] Chunker test: 35-min synthetic WAV with silences at known points → chunks ≤ 15
      min, all cuts within silence windows, reassembled timestamps monotonic and
      gapless (±20 ms).
- [ ] verbose_json word timestamps mapped to ms correctly (fixture JSON).
- [ ] No API key material in any log/exception (grep test with fake key).
- [ ] Capabilities reflect the timestamp rule for both configurations.

## Validation

`uv run pytest tests/providers/asr/test_openai_asr.py` (all HTTP mocked);
one manual live check against the real API documented in the PR (U-01 evidence).

## Dependencies

13, 07.

## Non-goals

Realtime/GPT-Live-1 family (out of scope per research doc §1), translation-while-
transcribing modes, diarization.

## Design References

DESIGN.md §7.5, §11.2 B2; research doc §1; ISSUE_PLAN U-01.
