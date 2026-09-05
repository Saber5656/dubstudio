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
   `export.embed_subtitles`); `config_subset` = `[export]`. All mux and verification
   probes go exclusively through issue 07 helpers (`mux`, `probe`) over `procs.run`
   (argv lists, project cwd, env allowlist, media timeout — §11.3); every input and
   output path resolves through the store registry under the project root (§11.2
   B6).
2. Container/codec matrix: output container = `export.container` or input container;
   audio codec AAC 192k (mp4/mov) / Opus 128k (mkv/webm); video `-c:v copy` always.
   Mux failure (incompatible codec/container combos) → `DS-MEDIA-005` with hint
   `export.container = "mkv"` (§5.9).
3. Expected output stream layout (exact, verified by the req 7 probe):
   - stream 0: copied video (when `has_video`);
   - stream 1: dubbed audio, disposition `default`, language tag = target;
   - stream 2: original audio (only when `keep_original_audio`, stream-copied from
     the selected input audio stream), non-default, source-language tag;
   - then subtitle streams (only when `embed_subtitles=true`): **target first**
     (`subtitles/<lang>/<lang>.srt`), then source (`subtitles/<src>.srt`) — SRT
     sidecars are the embedding source (VTT stays sidecar-only); codec mov_text for
     mp4/mov, srt for mkv; webm → documented warning, no subtitle streams. A
     missing sidecar despite `embed_subtitles=true` → `StageError DS-STAGE-001`
     (subtitles stage not completed).
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

28, 11 (snapshot semantics), 07 (mux/probe builders), 30 when `embed_subtitles` —
matches the ISSUE_PLAN row.

## Non-goals

Burn-in subtitles, re-encoding profiles, thumbnails/chapters, platform upload.

## Design References

DESIGN.md §5.9, §11.4; ADR-005.
