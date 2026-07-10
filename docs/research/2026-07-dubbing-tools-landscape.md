# Research: Existing Open-Source Dubbing Tools (as of 2026-07-10)

Status: informative (feeds DESIGN.md §1 positioning and §2 scope decisions)

## Survey

| Tool | Language/Stack | Pipeline coverage | Notable traits | Gaps dubstudio targets |
|---|---|---|---|---|
| [KrillinAI](https://github.com/krillinai/KrillinAI) | Go | download → transcribe → translate → TTS dub → reformat → covers | Staged CLI usable by agents; 100+ languages; platform-optimized (YouTube/TikTok/Bilibili) | Broad scope incl. downloading (ToS surface); not provider-pluggable in a Python-native way; weak HITL review loop |
| [pyvideotrans](https://github.com/jianchang512/pyvideotrans) | Python (GUI-first) | full auto pipeline, many API/local backends | Popular; zh-focused docs/UX | GUI monolith, hard to script; limited review workflow; not designed as a library |
| [VideoLingo](https://github.com/Halfrost/VideoLingo) *(repo per search: VideoLingo)* | Python | subtitle-first: WhisperX word-level ASR, 3-step translation refinement, multiple TTS | Excellent subtitle quality focus | Dubbing (voice clone + BGM preservation) secondary; consent/safety posture absent |
| [Linly-Dubbing](https://github.com/Kedreamix/Linly-Dubbing) | Python | dubbing + digital-human lip-sync | Lip-sync integration | Research-flavored; heavyweight; license/maintenance risk |
| [open-dubbing (Softcatala)](https://github.com/softcatala/open-dubbing) | Python CLI | translate + synchronize dialogue audio | Clean CLI concept | Small language set focus; no clone-consent posture; no review UI |
| [Auto-Synced-Translated-Dubs (ThioJoe)](https://github.com/ThioJoe/Auto-Synced-Translated-Dubs) | Python | subtitle-timing-driven dub with cloud TTS | Simple, proven timing approach (speech-rate fitting to subtitle slots) | Requires pre-made accurate subtitles; cloud-only voices |

Reference roundup: [Best Open-Source AI Video Dubbing Tools in 2026](https://videodubbing.com/blog/post/best-open-source-video-dubbing-tools-2026/),
[video-dubbing GitHub topic](https://github.com/topics/video-dubbing).

## Positioning conclusions (drive DESIGN.md §1)

1. **Review-first quality loop is the gap.** Every surveyed tool is "run and pray";
   correction means re-running everything. dubstudio treats transcript/translation as
   editable artifacts with staged resume and a thin review UI.
2. **Consent + provenance as a feature.** None of the surveyed tools has a voice-clone
   consent gate, AI-disclosure metadata, or watermark-by-default posture. As a public OSS
   voice-cloning tool this is both an ethical requirement and a differentiator.
3. **Provider-pluggable, local-first.** Hybrid local/cloud per stage with a uniform
   abstraction (and safe defaults) is only partially present in pyvideotrans and is its
   weakest architectural point.
4. **Scope discipline.** No video downloading (ToS risk — KrillinAI carries it), no
   lip-sync in v1 (Linly-Dubbing shows the complexity), no GUI monolith.
5. **Timing model validated.** ThioJoe's subtitle-slot + speech-rate-fitting approach
   confirms our fit-stage design (atempo within bounds + length-aware translation) is a
   proven, low-risk baseline.
