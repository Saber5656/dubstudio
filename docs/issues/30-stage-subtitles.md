# Issue 30: Stage — subtitles

## Title

Implement subtitles stage: SRT/VTT serializers with language-aware line wrapping

## Summary

Implement `stages/subtitles.py` per DESIGN.md §5.10: generate source- and
target-language SRT and VTT files from transcript/translations with deterministic
wrapping and cue-splitting rules, using in-repo serializers (no third-party deps).

## Context

Subtitles are a top-value, low-cost output (user decision in requirements interview)
and must be byte-deterministic for golden tests.

## Scope

In: per-lang stage + source-lang emission + serializers + wrapping (pure functions).
Out: embedding into containers (29).

## Detailed Requirements

1. `Stage` `name="subtitles"`, per-lang instances deps `["transcribe",
   "translate:<lang>"]`; source-language files are (re)written by whichever subtitles
   stage runs first (idempotent, identical bytes); `config_subset` = `[subtitles]`.
2. Serializers (`stages/subtitles.py` or `media/subs.py`):
   - SRT: `NN\nHH:MM:SS,mmm --> HH:MM:SS,mmm\ntext\n\n`, CRLF-free (LF), UTF-8 no
     BOM. **SRT carries no disclosure**: the format has no comment facility and a
     fake on-screen cue would harm viewers — the exemption is deliberate and
     documented in the module docstring; disclosure rides on VTT NOTE + container
     metadata (issue 29);
   - VTT: line 1 `WEBVTT`, blank line, then for **target-language files only** the
     byte-exact block `NOTE dubstudio: AI-generated translation used for dubbing`
     followed by a blank line (source files get no NOTE); timestamps
     `HH:MM:SS.mmm`.
3. Wrapping (pure): the wrapping language is **the language of the file being
   emitted** (source files → `transcript.language`; target files →
   `target_language`); max chars/line = `cjk_max_chars_per_line` (21) when that
   language ∈ {ja, zh, ko} else `max_chars_per_line` (42); ≤ `max_lines` (2) lines
   per cue; space-delimited langs break at spaces (greedy, no mid-word breaks; a
   single word longer than the limit stays unbroken on its own line); CJK breaks at
   char boundaries, preferring after punctuation `、。！？」`.
4. Cue splitting for text needing > max_lines lines:
   - **source-language files**: split at word timestamps when the transcript has
     them (accumulate words until char capacity), else proportional
     character-count time split;
   - **target-language files**: always proportional character-count time split —
     translations have no word timings, and source word timings must never be
     applied to target text.
   Minimum cue duration 700 ms, enforced deterministically after splitting by a
   left-to-right pass: a cue shorter than 700 ms extends its end into the following
   cue's start while that cue stays ≥ 700 ms; the final cue extends backward into
   its predecessor instead; when the segment is too short for all cues
   (`segment_duration < cue_count × 700 ms`), merge the last two cues repeatedly
   until it fits or one cue remains (a single cue may be shorter than 700 ms when
   the segment itself is — allowed, debug log). Cues never overlap.
5. Timing: cue times = segment (or split) times verbatim; no global offset; cues
   sorted, non-overlapping (clamp identical to issue 23 rules).
6. Outputs per §4.1: source `subtitles/<src>.srt|.vtt`; target
   `subtitles/<lang>/<lang>.srt|.vtt`. Text source: transcript `text` /
   translation `text` (whatever status — subtitles always reflect current text).

## Acceptance Criteria

- [ ] Golden-file tests: fixture transcript+translation → byte-exact SRT and VTT for
      en (42/2) and ja (21/2) including a source-file word-timed cue split and a
      target-file proportional split.
- [ ] Wrapping property tests: no line over limit (except single-long-word case,
      asserted separately); CJK preferred-break after punctuation verified; source
      file wraps by source language, target by target language (ja→en project
      asserts both directions).
- [ ] VTT target file contains the byte-exact NOTE block; source VTT and all SRT
      files do not (SRT exemption documented in docstring — grep test).
- [ ] Min-cue-duration passes: borrowing pass, final-cue backward case, and the
      merge-until-fits fallback on a crafted too-short segment.
- [ ] Editing a translation text and re-running regenerates only that language's files
      (fingerprint test).

## Validation

`uv run pytest tests/stages/test_subtitles.py` (pure; no media marker needed).

## Dependencies

23, 24 — matches the ISSUE_PLAN row (10 transitive).

## Non-goals

ASS/TTML formats, styling/positioning, karaoke timing, burn-in (29/v2).

## Design References

DESIGN.md §5.10, §4.1, §11.4.
