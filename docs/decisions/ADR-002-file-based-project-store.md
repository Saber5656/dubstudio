# ADR-002: File-based project store with fingerprint-driven resumable stages

- Status: accepted
- Date: 2026-07-10

## Context

The pipeline is long-running and expensive (API cost, GPU time). Users must be able to
edit intermediate artifacts (translations) and re-run only what changed. The tool is
local-first, single-user, and should be transparent and debuggable.

## Decision

No database. A project is a directory (DESIGN.md §4): `manifest.json` for stage state,
JSON artifacts per stage, WAV audio artifacts, JSONL run logs. Staleness is computed from
content-hash fingerprints (inputs + config subset + provider identity), cascading to
descendants; synthesize/fit additionally track per-segment hashes for segment-level
incremental re-runs. Writes are atomic (`tmp` + `os.replace`); a project-level advisory
lock enforces a single writer.

## Consequences

- Users can inspect, diff, back up, or hand-edit artifacts with normal tools; the review
  UI and CLI edit the same files.
- No migrations infra beyond `schema_version` gates in v1.
- Cross-process concurrency is deliberately coarse (one mutating command at a time).

## Alternatives considered

- **SQLite state**: better queryability, but hides state from users, complicates HITL
  editing, and adds migration burden disproportionate to v1 needs.
- **Make/DAG tools (snakemake etc.)**: wrong UX for an end-user product; fingerprint
  semantics need to be domain-aware (per-segment).
