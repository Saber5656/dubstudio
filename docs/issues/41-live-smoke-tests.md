# Issue 41: Opt-in live provider smoke tests

## Title

Implement env-gated live smoke tests with hard cost guards

## Summary

Implement the `live`-marked test suite per DESIGN.md §13: tiny real-API/real-model
runs for each provider and one micro end-to-end dub, gated by env vars, capped by cost
estimation asserts, and never executed in CI by default.

## Context

Mock tests prove logic; live smokes prove the integrations still match reality
(API shapes, model behavior). They run manually before releases (ISSUE_PLAN §6.5).

## Scope

In: live test module + gating/cost-guard helpers + release-checklist doc hook.
Out: quality evaluation (U-02 manual notes), CI wiring (explicitly excluded).

## Detailed Requirements

1. Gating: all tests marked `live` (deselected by default via issue 01 addopts);
   additionally each test skips unless `DUBSTUDIO_LIVE_TESTS=1` AND its provider's
   key/extra is present (`skip` reason names what's missing). Consent: tests set
   `DUBSTUDIO_ACCEPT_VOICE_POLICY=1` in-process and assert the NOTICE log line.
2. Cost guard helper `assert_estimate_under(estimate, usd)`: every paid test computes
   the provider estimate first and hard-fails (not skip) if > $0.10 — guard against
   accidental fixture bloat.
3. Speech fixture helper (no committed media): `tests/live/speech_fixture.py` —
   locates a TTS binary via `shutil.which("say")` (macOS) or
   `shutil.which("espeak-ng")` (Linux); invokes it with **list-form argv, no
   shell, 30 s timeout** (through `core/procs.run`), writing only under
   `tmp_path`; converts to WAV via issue 07 helpers when needed; each missing
   binary produces the precise skip reason `"no speech synthesizer (say/espeak-ng)
   on PATH"` and only the ASR-accuracy assertion is skipped (synthetic fixtures
   cover the rest).
   Tests (one per integration, ~5 s media from the issue 40 generator):
   - `openai-asr`: transcribe 5 s speech; asserts non-empty segments, monotonic times;
   - `openai` MT: translate 3 fixture segments ja→en; asserts id round-trip, budgets
     respected within 2× (soft);
   - `openai-tts`: 1 short sentence with a preset voice; asserts nonzero WAV,
     measured duration > 0, estimate guard, no key in logs;
   - `elevenlabs`: IVC from 30 s synthetic-speech reference → 1 sentence → cleanup;
     asserts audio duration > 0; cleanup verified with a raw, test-only
     `GET https://api.elevenlabs.io/v1/voices/{voice_id}` (auth header from env,
     no retries) expecting 404 after deletion (this endpoint check lives in the
     test, not the provider contract);
   - `chatterbox` + `faster-whisper` + `demucs` local: 1 sentence synth (cpu),
     tiny transcribe (`tiny` model override), 5 s separation;
   - micro-E2E: ja fixture → en dub with `faster-whisper tiny` + `openai` MT +
     `chatterbox` (all-local except MT; MT allowed against Ollama when
     `DUBSTUDIO_LIVE_MT_BASE_URL` set — fully offline live run possible);
     asserts pipeline completes and export probes clean.
4. Each test prints a one-line actual-usage summary (from provider usage data where
   returned) for the release checklist.
5. `docs/dev/release-smoke.md`: how to run (`uv run pytest -m live`), expected cost
   (< $0.50 total), what to eyeball (listen to the ElevenLabs/Chatterbox outputs),
   and a forward reference to the release checklist path
   (`docs/dev/release-checklist.md`, created by issue 42 — the reference is a path
   string only; no dependency on 42's existence at merge time).

## Acceptance Criteria

- [ ] Default `pytest` **runs** zero live tests (they are deselected by the
      issue 01 addopts marker expression); `-m live` without env → all skipped with
      precise reasons.
- [ ] With keys (manual run): all pass; total logged estimate < $0.50; ElevenLabs
      voice provably cleaned up.
- [ ] Cost-guard failure path unit-tested (fake estimate $0.20 → fail, not skip).
- [ ] Consent env NOTICE asserted; no key material in captured logs.
- [ ] release-smoke.md exists and matches the actual commands.

## Validation

Unit-test the guards in CI; full `-m live` run executed once by the implementer with
results (durations, usage lines) pasted into the PR.

## Dependencies

40, 14–20.

## Non-goals

CI execution, quality scoring, benchmark timing, provider uptime monitoring.

## Design References

DESIGN.md §13, §7.6; ISSUE_PLAN §6 (validation strategy), U-01/U-02/U-07 evidence
collection.
