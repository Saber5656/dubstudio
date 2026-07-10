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

In: provider + tests (respx). Out: reference-audio construction (25), consent gate
enforcement (stage 26 calls it; provider double-checks flag presence).

## Detailed Requirements

1. Key `ELEVENLABS_API_KEY` via `resolve_api_key`; endpoints: `POST /v1/voices/add`
   (multipart, name `dubstudio-<voice_hash[:12]>`, description marks it
   tool-managed), `POST /v1/text-to-speech/{voice_id}` (`output_format=pcm_44100`),
   `DELETE /v1/voices/{voice_id}`, `GET /v1/user/subscription` (healthcheck).
2. Config `[tts.elevenlabs]`: `model` default `eleven_multilingual_v2` (allow
   `eleven_v3`), `stability=0.5`, `similarity_boost=0.75`, `style=0.0`.
3. `capabilities()`: `supports_cloning=True`, languages per model (v2: 29-lang set;
   v3: "*"), `watermark_builtin=False`, `max_chars_per_request` per model constant.
4. `prepare_voice(ref)`:
   - guard: refuse to run unless the process-level consent snapshot marker is present
     in the call context (defense in depth; the stage is the primary gate);
   - reuse: voice list is not queried — reuse is driven by the synth doc's stored
     `(voice_hash → provider_voice_id)` mapping passed in via `VoiceReference`;
     create only when absent; return `VoiceHandle{provider_voice_id, voice_hash}`.
5. `synthesize(text, voice, language, params)`: request PCM 44.1 kHz output and wrap
   into WAV (`SynthAudio.duration_ms` measured, not trusted from API); texts longer
   than `max_chars_per_request` → `ProviderError DS-PROVIDER-009` (stage guarantees
   segment texts are shorter; guard anyway).
6. `cleanup_voice(handle)`: DELETE; 404 tolerated (already gone). Called by
   `invalidate --stage synthesize` flow (issue 33 wires it) and never automatically.
7. Errors: 401 auth; 429 quota with Retry-After; 422 (e.g. too-short reference) →
   `ProviderError` with the API's message passed through (redaction-safe).
8. `estimate_cost`: total chars × per-char rate for the model from the price table.
9. **U-07 verification task**: during implementation, check current ElevenLabs terms
   for IVC and generated-audio commercial use; record findings (date + link + quote)
   in `docs/research/2026-07-audio-model-landscape.md` §2 and surface any restriction
   in the provider docstring and user docs. If terms prohibit our default use pattern,
   stop and escalate in the PR (user decision required).

## Acceptance Criteria

- [ ] respx tests: IVC multipart contains the reference file and tool-managed name;
      voice reuse skips creation when mapping provided; synthesis wraps PCM to valid
      WAV with measured duration; DELETE tolerated on 404.
- [ ] 401/429/422/5xx mapping tests; Retry-After honored.
- [ ] No key material in logs (grep test).
- [ ] Cost estimate matches table × chars in a golden test.
- [ ] U-07 findings recorded in the research doc (PR includes the diff).

## Validation

`uv run pytest tests/providers/tts/test_elevenlabs.py`; manual live smoke: clone from
30 s fixture voice, synthesize one ja + one en sentence, listen, then verify cleanup
deletes the voice (documented with subscription-safe tiny usage).

## Dependencies

13, 07, 11 (consent snapshot marker contract).

## Non-goals

PVC, dubbing-API product usage, streaming, voice library management UI.

## Design References

DESIGN.md §5.6, §7.5, §11.2 B2, §11.4; research doc §2; ISSUE_PLAN U-07.
