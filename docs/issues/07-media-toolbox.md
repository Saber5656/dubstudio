# Issue 07: Media toolbox

## Title

Implement ffmpeg/ffprobe discovery, safe subprocess runner, and media probe models

## Summary

Implement `core/procs.py` (policy-enforcing subprocess helper) and `media/ffmpeg.py` +
`media/probe.py` + `media/audio.py`: binary discovery with version gate, typed ffprobe
results, and the small set of audio operations every stage reuses.

## Context

This is trust boundary B1 (DESIGN.md §11.2): all ffmpeg interaction must go through one
audited path with argv lists, timeouts, and env allowlisting.

## Scope

In: subprocess policy, discovery/version check, probe, audio helpers (extract, slice,
concat, silence, atempo, loudnorm two-pass, place-on-timeline). Out: stage semantics
(issues 21–30), separation/TTS model execution.

## Detailed Requirements

1. `core/procs.py` single entry `run(argv: list[str], *, timeout_s: float,
   cwd: Path, env_allowlist: tuple[str, ...] = ("PATH", "HOME", "TMPDIR"),
   stdin: bytes | None = None) -> CompletedProc`:
   - never `shell=True`; rejects non-list argv via type; strips env to allowlist;
   - kills process group on timeout (SIGTERM → 5 s → SIGKILL) raising
     `StageError DS-STAGE-005` with the tail (≤ 1 MB) of stderr;
   - captures stdout/stderr with 1 MB rolling tails; returns rc, tails, duration.
2. `media/ffmpeg.py`:
   - `find_ffmpeg()/find_ffprobe()`: env `DUBSTUDIO_FFMPEG`/`_FFPROBE` override, else
     `shutil.which`; missing → `EnvMissingError DS-MEDIA-000` hint "install ffmpeg ≥ 6".
   - `version()` parse; < 6 → warning log (not fatal), recorded for `doctor`.
   - Command builders returning argv lists (no string concat), each with a focused
     function: `extract_audio(src, dst, sample_rate, channels)`,
     `slice_audio(src, dst, start_ms, end_ms)`, `concat_wavs(parts, dst, gap_ms)`,
     `atempo(src, dst, ratio)` (validate 0.5 ≤ ratio ≤ 2.0),
     `silence(dst, duration_ms, sample_rate)`,
     `loudnorm_two_pass(src, dst, lufs, true_peak)` (first pass JSON stats parse),
     `mux(video_src, audio_src, dst, *, audio_codec, keep_original_audio,
     original_src, subtitle_files: dict[lang, path] | None, metadata: dict[str, str],
     default_lang)`.
   - All builders write only under the destination directory given; assert
     `dst.is_relative_to(project_root)` at call sites (helper provided).
3. `media/probe.py`: `probe(path) -> MediaInfo` via `ffprobe -print_format json
   -show_format -show_streams`; `MediaInfo` pydantic model: duration_ms, container,
   size_bytes, video: {codec, width, height, fps} | None, audio_streams:
   [{index, codec, sample_rate, channels, lang}]; raises `MediaError DS-MEDIA-001`
   with stderr tail on parse failure.
4. `media/audio.py`: `wav_duration_ms(path)` (header read, no subprocess),
   `rms_and_clipping(path, window_ms)` (stdlib `wave` + `array`; returns per-window RMS
   dbFS and clip ratio — used by voice_ref scoring), loudness helpers delegating to
   ffmpeg builders.
5. Timeout defaults: `2 × media_duration + 120 s` helper `media_timeout(duration_ms)`.

## Acceptance Criteria

- [ ] `run()` never inherits unlisted env vars (test asserts `OPENAI_API_KEY` absent in
      child env via `env` dump helper binary/`printenv`).
- [ ] Timeout kills a `sleep`-like ffmpeg invocation and raises DS-STAGE-005 with tail.
- [ ] `probe()` on the generated fixture video (issue 40 script or inline tiny fixture)
      returns correct duration ±20 ms, stream layout.
- [ ] `atempo` rejects ratio outside [0.5, 2.0].
- [ ] `extract_audio` → `wav_duration_ms` round-trip matches probe duration ±20 ms.
- [ ] Missing ffmpeg (PATH stripped) raises DS-MEDIA-000 with install hint.

## Validation

`uv run pytest -m media tests/media/` on macOS+Linux CI (ffmpeg installed by issue 02).
mypy strict clean.

## Dependencies

05. (06 not required.)

## Non-goals

GPU probing, codec transcoding policy (issue 29), waveform rendering.

## Design References

DESIGN.md §3.2, §11.2 B1, §11.3, §5 (all stages), §15.
