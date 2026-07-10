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
   - SRT: `NN\nHH:MM:SS,mmm --> HH:MM:SS,mmm\ntext\n\n`, CRLF-free (LF), UTF-8 no BOM;
   - VTT: `WEBVTT` header + a `NOTE dubstudio` header line carrying the §11.4
     disclosure sentence for **target** languages (source files get no AI note);
     timestamps `HH:MM:SS.mmm`.
3. Wrapping (pure): max chars/line = `cjk_max_chars_per_line` (21) when target lang ∈
   {ja, zh, ko} else `max_chars_per_line` (42); ≤ `max_lines` (2) lines per cue;
   space-delimited langs break at spaces (greedy, no mid-word breaks; a single word
   longer than the limit stays unbroken on its own line); CJK breaks at char
   boundaries, preferring after punctuation `、。！？」`.
4. Cue splitting: text needing > max_lines lines → split into sequential cues at word
   timestamps when available (accumulate words until char capacity), else proportional
   time split by character count; minimum cue duration 700 ms enforced by borrowing
   from the neighbor split (never overlapping next cue).
5. Timing: cue times = segment (or split) times verbatim; no global offset; cues
   sorted, non-overlapping (clamp identical to issue 23 rules).
6. Outputs per §4.1: source `subtitles/<src>.srt|.vtt`; target
   `subtitles/<lang>/<lang>.srt|.vtt`. Text source: transcript `text` /
   translation `text` (whatever status — subtitles always reflect current text).

## Acceptance Criteria

- [ ] Golden-file tests: fixture transcript+translation → byte-exact SRT and VTT for
      en (42/2) and ja (21/2) including a forced cue split with word timestamps and
      one without (proportional).
- [ ] Wrapping property tests: no line over limit (except single-long-word case,
      asserted separately); CJK preferred-break after punctuation verified.
- [ ] VTT target file contains the NOTE disclosure; source file does not.
- [ ] Min-cue-duration rule holds on crafted dense words.
- [ ] Editing a translation text and re-running regenerates only that language's files
      (fingerprint test).

## Validation

`uv run pytest tests/stages/test_subtitles.py` (pure; no media marker needed).

## Dependencies

23, 24, 10.

## Non-goals

ASS/TTML formats, styling/positioning, karaoke timing, burn-in (29/v2).

## Design References

DESIGN.md §5.10, §4.1, §11.4.
