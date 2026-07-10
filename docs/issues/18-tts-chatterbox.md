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

1. Import policy mirrors issue 14: constructor availability check
   (`find_spec("chatterbox")`) raising `ProviderNotInstalled DS-PROVIDER-005` with
   hint `uv pip install 'dubstudio[local-tts]'`; import on first use.
2. Config `[tts.chatterbox]` (schema in issue 06): `model="multilingual"`,
   `device="auto"` (auto: cuda → mps → cpu), `exaggeration=0.5`, `cfg_weight=0.5`.
3. Library contract (as of the 2026-07 research; **verify symbol names against the
   pinned `chatterbox-tts` version at implementation time — if they differ, update
   this issue file first, docs-first rule**): pin the package version in pyproject;
   `from chatterbox.mtl_tts import ChatterboxMultilingualTTS` (multilingual) /
   `from chatterbox.tts import ChatterboxTTS` (english fallback when
   `model="english"`); load via `.from_pretrained(device=...)`; synthesize via
   `.generate(text, audio_prompt_path=str(ref), language_id=<normalized lang>,
   exaggeration=..., cfg_weight=...)` returning an audio tensor at the model's
   native sample rate (`model.sr`), converted to 44.1 kHz mono WAV via issue 07
   helpers.
4. Model integrity (§11.2 B4, mirrors issue 14): weights resolve from the official
   `ResembleAI/chatterbox` HF repo **pinned by revision constant** in this module;
   safetensors files preferred where the repo provides them (reject pickle-format
   weights if both exist); first download records sha256 of each weight file into
   `<user_cache>/models.lock.json` (TOFU) and later resolves verify; mismatch →
   `ProviderError` with deliberate-cache-clear guidance; no `trust_remote_code`.
5. `capabilities()`: `supports_cloning=True`, languages = the library's supported set
   (constant, incl. `ja`, `en`), **`watermark_builtin=True`**,
   `max_chars_per_request` per library guidance (constant with source comment).
6. `prepare_voice(ref)`: validate reference WAV (mono-able, ≥ 10 s recommended — warn
   under 10 s); return `VoiceHandle{kind:"cloned", local_ref=ref.path,
   provider_voice_id=None, voice_hash}` (no remote state).
7. `synthesize(text, voice, language, params)`: per req 3; duration measured from the
   produced WAV. Long text (> max_chars) → `DS-PROVIDER-009` (stage prevents this).
8. `ProviderInfo.version` format: `chatterbox/<package_version>/<repo>@<revision>`.
   Provider docstring + user-docs stub (`docs/guide/providers.md` stub note if not
   yet created) state: weight source repo, approximate download size, cache
   location, and the §11.2 B4 posture summary.
9. Watermark: do **not** disable PerTh watermarking (no config to turn it off);
   `providers list` shows `watermark=yes`.
10. `healthcheck()`: import + device resolution + weights-present check (no synthesis).
11. **U-02 evaluation** (manual, evidence in PR): with a maintainer-recorded ~60 s
    reference voice (not committed to the repo; consent trivially the maintainer's
    own), synthesize exactly these two paragraphs and attach the WAVs + subjective
    notes: (ja) 「こんにちは。今日は動画の多言語吹き替えパイプラインについて説明します。
    音声の分離、文字起こし、翻訳、そして声のクローン合成までを一気に実行します。」
    (en) "Hi there. Today I'll walk you through the multilingual dubbing pipeline:
    separation, transcription, translation, and voice-cloned synthesis, end to end."
    If ja quality is clearly unusable, record it in research doc §3 and propose
    flipping the ja-docs default to ElevenLabs (user decision; do not change
    defaults unilaterally).

## Acceptance Criteria

- [ ] Provider satisfies TtsProvider Protocol; registry metadata shows mode=local,
      watermark_builtin=True, extra status.
- [ ] Mocked-library tests: language mapping, reference validation (short-ref warning),
      resample-to-44.1k path, deterministic voice_hash propagation, TOFU mismatch
      error, pinned-revision resolution.
- [ ] Missing extra (find_spec mocked) → DS-PROVIDER-005 with install hint at
      construction.
- [ ] Docstring/user-docs stub contains source repo, size, cache location, B4 posture
      (grep test on docstring).
- [ ] `-m live` test (skipped in CI): 1 sentence ja + en synthesized on cpu; nonzero,
      correct-samplerate WAVs.
- [ ] U-02 evaluation evidence (two paragraph WAVs + notes) attached to PR; research
      doc updated.

## Validation

`uv run pytest tests/providers/tts/test_chatterbox.py`; local live run for U-02.

## Dependencies

13, 07, 11.

## Non-goals

Fine-tuning, watermark *verification* tooling (v2), voice conversion, streaming.

## Design References

DESIGN.md §7.5, §11.2 B4, §11.4; ADR-005; research doc §3; ISSUE_PLAN U-02.
