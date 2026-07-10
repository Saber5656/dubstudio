# Issue 19: TTS provider — OpenAI preset voices (non-clone fallback)

## Title

Implement OpenAI preset-voice TTS provider as the non-cloning fallback

## Summary

Implement `providers/tts/openai_tts.py`: synthesis with `gpt-4o-mini-tts` preset voices
(no cloning), usable only when the user explicitly opts into a non-cloned voice.

## Context

Some users won't or can't clone (no consent, no reference quality); DESIGN.md §7.2
defines the explicit opt-in (`tts.allow_non_cloned_voice=true`). OpenAI has no
self-serve cloning (research doc §1), so this provider honestly reports
`supports_cloning=False`.

## Scope

In: provider + tests. Out: the engine-side opt-in check (issue 26 enforces §7.2).

## Detailed Requirements

1. Config `[tts.openai]`: `model="gpt-4o-mini-tts"`, `voice="alloy"`, optional
   `instructions` string (delivery steering supported by this model family).
   Voice validation: module constant
   `OPENAI_TTS_VOICES = {"alloy","ash","ballad","coral","echo","fable","onyx",
   "nova","sage","shimmer","verse"}` with a dated source comment — **verify the set
   against current OpenAI docs at implementation time** and update the constant if it
   changed; invalid voice → `ConfigError DS-CONFIG-001` listing the set.
2. Key via `resolve_api_key("openai")`; missing → `ProviderAuthError DS-PROVIDER-002`
   naming `OPENAI_API_KEY`. Endpoint `POST {base_url or
   https://api.openai.com/v1}/audio/speech`, JSON `{model, voice, input,
   instructions?, response_format:"wav"}`; duration measured from the returned audio
   bytes, never trusted from headers.
3. `capabilities()`: `supports_cloning=False`, languages "*" (multilingual synthesis),
   `watermark_builtin=False`, `max_chars_per_request=4096` (constant w/ source
   comment).
4. `prepare_voice(ref)`: ignores `ref` and returns
   `VoiceHandle{kind:"preset", provider_voice_id=None, local_ref=None, voice_hash:
   sha256("openai-tts:"+voice)}` (issue 13's contract — `kind` is a first-class
   field there); if a reference is passed, log NOTICE that it is unused.
5. Disclosure interplay: issue 26 records `voice_kind` from `VoiceHandle.kind`;
   issue 29 then writes "preset AI voice" (not "cloned") into disclosure metadata.
6. Errors/retry: 401 → DS-PROVIDER-002; 429 → DS-PROVIDER-003 (Retry-After honored
   via issue 13 `retry_policy`); 5xx → DS-PROVIDER-004 (retried). Healthcheck:
   GET `/models/{model}`, 10 s timeout. Cost: `TtsWork(chars)` ×
   `DEFAULT_PRICES["openai-tts.char"]`.
7. Privacy disclosure (§11.2 B2): docstring + user-docs stub state that translated
   segment texts, target language, selected voice, and instructions are sent to
   OpenAI over TLS; key env-only and redacted.

## Acceptance Criteria

- [ ] respx tests: request body golden (model/voice/instructions/format), WAV
      passthrough, duration measured; invalid `voice` → DS-CONFIG-001 listing the
      valid set; valid non-default voice accepted.
- [ ] `supports_cloning=False` and `kind="preset"` visible via registry metadata.
- [ ] Auth/quota/5xx mapping per req 6; no key leakage (grep test).
- [ ] Privacy disclosure text present in docstring (grep test).
- [ ] Cost estimate = chars × table rate.

## Validation

`uv run pytest tests/providers/tts/test_openai_tts.py`; manual live sentence check.

## Dependencies

13, 07.

## Non-goals

OpenAI sales-gated custom voices; realtime models (GPT-Live-1 family, research doc §1).

## Design References

DESIGN.md §7.2, §7.5, §11.4 (disclosure nuance); research doc §1.
