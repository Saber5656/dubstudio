# Issue 35: UI read API

## Title

Implement UI read endpoints: project, segments, audio streaming, fit report

## Summary

Implement the GET routes of DESIGN.md §10.3 on the issue 34 foundation: project
summary, the merged segments view the frontend renders, Range-capable audio streaming
by id (including on-the-fly original-segment slicing with cache), and the fit report.

## Context

The segments view is the UI's core data structure — a join of transcript, translation,
synth, and fit docs. Audio routes are the only artifact byte access and must stay
id-based (B6).

## Scope

In: `GET /api/project`, `/api/segments`, `/api/audio/{kind}/{id}`, `/api/fit-report`.
Out: mutations/jobs/SSE (36), frontend (37/38).

## Detailed Requirements

1. `GET /api/project`: manifest summary — project name (dir basename), input basename,
   duration_ms, languages, per-stage `{status, finished_at, error?}` grouped like
   `status` (31), consent snapshot presence, providers in effect (names + cloud flag +
   watermark flag). No absolute paths in the payload (relative/base names only).
2. `GET /api/segments?lang=L` (lang required, validated against manifest targets):
   rows joining docs that exist so far — `{id, start_ms, end_ms, slot_ms, source_text,
   text?, status?, char_budget?, over_budget?, synth: {duration_ms, cached_at}?,
   fit: {result, atempo, overrun_ms}?, warnings: [str]}`; missing docs → partial rows
   (frontend renders progressively). ETag = sha256 of underlying doc hashes; 304
   support.
3. `GET /api/audio/{kind}/{id}?lang=L`, kind ∈ `source|synth|fit`:
   - `synth`/`fit`: stream the artifact WAV (id validated `^seg_\d{4}$`, path resolved
     via store registry only);
   - `source`: slice `[start_ms, end_ms]` of the vocal source on first request via
     issue 07 `slice_audio` into `artifacts/ui_cache/source/<id>.wav`, then stream;
     cache invalidated by transcript hash embedded in the cache filename;
   - HTTP Range supported (206) for scrubbing; `Accept-Ranges: bytes`;
     `Content-Type: audio/wav`; 404 JSON error when the artifact doesn't exist yet.
4. `GET /api/fit-report?lang=L`: the FitReport doc as-is + summary counts.
5. All routes read-only (no lock); concurrent mutation safety = atomic-rename reads
   (may serve the pre-write version — acceptable, documented).
6. `artifacts/ui_cache/` added to the store path registry (09) and .gitignore
   template.

## Acceptance Criteria

- [ ] Segments join correctness on a fixture project at three completion levels
      (after transcribe only / after translate / after fit) — snapshot tests of the
      JSON rows.
- [ ] `lang=zz` → 422 with stable error shape; `id=../../x` → 422 (regex), never
      touches the filesystem.
- [ ] Range request returns 206 with correct byte slice (compare bytes).
- [ ] Source-slice cache: first request creates the file, second serves without
      re-slicing (mtime unchanged), transcript change (different hash) re-slices.
- [ ] ETag/304 behavior on `/api/segments`.
- [ ] No response payload contains an absolute filesystem path (regex scan test).

## Validation

`uv run pytest tests/ui/test_read_api.py` (TestClient + `-m media` for the real slice
test).

## Dependencies

34, 09, 07, 08.

## Non-goals

Waveform peaks endpoint (v2), pagination (segment counts in v1 scope are fine),
transcript editing.

## Design References

DESIGN.md §10.3, §11.2 B3/B6, §4.1 (ui_cache).
