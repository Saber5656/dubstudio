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
2. `ProjectStore.create(dir, *, input_path, source_lang, targets, copy_input: bool,
   separation_enabled: bool = True)`:
   - refuses non-empty target dir (`ConfigError DS-CONFIG-002`);
   - probes input (issue 07), builds Manifest with all stage keys for the configured
     targets in status `pending`; the `separate` stage record starts `pending` when
     `separation_enabled` else `skipped` (the flag is passed explicitly by the CLI,
     which owns config — no dependency on the config system here);
   - `copy_input=True` → copy to `<project>/input/<name>` and reference that path;
   - writes `dubstudio.toml` (project table filled from the arguments; the rest is a
     **static commented template** maintained as a string constant in this module,
     not derived from Settings), `manifest.json`, `.gitignore` (`artifacts/`,
     `logs/`, `.env`, `.dubstudio.lock`, plus `input/` when copied);
   - initial on-disk tree (golden-tested, both modes): `dubstudio.toml`,
     `manifest.json`, `.gitignore`, empty `artifacts/` and `logs/` dirs, and
     `input/<name>` only when `copy_input=True`. Stage subdirectories under
     `artifacts/` are created lazily by `art()`, not at create time.
3. `ProjectStore.open(dir)`: locates project root by walking up ≤ 3 levels for
   `manifest.json` (explicit `--project` beats walking); validates manifest through
   models (issue 08).
4. Path registry: `art(stage, lang=None, *names) -> Path` mapping exactly the §4.1
   tree (single source of truth table in code, e.g. `art("translate", "en",
   "translation.json")`); creates parent dirs; every component of `names` must be a
   plain filename (no separators, no `.`/`..`, non-empty) and the resolved path must
   satisfy `is_relative_to(project_root/"artifacts")` — violations raise
   `ConfigError DS-CONFIG-002`, not assert. Segment audio paths go through
   `segment_art(stage, lang, seg_id, ext=".wav")` which validates
   `seg_id` against `^seg_\d{4}$` before delegating to `art()`.
5. Atomic JSON: `write_json(path, model)` → serialize via issue 08's
   `to_canonical_json` (UTF-8, 2-space indent, trailing newline — byte format
   asserted by test), tmp file in same dir + fsync + `os.replace`;
   `read_json(path, model_cls)` with `parse_document` gate; corrupted JSON →
   `StageError DS-STAGE-006` naming the file and suggesting `invalidate --stage`.
6. `project/lock.py`: `acquire(project_root, command: str)` context manager —
   `.dubstudio.lock` JSON `{pid, started_at, command}` created with `O_EXCL`; on
   conflict: if pid dead → reclaim with warning log; else `LockHeldError DS-LOCK-001`
   (message includes holder pid/command). `fcntl.flock` used additionally where
   available.
7. Input verification helper `verify_input(manifest) -> InputCheck` re-hashing lazily:
   size+mtime fast path; full sha256 when they changed; mismatch → structured result
   (consumed by ingest stage and `invalidate --input`).
8. Manifest update API: `mutate_manifest(lock_token, fn)` — read → fn(manifest) →
   atomic write. `acquire()` (req 6) returns an opaque `LockToken`; passing a missing
   or stale token raises `LockHeldError DS-LOCK-002` ("manifest mutation without
   holding the project lock") — a real check, not `assert`.

## Acceptance Criteria

- [ ] `create` produces exactly the documented initial tree (golden listing test,
      both copy modes) and a loadable manifest with correct stage keys for
      `targets=["en","de"]`, incl. `separate` = `skipped` when
      `separation_enabled=False`.
- [ ] Kill -9 during `write_json` (simulated: crash between tmp write and replace)
      leaves the previous file intact.
- [ ] Second `acquire` in another process (subprocess test) raises DS-LOCK-001; after
      killing holder, reclaim succeeds with warning.
- [ ] `art()` rejects traversal (`lang="../x"`, `names` containing `/` or `..`) and
      unknown stage names with DS-CONFIG-002; `segment_art` rejects `seg_12345` and
      `seg_00x1`.
- [ ] `write_json` byte format matches `to_canonical_json` exactly (golden bytes).
- [ ] `mutate_manifest` without a valid LockToken raises DS-LOCK-002.
- [ ] `verify_input` detects a 1-byte modification of the input file.

## Validation

`uv run pytest tests/project/` incl. the subprocess lock test; mypy strict.

## Dependencies

07 (probe in `create`), 08.

## Non-goals

Fingerprint/staleness policy (issue 10), UI cache paths (issue 35 adds
`artifacts/ui_cache/` through this registry).

## Design References

DESIGN.md §4.1, §4.4, §4.5, §11.2 B6; ADR-002.
