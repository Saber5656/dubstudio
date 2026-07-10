# Research: Audio/Speech Model Landscape (as of 2026-07-10)

Status: informative (feeds DESIGN.md §7 Provider Abstraction and provider issues)
Researched: 2026-07-10 via web search. Model availability and pricing change quickly;
re-verify before implementing each provider issue.

## Why this research exists

dubstudio needs concrete v1 picks for four provider slots: ASR (speech-to-text),
translation, voice-clone TTS, and vocal/background separation. The user also asked to
evaluate "the latest OpenAI audio models" for applicability.

## 1. OpenAI audio models (2025-12 → 2026-07)

| Model | Released | Type | Relevance to batch dubbing |
|---|---|---|---|
| `gpt-4o-transcribe`, `gpt-4o-mini-transcribe` (snapshots `-2025-12-15`) | 2025-12-15 | STT (Transcription/Realtime API) | **High** — lower WER than Whisper v2, ~90% fewer hallucinations on noise/silence. Candidate for the API ASR provider. |
| `gpt-4o-mini-tts` (snapshot `-2025-12-15`) | 2025-12-15 | TTS, instructable delivery, preset voices | **Medium** — good non-clone TTS fallback. ~35% lower WER on TTS benchmarks, better multilingual. **No self-serve voice cloning**: "custom voices" require eligibility verification through sales channels. |
| `gpt-realtime`, `gpt-realtime-mini`, `gpt-audio-mini` (snapshots `-2025-12-15`) | 2025 / 2025-12-15 | Speech-to-speech (Realtime / Chat Completions) | Low — realtime conversational primitives. |
| GPT-Realtime-2, GPT-Realtime-Translate, GPT-Realtime-Whisper | 2026-05 | Realtime family: reasoning voice, live translation (70+ langs), streaming STT | Low-Medium — streaming-first design; batch dubbing gains little. Revisit if batch endpoints appear. |
| **GPT-Live-1 / GPT-Live-1 mini** | **2026-07-08** | Full-duplex conversational voice (speak+listen simultaneously, live translation) | **Low for v1** — designed for live conversation (replacing ChatGPT Advanced Voice Mode). No published API model IDs/pricing at research time; no voice cloning announced. Not a batch dubbing primitive. |

**Conclusions for dubstudio:**

- OpenAI is a strong choice for **API ASR** (`gpt-4o-mini-transcribe` family) and for
  **translation** (GPT-5.x text models via Chat Completions).
- OpenAI offers **no generally-available arbitrary voice cloning**; the dubbing voice must
  come from ElevenLabs (API) or a local model. The provider abstraction must allow a future
  OpenAI cloning/dubbing model to slot in without core changes.
- Timestamp caveat: word-level timestamps are first-class in local Whisper implementations;
  for the OpenAI transcription API, timestamp support differs per model (`whisper-1`
  `verbose_json` vs. `gpt-4o-transcribe` granularity options). The API ASR provider issue
  must verify current capabilities during implementation and normalize to dubstudio's
  segment model.

## 2. ElevenLabs

- **Eleven v3** went GA 2026-02-02: audio tags (`[whispers]`, `[laughs]`, …), multi-speaker
  dialogue, 70+ languages. Most expressive; not realtime (fine for batch dubbing).
- `eleven_multilingual_v2` remains the stable workhorse; Flash v2.5 is the low-latency line
  (not needed for batch).
- **Instant Voice Cloning (IVC)**: sub-minute reference audio, available from the $6/mo
  Starter tier. Professional Voice Cloning (PVC) needs hours of audio (out of v1 scope).
- ElevenLabs also sells an end-to-end Dubbing API. dubstudio deliberately composes its own
  pipeline (transparency, HITL editing, provider choice) and only uses TTS + IVC.

**Conclusion:** ElevenLabs = v1 default **cloud voice-clone TTS provider** (IVC + TTS,
model `eleven_multilingual_v2` default, `eleven_v3` opt-in).

## 3. Open-weight local voice-clone TTS

| Model | License | Languages (ja/en?) | Cloning | Notes |
|---|---|---|---|---|
| **Chatterbox Multilingual v3 (Resemble AI)** | **MIT** | 23 languages incl. **Japanese**, English | Zero-shot from short reference | **Ships with PerTh perceptual watermarking embedded by default** on every output; benchmarked competitively vs. ElevenLabs. |
| Qwen3-TTS (Alibaba) | Apache-2.0 | 10 langs incl. ja/en | 3-second zero-shot | Strong; candidate second local provider. |
| CosyVoice2-0.5B | Commercial-use permitted | 9 langs incl. ja | Cross-lingual cloning | Good streaming latency (irrelevant for batch). |
| Fish Speech V1.5 | Weights non-commercial (CC-BY-NC-SA) | en/zh/ja | Yes | License blocks Apache-2.0-friendly default. |
| IndexTTS-2 | Model license **disallows commercial use** without authorization | zh/en | Yes, strong quality | Excluded from defaults for license reasons. |
| XTTS-v2 (Coqui) | CPML (non-commercial) | 16+ | Yes | Legacy pick; license unsuitable. |

**Conclusion:** v1 local voice-clone TTS = **Chatterbox Multilingual** — MIT license,
ja/en coverage, and default watermarking that directly reinforces dubstudio's misuse-
mitigation posture (DESIGN.md §11). Qwen3-TTS documented as the next candidate provider.

## 4. Open ASR (local)

- **faster-whisper** (CTranslate2 reimplementation) remains the production default in 2026:
  same accuracy as OpenAI Whisper, ~4x faster on GPU / 2x on CPU, `word_timestamps=True`
  built in (small latency overhead).
- **large-v3** = accuracy pick; **large-v3-turbo** = 809M-param decoder-reduced variant,
  ~6x faster with accuracy within 1–2% — good default for consumer hardware.
- **WhisperX** adds wav2vec2 forced alignment + pyannote diarization. Diarization is out of
  v1 scope (single-speaker assumption, ADR-006); faster-whisper's native word timestamps
  are sufficient for v1 subtitle/segment alignment.

**Conclusion:** v1 local ASR = **faster-whisper**, default model `large-v3-turbo`,
configurable; word timestamps enabled.

## 5. Vocal/background separation

- **Demucs** (`htdemucs` family, MIT, PyTorch) is the de-facto standard for two-stem
  vocals/accompaniment separation and is pip-installable. Chosen for v1 (local only —
  no API dependency worth adding).

## 6. Derived provider matrix (v1)

| Slot | Default (local-first) | Cloud alternative | Non-goal in v1 |
|---|---|---|---|
| ASR | faster-whisper `large-v3-turbo` | OpenAI `gpt-4o-mini-transcribe` | Diarization |
| Translation | — (LLM API required) | OpenAI-compatible Chat Completions (works with OpenAI, Ollama, LM Studio via `base_url`) | Dedicated MT engines (DeepL) → v2 provider |
| Voice-clone TTS | Chatterbox Multilingual (MIT, watermarked) | ElevenLabs IVC + TTS | OpenAI custom voices (sales-gated) |
| Non-clone TTS | (Chatterbox with stock voice) | OpenAI `gpt-4o-mini-tts` preset voices | — |
| Separation | Demucs `htdemucs` | — | Cloud separation APIs |

## Sources

- [Introducing next-generation audio models in the API (OpenAI)](https://openai.com/index/introducing-our-next-generation-audio-models/)
- [Updates for developers building with voice (OpenAI Developers blog)](https://developers.openai.com/blog/updates-audio-models)
- [Introducing gpt-realtime (OpenAI)](https://openai.com/index/introducing-gpt-realtime/)
- [OpenAI releases new voice models for more natural live conversations (TechCrunch, 2026-07-08)](https://techcrunch.com/2026/07/08/openai-releases-new-voice-models-for-more-natural-live-conversations/)
- [OpenAI has new voice models that reason, translate, and transcribe as you speak (9to5Mac, 2026-05-07)](https://9to5mac.com/2026/05/07/openai-has-new-voice-models-that-reason-translate-and-transcribe-as-you-speak/)
- [ElevenLabs Cheat Sheet 2026](https://www.webfuse.com/elevenlabs-cheat-sheet)
- [ElevenLabs Instant Voice Cloning quickstart](https://elevenlabs.io/docs/eleven-api/guides/how-to/voices/instant-voice-cloning)
- [ElevenLabs Review 2026: v3, Scribe & Agents (Coval)](https://www.coval.ai/blog/elevenlabs-review-2026-voice-cloning-and-synthesis-capabilities-explained)
- [Chatterbox Multilingual (Resemble AI)](https://www.resemble.ai/learn/models/chatterbox-multilingual)
- [Chatterbox Multilingual v3: TTS with embedded watermarking (Resemble AI)](https://www.resemble.ai/resources/chatterbox-multilingual-v3-tts-with-embedded-watermarking-for-25-languages)
- [ResembleAI/chatterbox on Hugging Face](https://huggingface.co/ResembleAI/chatterbox)
- [Qwen3-TTS Technical Report (arXiv)](https://arxiv.org/html/2601.15621v1)
- [Best Open-Source Text to Speech 2026 (TextToLab)](https://texttolab.com/blog/open-source-text-to-speech)
- [Best open source STT model in 2026 (Northflank)](https://northflank.com/blog/best-open-source-speech-to-text-stt-model-in-2026-benchmarks)
- [faster-whisper guide 2026 (knightli.com)](https://knightli.com/en/2026/05/01/faster-whisper-speech-to-text/)
