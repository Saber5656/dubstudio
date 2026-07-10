# Issue 26: Stage — synthesize

## Title

Implement synthesize stage: per-segment cloned TTS with cache, concurrency, cost gate

## Summary

Implement `stages/synthesize.py` per DESIGN.md §5.6: prepare the provider voice once,
synthesize each translated segment with per-segment caching and bounded concurrency,
enforce the consent and non-clone opt-in rules, and record everything in `synth.json`.

## Context

The most expensive stage (API cost / GPU time); correctness of its cache keys and
partial-failure behavior defines the product's re-run economics.

## Scope

In: stage + cache/concurrency + tests (mock TTS). Out: providers (17–19), timing (27),
cost prompt UX (engine exposes estimate; issue 32 prompts).

## Detailed Requirements

1. `Stage` `name="synthesize"`, per-lang `synthesize:<lang>`, deps
   `["translate:<lang>", "voice_ref"]`; `config_subset` = `[tts.*]` +
   `[synthesize]`.
2. Guards before any provider call:
   - consent: `require_consent` (issue 11) and refresh `manifest.consent_snapshot`;
   - capability: provider `supports_cloning=False` requires config
     `tts.allow_non_cloned_voice=true`, else `ConfigError DS-CONFIG-005` explaining the
     opt-in (§7.2); when opted in, record `voice_kind="preset"` in synth doc header;
   - language: target lang ∉ provider languages → `ProviderUnsupported
     DS-PROVIDER-006` (also caught at plan time by issue 10);
   - segment length: any text > provider `max_chars_per_request` → `DS-STAGE-007`
     listing offending ids and hinting to edit/split those translations (v1 does not
     auto-split synthesis text).
3. Voice preparation: `voice_hash = sha256(reference.wav bytes)` (reference identity
   only — provider identity lives in the provider stamp / cache key, per §5.6's
   `(provider, voice_hash)` framing); when the previous `synth.json` header has the
   same voice_hash **and** the same provider stamp, reuse its `provider_voice_id`
   by constructing the `VoiceHandle` directly (issue 17 contract); else call
   `prepare_voice` and persist the mapping. Persisted field:
   `header.voice = {voice_hash, provider_voice_id|null, kind}` with
   `kind ∈ {"cloned","preset"}` taken from `VoiceHandle.kind` (issues 08/13 define
   these fields).
4. Per-segment cache key: `sha256(text_hash + voice_hash + provider name/model +
   canonical params)`. Segment skipped when key matches existing entry AND its wav
   exists (engine `SegmentCache`, issue 10). Cache hits emit `segment_completed`
   events with `data.cached=true`.
5. Execution: thread pool of `synthesize.concurrency` (default 2) — providers are
   sync; each worker checks `ctx.cancelled` before starting a segment. **Retries are
   provider-owned** (issue 13 `retry_policy` inside each provider); the stage never
   re-retries — any `ProviderError` that escapes a provider call is recorded as that
   segment's failure. Output WAVs normalized to 44.1 kHz mono via issue 07 helpers
   when the provider returns other rates.
6. Partial failure policy (§5.6): individual segment failure → record
   `{id, error_code}` in a `failures` list, continue others; at end, if failures
   non-empty → `StageError DS-STAGE-008` listing failed ids (completed WAVs + doc
   entries persist; re-run retries only failures + cache misses).
7. `synth.json` per issue 08 `SynthDoc`: header (provider stamp incl. params hash,
   voice {voice_hash, provider_voice_id|None, kind}), per-segment entries with
   measured `duration_ms`.
8. Exported single-segment helper (consumed by fit auto-shorten, issue 27, and UI
   resynthesize, issue 36):
   `synthesize_segment(ctx: StageContext, lang: str, segment_id: str) ->
   SynthSegment` — synthesizes exactly that segment through the same guards/voice/
   provider plumbing as the stage (consent, capability, length), overwrites its
   synth-doc entry and WAV atomically under the project lock, emits
   `segment_completed`, and returns the updated entry; provider errors propagate
   unchanged (no partial-failure list for the single-segment path).
8. Cost: aggregate `estimate_cost` over cache-miss texts; expose on the plan
   (issue 10 contract); the runner receives the engine-level confirmation result — this
   stage never prompts.

## Acceptance Criteria

- [ ] Consent-missing non-interactive run → exit-3 error before any provider call
      (mock records zero calls).
- [ ] Non-clone provider without opt-in → DS-CONFIG-005; with opt-in →
      `voice_kind="preset"` recorded.
- [ ] Cache: run twice → second run makes zero provider calls; edit one segment's
      translation → exactly that segment re-synthesized.
- [ ] Concurrency honored (mock TTS with barrier asserts ≤ N in flight); cancellation
      mid-run leaves completed segment files + doc entries, stage rolls back per §6.1.
- [ ] Two mock-injected failures → DS-STAGE-008 lists exactly those ids; re-run
      retries only them; a `ProviderQuotaError` escaping the provider is recorded as
      a failure without any stage-level retry (mock call-count assertion).
- [ ] Over-long text → DS-STAGE-007 with ids.
- [ ] `header.voice.kind` persisted correctly for cloned and preset runs.

## Validation

`uv run pytest tests/stages/test_synthesize.py` (mock TTS from issue 13).

## Dependencies

24, 25, 11, one of 17/18 (13 transitively via the providers) — matches the
ISSUE_PLAN row.

## Non-goals

Prosody transfer/SSML, per-segment voice switching (ADR-006), automatic text
splitting.

## Design References

DESIGN.md §5.6, §7.2, §7.6, §6.1–6.2, §11.4.
