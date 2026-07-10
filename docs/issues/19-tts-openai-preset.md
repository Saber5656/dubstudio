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

1. Config `[tts.openai]`: `model="gpt-4o-mini-tts"`, `voice="alloy"` (validated against
   the documented voice list constant), optional `instructions` string (delivery
   steering supported by this model family).
2. Key via `resolve_api_key("openai")`; endpoint `POST /v1/audio/speech`,
   `response_format="wav"`; duration measured from returned audio.
3. `capabilities()`: `supports_cloning=False`, languages "*" (multilingual synthesis),
   `watermark_builtin=False`, `max_chars_per_request=4096` (constant w/ source
   comment).
4. `prepare_voice(ref)`: ignores `ref` (returns preset VoiceHandle); if a reference is
   passed, log NOTICE that it is unused by this provider.
5. Disclosure interplay: `VoiceHandle.kind="preset"` — issue 26 records it and issue 29
   writes "preset AI voice" (not "cloned") into disclosure metadata.
6. Errors/cost/healthcheck consistent with issues 15/16 patterns.

## Acceptance Criteria

- [ ] respx tests: request body golden (model/voice/instructions/format), WAV
      passthrough, duration measured; invalid `voice` fails config validation.
- [ ] `supports_cloning=False` visible via `providers list`.
- [ ] Auth/quota/5xx mapping; no key leakage.
- [ ] Cost estimate = chars × table rate.

## Validation

`uv run pytest tests/providers/tts/test_openai_tts.py`; manual live sentence check.

## Dependencies

13, 07.

## Non-goals

OpenAI sales-gated custom voices; realtime models (GPT-Live-1 family, research doc §1).

## Design References

DESIGN.md §7.2, §7.5, §11.4 (disclosure nuance); research doc §1.
