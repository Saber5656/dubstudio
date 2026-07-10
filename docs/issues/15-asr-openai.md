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

1. httpx client with **TLS verification always on** (no `verify` config exists in
   v1 — §11.2 B2); base URL `https://api.openai.com/v1` (config-overridable
   `base_url` for compatible gateways — must be `https://`, except `http://` is
   allowed for `127.0.0.1`/`localhost` only, else `ConfigError DS-CONFIG-001`);
   auth via `resolve_api_key("openai")`; missing key → `ProviderAuthError
   DS-PROVIDER-002` naming `OPENAI_API_KEY` (never echoing values).
2. Config `[asr.openai]` (schema fields added by issue 06): `model` default
   `gpt-4o-mini-transcribe`; `base_url` (None → OpenAI); `timestamps`
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
     avg_logprob,no_speech_prob}` (s→ms rounding half-up); `words[].word →
     RawWord.w`, `words[].{start,end}` → `RawWord.{start_ms,end_ms}` (same half-up
     rule); detected language = response `language` when present, else the
     request language, else `"auto"` with `language_confidence=None`.
5. Chunking: inputs exceeding **either** cap — 18 MB WAV bytes or 15 min — are
   split via silence scan into chunks satisfying **both** caps (chunk duration
   limit = `min(15 min, 18 MB ÷ bytes_per_minute)`; for 16 kHz mono s16 that is
   ≈ 9.3 min). Silence detection uses a `detect_silences(src, threshold_db=-35,
   min_len_ms=400) -> list[(start_ms, end_ms)]` builder **added to
   `media/ffmpeg.py` by this issue** (ffmpeg `silencedetect` stderr parsing,
   following issue 07's conventions: argv builder + `procs.run`, timeout, parsed
   typed output). Cuts land at the nearest silence; never inside detected speech
   (fallback hard cut at the duration cap when no silence found, warning logged);
   per-chunk offsets re-applied to timestamps. Provider keys never reach the ffmpeg
   child (env allowlist); chunk temp files live under
   `user_cache_dir("dubstudio")/asr_chunks/<input sha256>/` and are deleted after a
   successful transcribe (kept on failure for debugging, path logged).
   Progress: `on_progress(processed_duration / total_duration, "chunk i/N")` after
   each chunk, monotonic, 1.0 at completion.
6. Retry/backoff via issue 13 `retry_policy`; 401→auth, 429→quota (honor
   `Retry-After`), 5xx→remote.
7. `estimate_cost`: audio minutes × price-table rate.
8. `healthcheck()` (network-gated by doctor): GET `/models/<model>`.
9. Privacy note in docstring + user-docs stub: this provider uploads project audio to
   OpenAI (DESIGN.md §11.2 B2 disclosure duty).

## Acceptance Criteria

- [ ] respx tests: auth header set exactly once; 401/429/500 map to the right error
      classes; Retry-After honored (fake clock).
- [ ] Chunker test: 35-min synthetic WAV with silences at known points → every chunk
      satisfies both caps (≤ 18 MB asserted on file size, duration ≤ the computed
      cap), all cuts within silence windows; reassembled absolute timestamps
      preserve original timing within ±20 ms (silence gaps preserved, no duplicated
      or lost speech coverage); progress calls monotonic ending at 1.0.
- [ ] `base_url` scheme validation: `http://example.com` rejected;
      `http://127.0.0.1:8080/v1` accepted.
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
