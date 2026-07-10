# Issue 17: TTS provider — ElevenLabs

## Title

Implement ElevenLabs voice-clone TTS provider (IVC + synthesis)

## Summary

Implement `providers/tts/elevenlabs.py`: Instant Voice Cloning from the project's
reference audio, per-segment synthesis with `eleven_multilingual_v2` (default) or
`eleven_v3`, voice reuse/cleanup, rate-limit handling, and cost estimation.

## Context

The cloud voice-clone path (research doc §2; DESIGN.md §7.5). Voice objects are remote
state that must be cached and cleaned deliberately. Known unknown U-07 (IVC terms for
generated-audio redistribution) is verified here.

## Scope

In: provider + tests (respx). Out: reference-audio construction (25), consent —
enforced solely by the synthesize stage (26); this provider performs no consent
checks.

## Detailed Requirements

1. Key `ELEVENLABS_API_KEY` via `resolve_api_key`; httpx with **TLS verification
   always on** (no `verify` escape hatch in v1 — §11.2 B2); endpoints:
   `POST /v1/voices/add` (multipart, name `dubstudio-<voice_hash[:12]>`,
   description marks it tool-managed), `POST /v1/text-to-speech/{voice_id}`
   (`output_format=pcm_44100`), `DELETE /v1/voices/{voice_id}`.
   `healthcheck()`: `GET /v1/user/subscription`, 10 s timeout — 200 → `ok=true`
   with the tier in `detail`; 401/429/5xx → `ok=false` with the status (no
   exception from healthcheck).
2. Config `[tts.elevenlabs]`: `model` default `eleven_multilingual_v2` (allow
   `eleven_v3`), `stability=0.5`, `similarity_boost=0.75`, `style=0.0`.
3. `capabilities()`: `supports_cloning=True`, `watermark_builtin=False`; the
   per-model language sets and `max_chars_per_request` limits are **verified against
   the official ElevenLabs docs at implementation time** and committed as constants
   with a dated source-URL comment (tests assert the constants exist and are
   non-trivial; do not trust memory or this issue for the numbers).
4. `prepare_voice(ref)` always creates a new IVC voice (multipart with the reference
   file; name `dubstudio-<voice_hash[:12]>`; description marks it tool-managed) and
   returns `VoiceHandle{kind:"cloned", provider_voice_id, voice_hash}`. **Voice
   reuse is owned by the synthesize stage** (issue 26): when its synth doc already
   maps this `voice_hash` to a `provider_voice_id`, the stage constructs the
   `VoiceHandle` itself and never calls `prepare_voice`. Consent enforcement is also
   the stage's job (issue 26 / DESIGN §11.4) — this provider performs no consent
   checks of its own.
5. `synthesize(text, voice, language, params)`: request PCM 44.1 kHz output and wrap
   into WAV (`SynthAudio.duration_ms` measured, not trusted from API); texts longer
   than `max_chars_per_request` → `RequestTooLarge DS-PROVIDER-009`, non-retryable,
   message carrying the char count and the limit (stage guarantees segment texts
   are shorter; guard anyway).
6. `cleanup_voice(handle)`: DELETE; 404 tolerated (already gone). Called by
   `invalidate --stage synthesize` flow (issue 33 wires it) and never automatically.
7. Errors (exact mapping, via issue 13's classes and `retry_policy`):
   401 → `ProviderAuthError` DS-PROVIDER-002; 429 → `ProviderQuotaError`
   DS-PROVIDER-003 with `retry_after_s` from the header (retried by policy);
   5xx → `ProviderRemoteError` DS-PROVIDER-004 (retried); 422 (e.g. too-short
   reference) → `ProviderInvalidResponse` DS-PROVIDER-007, non-retryable, API
   message passed through the redaction filter.
8. `estimate_cost(TtsWork(chars))`: `chars × DEFAULT_PRICES["elevenlabs.char"]`
   (dated approximate constant; `[cost.tables] "elevenlabs.char"` overrides — one
   rate for all models in v1, noted in the constant's comment).
9. Privacy disclosure (§11.2 B2): provider docstring + user-docs stub state exactly
   what leaves the machine — the reference audio (voice cloning upload), every
   synthesized text with language/model/voice settings, and account/subscription
   metadata requests — and name the local alternative (`chatterbox`).
10. **U-07 verification task**: during implementation, check current ElevenLabs terms
    for IVC and generated-audio commercial use; record findings (date + link + quote)
    in `docs/research/2026-07-audio-model-landscape.md` §2, and when restrictions
    exist, surface them in all three of: the provider docstring, `docs/POLICY.md`,
    and the providers user-doc stub (`docs/guide/providers.md`, created by issue 43 —
    leave a stub note if it does not exist yet). If terms prohibit our default use
    pattern, stop and escalate in the PR (user decision required).

## Acceptance Criteria

- [ ] respx tests: IVC multipart contains the reference file and tool-managed name;
      `prepare_voice` always creates (reuse is stage-side, asserted in issue 26's
      tests); synthesis wraps PCM to valid WAV with measured duration; DELETE
      tolerated on 404.
- [ ] 401/429/422/5xx map to the exact classes/codes above; Retry-After honored;
      422 not retried.
- [ ] No key material in logs (grep test).
- [ ] Language/limit constants committed with dated source comment (test asserts
      presence and sane bounds).
- [ ] Cost estimate matches table × chars in a golden test.
- [ ] Privacy disclosure text present in docstring (+ stub) per req 9.
- [ ] U-07 findings recorded in the research doc (PR includes the diff).

## Validation

`uv run pytest tests/providers/tts/test_elevenlabs.py`; manual live smoke: clone from
30 s fixture voice, synthesize one ja + one en sentence, listen, then verify cleanup
deletes the voice (documented with subscription-safe tiny usage).

## Dependencies

13, 07 — matches the ISSUE_PLAN row. (Consent is enforced by the synthesize stage,
issue 26 — not here.)

## Non-goals

PVC, dubbing-API product usage, streaming, voice library management UI.

## Design References

DESIGN.md §5.6, §7.5, §11.2 B2, §11.4; research doc §2; ISSUE_PLAN U-07.
