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

1. `Stage` `name="transcribe"`, deps `["separate"]` (audio input resolved via
   issue 22's `vocals_source` accessor); `config_subset` = `[transcribe]` +
   **the effective source language** (see rule below), never the literal `"auto"`.
2. Effective-source-language rule (exact): the effective value is
   `manifest.languages.source`. When it is `"auto"`, the stage calls ASR with
   `language=None`, and on success writes the detected code into
   `manifest.languages.source` **before** the runner computes and stores the
   completed fingerprint — so the stored fingerprint is computed with the resolved
   code and stays stable on later evaluations (no perpetual staleness, no silent
   re-detection). `Transcript.language` always holds the resolved code.
   Re-detection happens only when the user runs `invalidate --stage transcribe`
   after manually resetting `source_language = "auto"` (documented in the command
   help).
3. Post-processing pipeline (pure functions in the stage module, exact order,
   thresholds from `[transcribe]` config):
   1. **drop**: `no_speech_prob > no_speech_prob_max` or empty/whitespace text;
   2. **merge**: segment duration < `merge_min_dur_ms` AND gap to previous <
      `merge_max_gap_ms` → append text to previous (single space joiner for
      space-delimited langs, no joiner for ja/zh — decide via source language),
      extend end_ms, merge word lists;
   3. **split** (applied **recursively** until every splittable segment is ≤
      `split_max_dur_ms`): split at the word boundary nearest a sentence punctuation
      mark (`。．.!?！？`) closest to the midpoint; without punctuation, nearest word
      boundary to midpoint; without word timestamps, proportional character split at
      midpoint sentence-punctuation (fallback plain midpoint) with interpolated
      times, warning logged once per file. A segment that cannot be split (single
      word / single character) is left over-length and logged once per segment;
   4. **hallucination collapse**: ≥ 3 identical consecutive texts → keep first,
      log warning with count;
   5. **overlap normalization**: words outside their segment bounds are clamped to
      the segment; consecutive segments with `prev.end > next.start` are clipped at
      the midpoint of the overlap (debug log). If clipping would produce
      `end_ms ≤ start_ms` (one segment contained in another), the **shorter**
      segment is dropped with a warning; word lists are re-clamped after clipping;
   6. **id assignment**: `seg_0001`… in start order (after all edits above).
   The list above is the exact and complete execution order.
5. Output `Transcript` (issue 08) with provider stamp
   `{name: <configured provider name>, model: <ProviderInfo.version>}` — using
   `ProviderInfo.version` captures effective model/revision and any provider-level
   substitution (issue 15's whisper-1 fallback) in the fingerprint; ≥ 1 segment
   required else `DS-STAGE-002` ("no speech detected").
6. Language-detection confidence < 0.5 → warning event (proceeds), per §5.3.

## Acceptance Criteria

- [ ] Table-driven unit tests for each rule with hand-built RawTranscripts, including:
      ja merge without space joiner; split lands on `。` nearest midpoint; recursive
      split of a 40 s segment yields all children ≤ max; word-less split interpolates
      times proportionally; contained-segment drop case; word re-clamp after clip.
- [ ] Property test: post-processed segments are sorted, non-overlapping, ids dense
      from seg_0001, all durations ≤ split_max_dur_ms (except unsplittable
      single-word segments — allowed and logged).
- [ ] Auto language: mock ASR returns `ja` (conf 0.9) → manifest updated **and** a
      second `evaluate()` reports the stage up-to-date (fingerprint stability test);
      conf 0.3 → warning emitted.
- [ ] Empty ASR result → DS-STAGE-002.
- [ ] Fingerprint sensitivity: changing `no_speech_prob_max` marks stage stale.

## Validation

`uv run pytest tests/stages/test_transcribe.py` (mock ASR; pure rules need no media).

## Dependencies

21, 22 (`vocals_source` accessor), 10, 14 (or 15) for real runs; tests use 13's mock.
ISSUE_PLAN row matches.

## Non-goals

Diarization (ADR-006), punctuation restoration, custom vocabulary.

## Design References

DESIGN.md §5.3, §4.3 (Transcript), §6.2.
