# Issue 09: Project store

## Title

Implement project directory store: layout, atomic IO, artifact paths, locking, hashing

## Summary

Implement `project/store.py`, `project/lock.py`, and `core/hashing.py`: create/open
project directories per DESIGN.md §4.1, atomic manifest/artifact JSON IO, id-based
artifact path resolution, single-writer lock, and content hashing.

## Context

The store is the only writer of project state (ADR-002) and enforces path safety (B6).

## Scope

In: store/lock/hashing modules + tests. Out: stage logic, staleness computation
(issue 10 uses hashing primitives from here).

## Detailed Requirements

1. `core/hashing.py`: `sha256_file(path, chunk=1 MiB) -> "sha256:<hex>"`;
   `sha256_text(text)` (NFC-normalized UTF-8); `fingerprint(payload: dict) -> str`
   (canonical JSON: sorted keys, no whitespace → sha256) — used by issue 10.
2. `ProjectStore.create(dir, *, input_path, source_lang, targets, copy_input: bool)`:
   - refuses non-empty target dir (`ConfigError DS-CONFIG-002`);
   - probes input (issue 07), builds Manifest with all stage keys for the configured
     targets in status `pending`, `separate` honoring `separation.enabled`;
   - `copy_input=True` → copy to `<project>/input/<name>` and reference that path;
   - writes `dubstudio.toml` (template with the project table filled + commented
     defaults), `manifest.json`, `.gitignore` (`artifacts/`, `logs/`, `.env`,
     `.dubstudio.lock`, `input/` when copied).
3. `ProjectStore.open(dir)`: locates project root by walking up ≤ 3 levels for
   `manifest.json` (explicit `--project` beats walking); validates manifest through
   models (issue 08).
4. Path registry: `art(stage, lang=None, *names) -> Path` mapping exactly the §4.1
   tree (single source of truth table in code, e.g. `art("translate", "en",
   "translation.json")`); creates parent dirs; asserts resolved path
   `is_relative_to(project_root/"artifacts")`.
5. Atomic JSON: `write_json(path, model)` → tmp file in same dir + fsync +
   `os.replace`; `read_json(path, model_cls)` with `parse_document` gate;
   corrupted JSON → `StageError DS-STAGE-006` naming the file and suggesting
   `invalidate --stage`.
6. `project/lock.py`: `acquire(project_root, command: str)` context manager —
   `.dubstudio.lock` JSON `{pid, started_at, command}` created with `O_EXCL`; on
   conflict: if pid dead → reclaim with warning log; else `LockHeldError DS-LOCK-001`
   (message includes holder pid/command). `fcntl.flock` used additionally where
   available.
7. Input verification helper `verify_input(manifest) -> InputCheck` re-hashing lazily:
   size+mtime fast path; full sha256 when they changed; mismatch → structured result
   (consumed by ingest stage and `invalidate --input`).
8. Manifest update API: `mutate_manifest(fn)` — read → fn(manifest) → atomic write,
   requiring the lock to be held (assert).

## Acceptance Criteria

- [ ] `create` produces exactly the §4.1 tree (golden listing test) and a loadable
      manifest with correct stage keys for `targets=["en","de"]`.
- [ ] Kill -9 during `write_json` (simulated: crash between tmp write and replace)
      leaves the previous file intact.
- [ ] Second `acquire` in another process (subprocess test) raises DS-LOCK-001; after
      killing holder, reclaim succeeds with warning.
- [ ] `art()` rejects traversal (`lang="../x"` → error) and unknown stage names.
- [ ] `verify_input` detects a 1-byte modification of the input file.

## Validation

`uv run pytest tests/project/` incl. the subprocess lock test; mypy strict.

## Dependencies

08 (07 for probe in `create`).

## Non-goals

Fingerprint/staleness policy (issue 10), UI cache paths (issue 35 adds
`artifacts/ui_cache/` through this registry).

## Design References

DESIGN.md §4.1, §4.4, §4.5, §11.2 B6; ADR-002.
