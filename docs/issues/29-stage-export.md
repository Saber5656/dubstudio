# Issue 29: Stage — export

## Title

Implement export stage: mux dubbed audio into video with AI-disclosure metadata

## Summary

Implement `stages/export.py` per DESIGN.md §5.9: mux the original video stream (copy,
never re-encode) with the dubbed track (default) and optional original-audio and
soft-subtitle tracks, writing the §11.4 disclosure metadata.

## Context

The user-facing deliverable. Also where ADR-005's provenance promise becomes bytes in
the output file.

## Scope

In: stage + mux invocation matrix + tests. Out: mux command builder (07), subtitle
files (30 — consumed when embedding).

## Detailed Requirements

1. `Stage` `name="export"`, per-lang, deps `["mix:<lang>"]` (+ `subtitles:<lang>` when
   `export.embed_subtitles`); `config_subset` = `[export]`.
2. Container/codec matrix: output container = `export.container` or input container;
   audio codec AAC 192k (mp4/mov) / Opus 128k (mkv/webm); video `-c:v copy` always.
   Mux failure (incompatible codec/container combos) → `DS-MEDIA-005` with hint
   `export.container = "mkv"` (§5.9).
3. Tracks: dubbed audio = track 1, disposition default, language tag = target;
   `keep_original_audio` → original audio re-muxed (stream copy from input) as track
   2, non-default, source-language tag; `embed_subtitles` → source+target subtitle
   files as soft tracks (mov_text for mp4, srt codec for mkv; webm → warning that
   subtitles are skipped, sidecars only).
4. Audio-only projects (`has_video: false` in probe.json, §5.1): output
   `<basename>.<lang>.dub.m4a` (AAC) instead; no video muxing.
5. Metadata (§11.4): container-level tags —
   `comment="Audio dubbed with AI voice cloning (dubstudio v<X.Y>; consent policy
   v<N> accepted)"` (cloned) or `"…with an AI preset voice…"` when
   `voice_kind="preset"` (issue 26 header); plus `DUBSTUDIO=ai-dubbed`. Values pull
   version from `dubstudio.__version__` and policy version from
   `manifest.consent_snapshot` (missing snapshot for preset-voice runs without consent
   → policy clause omitted, tag still present).
6. Output name: `<input_basename>.<lang>.dub.<ext>` under `export/<lang>/`;
   existing file overwritten atomically (tmp + rename).
7. Verification pass: ffprobe the output — stream layout, language tags, disposition,
   metadata tags asserted; duration vs source ±100 ms (`DS-STAGE-004`).

## Acceptance Criteria

- [ ] mp4 fixture: track order/dispositions/language tags verified via ffprobe JSON;
      `comment` and `DUBSTUDIO` tags present with correct policy/version strings.
- [ ] `keep_original_audio=false` → single audio track.
- [ ] `embed_subtitles=true` on mkv embeds two subtitle tracks; on webm logs the
      documented warning and embeds none.
- [ ] Incompatible combo (crafted) → DS-MEDIA-005 with the mkv hint.
- [ ] Audio-only project → .m4a path, no video stream, tags present.
- [ ] Video stream md5 (ffmpeg `-map 0:v -c copy -f md5 -`) identical between input
      and output (proof of no re-encode).

## Validation

`uv run pytest -m media tests/stages/test_export.py`.

## Dependencies

28, 07, 11 (snapshot semantics), 30 (when embedding).

## Non-goals

Burn-in subtitles, re-encoding profiles, thumbnails/chapters, platform upload.

## Design References

DESIGN.md §5.9, §11.4; ADR-005.
