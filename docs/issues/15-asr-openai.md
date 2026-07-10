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
3. **Timestamp rule (U-01)**: at implementation time, verify current API support and
   commit the findings (dated comment). Behavior per `timestamps` mode:
   - `auto` (default): request the finest granularity the configured model supports;
     if it cannot return at least segment-level timestamps, transparently substitute
     `whisper-1` for the timing layer, documenting the substitution in
     `ProviderInfo.version` and a NOTICE log;
   - `segment` / `word` (explicit): if the configured model cannot honor the request,
     fail with `ProviderUnsupported DS-PROVIDER-006` whose hint names the
     `timestamps="auto"` fallback — no silent substitution on explicit settings.
   `capabilities().word_timestamps` reflects the **effective** resolved
   configuration.
4. Exact HTTP contract (`POST {base_url}/audio/transcriptions`, multipart):
   - primary path: fields `file` (chunk WAV), `model`, `language` (omitted when
     auto), plus the model-appropriate timestamp params verified in req 3;
   - fallback path (`whisper-1`): `response_format=verbose_json`,
     `timestamp_granularities[]=word` and `=segment`;
   - response mapping table into `RawTranscript`: `segments[].{start,end,text,
     avg_logprob,no_speech_prob}` (s→ms rounding half-up), `words[].{word,start,end}`
     → `RawWord`; detected language = response `language` when present, else the
     request language, else `"auto"` with `language_confidence=None`.
5. Chunking: files > 20 MB or > 20 min are split via ffmpeg silence-scan
   (`silencedetect`, threshold −35 dB, min 400 ms) into ≤ 15-min chunks cut at the
   nearest silence; per-chunk offsets re-applied to timestamps; chunk boundaries never
   split inside detected speech (fallback hard cut at 15 min when no silence found,
   warning logged). **All ffmpeg invocations go through issue 07's builders +
   `procs.run`** (argv lists, timeouts, env allowlist — provider keys never reach the
   ffmpeg child); chunk temp files live under
   `user_cache_dir("dubstudio")/asr_chunks/<input sha256>/` and are deleted after a
   successful transcribe (kept on failure for debugging, path logged).
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
      min, all cuts within silence windows; reassembled absolute timestamps preserve
      original timing within ±20 ms (silence gaps preserved, no duplicated or lost
      speech coverage).
- [ ] ffmpeg child env in chunking contains no `OPENAI_API_KEY` (env-dump helper,
      mirroring issue 07's test); temp chunks cleaned after success.
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
