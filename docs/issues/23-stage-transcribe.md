# Issue 23: Stage — transcribe

## Title

Implement transcribe stage with deterministic segment post-processing

## Summary

Implement `stages/transcribe.py` per DESIGN.md §5.3: run the ASR provider on the vocal
source, then apply the five provider-independent post-processing rules, assign stable
segment IDs, and write `transcript.json`.

## Context

Segment shape decides translation windows, subtitle cues, and synthesis slots; the
post-processing rules are the quality floor for everything downstream and must be
identical regardless of ASR provider.

## Scope

In: stage + post-processing functions (pure, separately unit-tested). Out: ASR
providers (14/15), subtitle line-wrapping (30).

## Detailed Requirements

1. `Stage` `name="transcribe"`, deps `["separate"]` (engine resolves to ingest when
   separate skipped via `vocals_source`, issue 22); `config_subset` = `[transcribe]` +
   `project.source_language`.
2. Call ASR with `language = source_language` (None when "auto"); write detected
   language back to `manifest.languages.source` when it was "auto" (locked thereafter;
   re-detection only via `invalidate --stage transcribe`).
3. Post-processing pipeline (pure functions in the stage module, exact order,
   thresholds from `[transcribe]` config):
   1. **drop**: `no_speech_prob > no_speech_prob_max` or empty/whitespace text;
   2. **merge**: segment duration < `merge_min_dur_ms` AND gap to previous <
      `merge_max_gap_ms` → append text to previous (single space joiner for
      space-delimited langs, no joiner for ja/zh — decide via source language),
      extend end_ms, merge word lists;
   3. **split**: duration > `split_max_dur_ms` → split at the word boundary nearest a
      sentence punctuation mark (`。．.!?！？`) closest to the midpoint; without
      punctuation, nearest word boundary to midpoint; without word timestamps,
      proportional character split at midpoint sentence-punctuation (fallback plain
      midpoint) with interpolated times, warning logged once per file;
   4. **hallucination collapse**: ≥ 3 identical consecutive texts → keep first,
      log warning with count;
   5. **id assignment**: `seg_0001`… in start order (post all edits).
4. Words outside their segment bounds (provider artifacts) are clamped; overlapping
   consecutive segments (end > next.start) are clipped at the boundary midpoint with a
   debug log.
5. Output `Transcript` (issue 08) with provider stamp; ≥ 1 segment required else
   `DS-STAGE-002` ("no speech detected").
6. Language-detection confidence < 0.5 → warning event (proceeds), per §5.3.

## Acceptance Criteria

- [ ] Table-driven unit tests for each rule with hand-built RawTranscripts, including:
      ja merge without space joiner; split lands on `。` nearest midpoint; word-less
      split interpolates times proportionally; clamp/clip cases.
- [ ] Property test: post-processed segments are sorted, non-overlapping, ids dense
      from seg_0001, all durations ≤ split_max_dur_ms (except unsplittable
      single-word segments — allowed and logged).
- [ ] Auto language: mock ASR returns `ja` (conf 0.9) → manifest updated; conf 0.3 →
      warning emitted.
- [ ] Empty ASR result → DS-STAGE-002.
- [ ] Fingerprint sensitivity: changing `no_speech_prob_max` marks stage stale.

## Validation

`uv run pytest tests/stages/test_transcribe.py` (mock ASR; pure rules need no media).

## Dependencies

21, 10, 14 (or 15) for real runs; tests use 13's mock.

## Non-goals

Diarization (ADR-006), punctuation restoration, custom vocabulary.

## Design References

DESIGN.md §5.3, §4.3 (Transcript), §6.2.
