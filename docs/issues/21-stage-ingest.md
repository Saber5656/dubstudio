# Issue 21: Stage — ingest

## Title

Implement ingest stage: input verification, probe, audio extraction

## Summary

Implement `stages/ingest.py` per DESIGN.md §5.1: verify the referenced input file,
enforce limits, write `probe.json`, and extract the working audio track to
`artifacts/ingest/source.wav`.

## Context

First stage of the DAG; every downstream artifact hangs off its outputs. Also the
enforcement point for the media trust boundary limits (§11.2 B1).

## Scope

In: the stage class registered into the engine graph + tests. Out: media toolbox
primitives (07), manifest/store mechanics (09/10).

## Detailed Requirements

1. Implement the `Stage` interface (issue 10) with `name="ingest"`, no deps.
2. `input_artifacts` = the input video file (its content hash drives staleness);
   `config_subset` = `[limits]` + `project.audio_stream`.
3. Behavior:
   1. `verify_input(manifest)` (issue 09): hash mismatch → `MediaError DS-MEDIA-002`
      with hint `dubstudio invalidate --input`.
   2. `probe()` (issue 07) → write `artifacts/ingest/probe.json` with the exact
      shape `{schema_version: 1, has_video: bool, media: <MediaInfo model dump>}`
      (canonical JSON rules of §4; issue 29 consumes `has_video`).
   3. Validate: at least one audio stream (`DS-MEDIA-003`); duration ≤
      `limits.max_duration_min` (`DS-MEDIA-004`); size ≤ `limits.max_input_gb`
      (`DS-MEDIA-004`).
   4. Select audio stream: `project.audio_stream` is the **zero-based ordinal within
      `MediaInfo.audio_streams`** (container order), not the ffprobe stream index;
      invalid ordinal → `DS-MEDIA-003` listing available streams as
      `ordinal (ffprobe index #N, codec, lang)`. Unset → the stream with
      `is_default=true`, else ordinal 0.
   5. Extract to `source.wav`: PCM s16le, 44.1 kHz, channels = min(source channels, 2)
      via `extract_audio` (07).
   6. Emit progress events (extraction is near-instant relative to later stages; a
      start/end pair is sufficient).
4. Audio-only inputs (no video stream) are valid; a `has_video: false` flag lands in
   probe.json, and export/mux later degrades per §5.9 (issue 29 reads this flag).
5. Idempotency: outputs written via temp+rename; partial extraction never leaves a
   truncated `source.wav` behind.

## Acceptance Criteria

- [ ] Fixture video → `probe.json` (schema_version/has_video/media keys) +
      `source.wav` created; `wav_duration_ms(source.wav)` equals probe duration
      ±20 ms.
- [ ] Input with no audio stream fails DS-MEDIA-003; over-duration input fails
      DS-MEDIA-004 (fixture generated at limit+1 min with `limits.max_duration_min=1`
      override for the test); over-size input fails DS-MEDIA-004 (probe size
      monkeypatched above `limits.max_input_gb` — no giant fixture).
- [ ] Modified input file (1 byte) → DS-MEDIA-002 with the invalidate hint.
- [ ] `--audio-stream 1` on a two-audio-stream fixture selects stream 1 (probe of the
      extracted WAV channel content differs from stream 0 — use different tones per
      stream in the fixture).
- [ ] Audio-only mp3 input completes with `has_video: false`.
- [ ] Stage fingerprint changes when `limits` config changes; re-run is no-op when
      nothing changed.

## Validation

`uv run pytest -m media tests/stages/test_ingest.py` (uses the issue 40 fixture
generator; if implemented before 40, inline a minimal ffmpeg-generated fixture in the
test and migrate to the shared generator later).

## Dependencies

07, 09, 10.

## Non-goals

Downloading, transcoding video, multi-input projects, subtitle-stream extraction.

## Design References

DESIGN.md §5.1, §5.9 (audio-only degradation), §11.2 B1, §4.1.
