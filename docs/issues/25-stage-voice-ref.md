# Issue 25: Stage — voice_ref

## Title

Implement voice_ref stage: automatic reference-voice builder with user override

## Summary

Implement `stages/voice_ref.py` per DESIGN.md §5.5: produce `reference.wav` +
`reference.json` either from a validated user-supplied file or by auto-selecting clean
speech spans from the vocal track, gated by consent.

## Context

Reference quality caps clone quality. The auto-builder must be conservative (prefer
fewer, cleaner seconds) and fully explainable via provenance (`reference.json`).

## Scope

In: stage + scoring heuristics + validation. Out: consent module (11), TTS providers'
use of the reference (17/18).

## Detailed Requirements

1. `Stage` `name="voice_ref"`, deps `["separate", "transcribe"]` (vocals via
   `vocals_source`, issue 22); `config_subset` = `[voice_ref]` **only** —
   `manifest.voice` is state, not config: the canonical JSON of the voice block
   (plus the user reference file's content hash in user mode) enters the fingerprint
   as an *input*, alongside the vocals and transcript hashes.
   All media operations (validation decode, slicing, concat, resample) go through
   issue 07 helpers / `procs.run` (§11.3); `voice.user_ref_path` is opened
   read-only; every write lands under `artifacts/voice_ref/` only.
2. **Consent**: call `require_consent(interactive=ctx.interactive)` (issue 11) before
   any work; store returned snapshot into `manifest.consent_snapshot`.
3. User mode (`voice.mode == "user"`): validate `voice.user_ref_path` — exists,
   decodable audio, duration 10 s–5 min (outside → `DS-STAGE-003` variants), sample
   rate ≥ 16 kHz; convert to 44.1 kHz mono `reference.wav`;
   `reference.json = {source: "user", input_sha256, built_at}`.
4. Auto mode: score each transcript segment over the vocals track:
   - hard filters: duration 3–15 s; `avg_logprob ≥ −0.5`; clip ratio < 0.1%
     (`rms_and_clipping`, issue 07); mean RMS within −30..−10 dBFS; **music
     dominance** (§5.5): when `separate` completed, the span's vocals-stem mean RMS
     must exceed the background-stem mean RMS over the same span by ≥ 10 dB
     (module constant `VOCAL_DOMINANCE_DB = 10.0`); when separation is skipped this
     filter is skipped with a one-line info log (documented limitation);
   - score (exact, all components clamped to [0, 1]):
     `logprob_score = clamp(avg_logprob + 1.0, 0, 1)`;
     `stability_score = clamp(1 − rms_std_db / 6.0, 0, 1)` where `rms_std_db` is the
     stddev of per-500 ms RMS dBFS;
     `duration_score = 1 − min(|duration_s − 8| / 8, 1)`;
     `score = 0.4·logprob_score + 0.4·stability_score + 0.2·duration_score`;
     sort **descending by score**, ties broken by earlier `start_ms`, then id;
   - select top segments until total ≥ `voice_ref.target_seconds` (60) or candidates
     exhausted; total < `voice_ref.min_seconds` (20) → `DS-STAGE-003` with hint to use
     `dubstudio voice set --ref`;
   - slice from vocals (issue 07 `slice_audio`), concat with 300 ms silence gaps →
     `reference.wav` (44.1 kHz mono);
   - `reference.json = {source: "auto", spans: [{segment_id, start_ms, end_ms,
     score}], total_seconds, built_at}`.
5. `voice set/auto/show` CLI (issue 33) mutates `manifest.voice` and invalidates this
   stage; this issue only reads manifest state.
6. Fingerprint inputs: vocals file hash, transcript hash, `[voice_ref]` config,
   `voice` block (mode + user file hash when user mode).

## Acceptance Criteria

- [ ] Consent missing + non-interactive → DS-CONSENT-001, nothing written.
- [ ] User mode: 8 s file rejected (too short); 48 kHz stereo 60 s file converted to
      44.1 kHz mono with duration preserved ±20 ms; provenance correct.
- [ ] Auto mode on synthetic vocals (fixture with 6 clean tone-speech segments of
      known RMS + 2 clipped + 2 quiet + 1 music-dominated span with a loud
      background stem): selects exactly the clean ones (music-dominated excluded),
      orders by the exact score formula (unit-tested separately with hand-computed
      values), totals ≥ target where possible; spans recorded with scores.
- [ ] Separation-skipped variant: dominance filter skipped, info log emitted.
- [ ] Under-20s scenario → DS-STAGE-003 with the voice-set hint.
- [ ] Switching mode user→auto marks stage stale (fingerprint test).

## Validation

`uv run pytest -m media tests/stages/test_voice_ref.py` (synthetic WAV fixtures;
scoring functions additionally pure-unit-tested without media marker).

## Dependencies

23, 11 (consent), 07; 22's accessor when separation is enabled — ISSUE_PLAN row
matches this list.

## Non-goals

Multi-speaker references (ADR-006), reference denoising, cross-project voice reuse.

## Design References

DESIGN.md §5.5, §11.4; ADR-005, ADR-006.
