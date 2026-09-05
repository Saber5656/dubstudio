# dubstudio — v1 Issue Plan

Status: **authoritative** for scope decomposition and execution order. Every issue below
has a full draft in `docs/issues/NN-short-title.md`; GitHub Issues are derived artifacts
generated from those drafts. If they disagree, fix the draft first.

## 1. v1 completion statement

**v1 is complete when issues 01–43 are all completed and validated.** At that point a
user can: install `dubstudio` from PyPI (or run `uvx dubstudio`), create a project from a
local video, run the full pipeline (transcribe → translate → clone-voice synthesize →
fit → mix → export + subtitles) with either local-first default providers or cloud
providers, review and edit translations via CLI file round-trip or the localhost web UI
and re-run only affected work, and obtain a dubbed video with AI-disclosure metadata plus
SRT/VTT subtitles — with the consent gate, secret handling, and localhost security
controls of DESIGN.md §11 enforced. Issue 44 (lip-sync) is v2 and not part of v1
completion.

## 2. Issue list in recommended execution order

| # | File | Title | Wave |
|---|---|---|---|
| 01 | `issues/01-project-scaffold.md` | Python project scaffold (uv, src layout, Typer entry, lint/type/test tooling) | 0 |
| 02 | `issues/02-ci-pipeline.md` | CI workflow: lint, type-check, tests on Linux+macOS | 0 |
| 03 | `issues/03-governance-docs.md` | Governance & policy docs (LICENSE, NOTICE, README, CONTRIBUTING, SECURITY, POLICY) | 0 |
| 04 | `issues/04-supply-chain-security.md` | Supply-chain hardening (Dependabot, CodeQL, pip-audit, pinned actions) | 0 |
| 05 | `issues/05-errors-logging.md` | Error taxonomy, exit codes, logging with secret redaction | 1 |
| 06 | `issues/06-config-system.md` | Configuration system (TOML merge, env overrides, secret policy) | 1 |
| 07 | `issues/07-media-toolbox.md` | Media toolbox: ffmpeg/ffprobe discovery, safe subprocess runner, probe models | 1 |
| 08 | `issues/08-domain-models.md` | Domain models & JSON schemas (manifest, transcript, translation, synth, fit) | 1 |
| 09 | `issues/09-project-store.md` | Project store: layout, atomic IO, artifact paths, lock, input hashing | 1 |
| 10 | `issues/10-stage-engine.md` | Stage engine: DAG, state machine, fingerprints, runner, events, cancellation | 1 |
| 11 | `issues/11-consent-gate.md` | Voice-clone consent gate module + `consent` command + policy text | 1 |
| 12 | `issues/12-doctor-command.md` | `doctor` environment diagnostics command | 2 |
| 13 | `issues/13-provider-core.md` | Provider core: protocols, capabilities, registry, errors, cost, mock providers | 2 |
| 14 | `issues/14-asr-faster-whisper.md` | ASR provider: faster-whisper (local) | 2 |
| 15 | `issues/15-asr-openai.md` | ASR provider: OpenAI transcription API | 2 |
| 16 | `issues/16-mt-openai-llm.md` | Translation provider: OpenAI-compatible LLM (length-aware dubbing translation) | 2 |
| 17 | `issues/17-tts-elevenlabs.md` | TTS provider: ElevenLabs (IVC clone + synthesis) | 2 |
| 18 | `issues/18-tts-chatterbox.md` | TTS provider: Chatterbox Multilingual (local, watermarked) | 2 |
| 19 | `issues/19-tts-openai-preset.md` | TTS provider: OpenAI preset voices (non-clone fallback) | 2 |
| 20 | `issues/20-separation-demucs.md` | Separation provider: Demucs two-stem | 2 |
| 21 | `issues/21-stage-ingest.md` | Stage: ingest (validate, probe, extract audio) | 3 |
| 22 | `issues/22-stage-separate.md` | Stage: separate (vocals/background) | 3 |
| 23 | `issues/23-stage-transcribe.md` | Stage: transcribe + segment post-processing rules | 3 |
| 24 | `issues/24-stage-translate.md` | Stage: translate (windowed, budget-aware, edit-preserving) | 3 |
| 25 | `issues/25-stage-voice-ref.md` | Stage: voice_ref (auto reference builder + user override) | 3 |
| 26 | `issues/26-stage-synthesize.md` | Stage: synthesize (per-segment TTS with cache & concurrency) | 3 |
| 27 | `issues/27-stage-fit.md` | Stage: fit (timing adaptation + auto-shorten loop + report) | 3 |
| 28 | `issues/28-stage-mix.md` | Stage: mix (timeline placement, background bed, loudness) | 3 |
| 29 | `issues/29-stage-export.md` | Stage: export (mux, tracks, AI-disclosure metadata) | 3 |
| 30 | `issues/30-stage-subtitles.md` | Stage: subtitles (SRT/VTT serializers + wrapping) | 3 |
| 31 | `issues/31-cli-init-status.md` | CLI: `init`, `status`, `config show`, `providers list` | 4 |
| 32 | `issues/32-cli-run-orchestration.md` | CLI: `run`/`plan` orchestration UX (progress, cost confirm, filters) | 4 |
| 33 | `issues/33-cli-segments-voice.md` | CLI: `segments export/import`, `voice`, `invalidate` | 4 |
| 34 | `issues/34-ui-server-foundation.md` | UI server foundation (FastAPI, token auth, CSRF, headers, static) | 5 |
| 35 | `issues/35-ui-read-api.md` | UI read API (project, segments, audio streaming, fit report) | 5 |
| 36 | `issues/36-ui-mutation-api.md` | UI mutation & job API (PATCH segment, resynthesize, run/jobs, SSE) | 5 |
| 37 | `issues/37-ui-frontend-segments.md` | UI frontend: segments review table | 5 |
| 38 | `issues/38-ui-frontend-pipeline.md` | UI frontend: pipeline panel with live progress | 5 |
| 39 | `issues/39-cli-ui-command.md` | CLI: `ui` launch command | 5 |
| 40 | `issues/40-test-fixtures-e2e.md` | Test fixtures, mock-provider golden E2E suite | 6 |
| 41 | `issues/41-live-smoke-tests.md` | Opt-in live provider smoke tests with cost guard | 6 |
| 42 | `issues/42-packaging-release.md` | Packaging & release automation (PyPI Trusted Publishing, CHANGELOG) | 6 |
| 43 | `issues/43-user-docs.md` | User documentation (quickstart EN/JA, provider guides, troubleshooting) | 6 |
| 44 | `issues/44-v2-lipsync.md` | [v2] Lip-sync stage (research + integration plan) | v2 |

## 3. Dependency table

Hard dependencies only (soft/informational noted in each issue file).

| Issue | Depends on |
|---|---|
| 01 | — |
| 02 | 01 |
| 03 | — (repo only) |
| 04 | 02 |
| 05 | 01 |
| 06 | 05 |
| 07 | 05 |
| 08 | 05 |
| 09 | 07, 08 |
| 10 | 06, 09 |
| 11 | 03, 06 |
| 12 | 06, 07, 11, 13 |
| 13 | 06 |
| 14 | 13, 07 |
| 15 | 13, 07 |
| 16 | 13 |
| 17 | 13, 07 |
| 18 | 13, 07 |
| 19 | 13, 07 |
| 20 | 13, 07 |
| 21 | 07, 09, 10 |
| 22 | 21, 20, 10 |
| 23 | 21, 22, 10, 14 (any ASR; 15 alternative) |
| 24 | 23, 16 |
| 25 | 23, 11, 07 (22 when separation enabled) |
| 26 | 24, 25, 11, one of 17/18 |
| 27 | 26, 24, 07 |
| 28 | 27 (22 for bed) |
| 29 | 28, 11, 07 (30 when embedding) |
| 30 | 23, 24 |
| 31 | 06, 07, 09, 10, 13 |
| 32 | 10, 11, 13, 31; 21–30 (behavioral) |
| 33 | 24, 25, 10, 26, 31 |
| 34 | 05, 06, 09 |
| 35 | 34, 09, 13 |
| 36 | 34, 35, 10, 33, 11 |
| 37 | 34, 35, 36 |
| 38 | 35, 36, 37 |
| 39 | 34, 31 |
| 40 | 02, 09, 10, 11, 13, 21–30, 31–33 |
| 41 | 40, 14–20 |
| 42 | 01, 02, 03 |
| 43 | 31–33, 39 (documents final UX) |
| 44 | v1 complete |

## 4. Implementation waves

| Wave | Issues | Gate to advance |
|---|---|---|
| 0 Foundation | 01–04 | CI green on empty package; governance docs merged |
| 1 Core | 05–11 | unit tests for config/store/engine pass; engine runs a no-op DAG |
| 2 Providers | 12–20 | provider contract tests pass with mocks; `doctor`/`providers list` accurate |
| 3 Pipeline | 21–30 | mock-provider pipeline produces artifacts end-to-end |
| 4 CLI | 31–33 | quickstart flow works headless with mock providers |
| 5 UI | 34–39 | UI TestClient suite green incl. auth/CSRF negative tests |
| 6 Release/QA | 40–43 | golden E2E green in CI; live smoke run once manually; 0.1.0 publishable |

Within a wave, issues are parallelizable unless the dependency table says otherwise.

## 5. Coverage: DESIGN.md sections → issues

| DESIGN.md | Issues |
|---|---|
| §1 Overview / positioning | 03, 43 |
| §2 Scope | this plan; 44 (v2 marker) |
| §3.2 Module layout | 01 (skeleton), all |
| §3.3 Extras/deps | 01, 42 |
| §4.1–4.6 Domain & storage | 08, 09 |
| §5.1–5.10 Stages | 21, 22, 23, 24, 25, 26, 27, 28, 29, 30 |
| §6 Stage engine | 10, 32 |
| §7.1–7.4 Provider core | 13 |
| §7.5 Providers | 14, 15, 16, 17, 18, 19, 20 |
| §7.6 Cost | 13, 32 |
| §8 Configuration | 06 |
| §9 CLI | 31, 32, 33, 11 (consent), 12 (doctor), 39 (ui) |
| §10 Web UI | 34, 35, 36, 37, 38, 39 |
| §11.2 B1 media boundary | 07, 21 |
| §11.2 B2 cloud boundary | 13, 15, 16, 17, 19 |
| §11.2 B3 UI boundary | 34, 36 |
| §11.2 B4 model downloads | 14, 18, 20 |
| §11.2 B5 plugins | 13 |
| §11.2 B6 project file validation | 08, 09, 33, 36 |
| §11.2 B7 supply chain | 04, 42 |
| §11.3 Subprocess policy | 07 |
| §11.4 Consent & provenance | 11, 25, 26, 29, 03 (POLICY.md) |
| §11.5 Privacy | 32 (plan badges), 43 |
| §12 Errors & logging | 05 |
| §13 Testing | 02, 40, 41 (+ per-issue Validation sections) |
| §14 Packaging | 42 |
| §15/§16 Perf & compat | 12, 43 (documented), U-04/U-05 |

Every DESIGN section is owned by at least one issue; conversely each issue names its
Design References. No v1 behavior exists only in prose outside this plan.

## 6. Product-wide validation strategy

1. **Per-issue**: each issue's Validation section is mandatory for its PR (unit tests +
   listed manual checks). CI (02) enforces lint/type/tests; coverage gate per §13.
2. **Wave gates**: table in §4 — each wave ends with an integration checkpoint that
   exercises everything beneath it (mock providers keep this free and deterministic).
3. **Security acceptance**: issues 04, 05, 06, 07, 11, 13, 34, 36 contain explicit
   security acceptance criteria (negative tests: redaction, path traversal, CSRF, Host
   check, plugin non-activation, secret-in-TOML rejection). A release is blocked if any
   security AC is unimplemented.
4. **Golden E2E** (40) is the regression backstop: full pipeline with mocks, byte-stable
   subtitle goldens, duration tolerances.
5. **Live smoke** (41): manual, opt-in, cost-capped; run before each release with real
   ja→en content; results recorded in the release checklist (42).
6. **Dogfood exit test**: before tagging 0.1.0, dub one real ~5-min ja video → en with
   local defaults and once with ElevenLabs; review via UI; publish outputs privately;
   confirm README quickstart matches reality (43).

## 7. Deferred to v2 (not blocking v1)

Lip-sync (44 — drafted now per user request); diarization/multi-speaker (ADR-006);
DeepL/dedicated MT providers; glossary/terminology; burn-in subtitles; batch projects;
watermark verification command; keyring secret storage; Windows CI; elastic timeline;
UI waveform view; ElevenLabs PVC; YouTube/urls ingestion (policy decision needed first).

## 8. Known unknowns (may create new issues during implementation)

| ID | Unknown | Trigger/next step |
|---|---|---|
| U-01 | OpenAI transcription API word-timestamp granularity/fidelity | verify during I15; may add alignment fallback issue |
| U-02 | Chatterbox Japanese prosody quality on real creator audio | evaluate during I18; may flip docs default for ja to ElevenLabs |
| U-03 | Audibility of atempo up to 1.15/beyond | tune during I27; may add rubberband option issue |
| U-04 | Perf assumptions (DESIGN §15) on 30–90 min inputs | measure during wave 3; may add chunking issues |
| U-05 | Windows support gaps (paths, locking, ffmpeg discovery) | community feedback; possible Windows CI issue |
| U-06 | PyPI name `dubstudio` availability & trademark conflicts | check before I42 executes; may force rename decision (user decision) |
| U-07 | ElevenLabs IVC terms for redistribution of generated audio | verify during I17; document in POLICY/user docs |
| U-08 | chars-per-second budget table accuracy per language | tune during I24/I27 with fixtures |
