# Issue 18: TTS provider — Chatterbox Multilingual (local)

## Title

Implement local voice-clone TTS provider backed by Chatterbox Multilingual

## Summary

Implement `providers/tts/chatterbox.py`: zero-shot voice cloning TTS running locally
via the `chatterbox-tts` package (MIT), with built-in PerTh watermarking surfaced as a
capability, behind the `local-tts` extra.

## Context

Default TTS (research doc §3; ADR-005): chosen specifically because outputs are
watermarked by default, ja/en are covered, and the license is MIT. Known unknown U-02
(Japanese prosody quality) is evaluated here.

## Scope

In: provider + registration + tests (library mocked) + U-02 evaluation notes.
Out: reference building (25), fit/timing (27).

## Detailed Requirements

1. Lazy import `chatterbox` inside methods; missing → `ProviderNotInstalled
   DS-PROVIDER-005` hint `uv pip install 'dubstudio[local-tts]'`.
2. Config `[tts.chatterbox]`: `model="multilingual"`, `device="auto"`
   (auto: cuda → mps → cpu), `exaggeration=0.5`, `cfg_weight=0.5` (expose the
   library's main synthesis knobs; exact names verified at implementation against the
   pinned library version and documented).
3. `capabilities()`: `supports_cloning=True`, languages = the library's supported set
   (constant, incl. `ja`, `en`), **`watermark_builtin=True`**,
   `max_chars_per_request` per library guidance (constant with source comment).
4. `prepare_voice(ref)`: validate reference WAV (mono, ≥ 10 s recommended — warn under
   10 s); return local `VoiceHandle` carrying the reference path + voice_hash (no
   remote state).
5. `synthesize(text, voice, language, params)`: call the library's generate with
   `audio_prompt_path=ref`, language code mapped via `normalize_lang`; output resampled
   to 44.1 kHz mono WAV; duration measured. Long text (> max_chars) →
   `DS-PROVIDER-009` (stage prevents this).
6. Model weights: downloaded by the library from the official
   `ResembleAI/chatterbox` HF repo on first use into the user cache;
   docstring + user-docs stub state source, size (~GB), and the §11.2 B4 posture
   (TLS, official repo, no trust_remote_code). `ProviderInfo.version` includes
   the library version + model revision when available.
7. Watermark: do **not** disable PerTh watermarking (no config to turn it off);
   `providers list` shows `watermark=yes`.
8. `healthcheck()`: import + device resolution + weights-present check (no synthesis).
9. **U-02 evaluation**: synthesize a fixed ja paragraph + en paragraph with a fixture
   reference voice; attach subjective notes + files to the PR; if ja quality is
   clearly unusable, record in research doc §3 and propose flipping the ja-docs
   default to ElevenLabs (user decision; do not change defaults unilaterally).

## Acceptance Criteria

- [ ] Provider satisfies TtsProvider Protocol; registry row shows mode=local,
      watermark=yes, extra status.
- [ ] Mocked-library tests: language mapping, reference validation (short-ref warning),
      resample-to-44.1k path, deterministic voice_hash propagation.
- [ ] Missing extra → DS-PROVIDER-005 with install hint.
- [ ] `-m live` test (skipped in CI): 1 sentence ja + en synthesized on cpu; nonzero,
      correct-samplerate WAVs.
- [ ] U-02 notes present in PR + research doc updated.

## Validation

`uv run pytest tests/providers/tts/test_chatterbox.py`; local live run for U-02.

## Dependencies

13, 07, 11.

## Non-goals

Fine-tuning, watermark *verification* tooling (v2), voice conversion, streaming.

## Design References

DESIGN.md §7.5, §11.2 B4, §11.4; ADR-005; research doc §3; ISSUE_PLAN U-02.
