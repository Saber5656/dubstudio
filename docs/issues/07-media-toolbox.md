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
   stdin: bytes | None = None, check: bool = True,
   cancel_event: threading.Event | None = None) -> CompletedProc`:
   - `cancel_event` (when provided) is polled every 200 ms; once set, the child gets
     the same SIGTERM → 5 s → SIGKILL treatment as a timeout and `run` raises
     `CancelledError` (exit 10) — this is the engine's cancellation hook for stages
     blocked in ffmpeg/model subprocesses (issue 10);
   - `CompletedProc` frozen dataclass: `argv: list[str]`, `returncode: int`,
     `stdout_tail: str`, `stderr_tail: str` (1 MB rolling tails, lossy-decoded UTF-8),
     `duration_s: float`;
   - never `shell=True`; rejects non-list argv via type; strips env to allowlist;
   - non-zero exit with `check=True` raises internal `CalledProcError` (carries the
     `CompletedProc`) which **call sites catch and map to domain errors**; procs.py
     itself raises no `DS-MEDIA-*`; `check=False` returns the `CompletedProc` as-is;
   - kills process group on timeout (SIGTERM → 5 s → SIGKILL) raising
     `StageError DS-STAGE-005` with the stderr tail.
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
     `place_on_timeline(segments: list[tuple[int, Path]], dst, *, duration_ms,
     sample_rate=48000, channels=2)` — one ffmpeg invocation (adelay + amix
     `normalize=0`) placing each `(start_ms, wav)` on a silent base and padding/
     trimming to exactly `duration_ms` (overlaps sum; collision policy is the mix
     stage's concern). When segment count exceeds one invocation's limits, batch in
     groups of 200: each batch produces an intermediate **full-duration** timeline
     WAV, and intermediates are combined with `amix` (never concatenated — concat
     would append audio and destroy timing),
     `mux(video_src, audio_src, dst, *, audio_codec, keep_original_audio,
     original_src, subtitle_files: dict[lang, path] | None, metadata: dict[str, str],
     default_lang)`.
   - Path safety: helper `ensure_under(root: Path, *paths: Path)` resolves each path
     and requires `is_relative_to(root)`, raising `ConfigError DS-CONFIG-002`
     otherwise. **Every builder calls it on dst and every temp/list/stats file before
     launching the subprocess**; temp files (concat lists, loudnorm stats) are created
     next to `dst` (same directory), never in system temp dirs.
3. `media/probe.py`: `probe(path) -> MediaInfo` via `ffprobe -print_format json
   -show_format -show_streams`; `MediaInfo` pydantic model: duration_ms, container,
   size_bytes, video: {codec, width, height, fps} | None, audio_streams:
   [{index (ffprobe stream index), codec, sample_rate, channels, lang,
   is_default (from disposition.default)}] in container order; raises
   `MediaError DS-MEDIA-001` with stderr tail on parse failure.
4. `media/audio.py`: `wav_duration_ms(path)` (header read, no subprocess);
   `rms_and_clipping(path, window_ms) -> list[WindowStats]` (stdlib `wave` + `array`,
   s16 input only) where `WindowStats = {start_ms, end_ms, rms_dbfs, clip_ratio}`:
   fixed windows (final partial window kept when ≥ 50% of window_ms), all channels'
   interleaved samples pooled per window, `rms_dbfs = 20·log10(rms/32768)` with
   silence floored at −120.0, `clip_ratio` = fraction of samples with
   `abs(sample) ≥ 32767`; loudness helpers delegating to ffmpeg builders.
5. Timeout defaults: `2 × media_duration + 120 s` helper `media_timeout(duration_ms)`.

## Acceptance Criteria

- [ ] `run()` never inherits unlisted env vars (test asserts `OPENAI_API_KEY` absent in
      child env via `env` dump helper binary/`printenv`).
- [ ] Timeout kills a `sleep`-like ffmpeg invocation and raises DS-STAGE-005 with tail.
- [ ] `probe()` on the generated fixture video (issue 40 script or inline tiny fixture)
      returns correct duration ±20 ms, stream layout.
- [ ] `atempo` rejects ratio outside [0.5, 2.0].
- [ ] `extract_audio` → `wav_duration_ms` round-trip matches probe duration ±20 ms.
- [ ] `place_on_timeline` with 3 tones at known offsets yields energy exactly in
      those windows and output duration == duration_ms ±20 ms (shared assertion
      style with issue 28's stage test).
- [ ] `ensure_under` rejects an escaping temp/dst path (negative test); a builder
      test proves temp files land next to dst.
- [ ] Non-zero-exit ffmpeg with `check=True` surfaces `CalledProcError` with tails;
      `check=False` returns rc.
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
