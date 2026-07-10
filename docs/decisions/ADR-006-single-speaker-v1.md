# ADR-006: v1 assumes a single speaker (one cloned voice per project)

- Status: accepted
- Date: 2026-07-10

## Context

The primary persona is a solo creator dubbing their own videos (DESIGN.md §1.2).
Multi-speaker dubbing requires diarization (who spoke when), per-speaker reference
building, per-speaker consent, and per-segment voice routing — a large complexity and
quality-risk multiplier (diarization errors compound into wrong-voice dubbing).

## Decision

v1 processes every segment with **one** cloned voice built from the project's reference
(auto-extracted or user-supplied). No diarization dependency (WhisperX/pyannote not
included). The segment schema carries no speaker field in v1; adding one is a
backward-compatible extension (`schema_version` bump) reserved for v2.

## Consequences

- Multi-speaker videos will dub all speech in one voice — documented limitation in
  README and `doctor`/`status` cannot detect it (user judgment).
- v2 diarization work is additive: new stage + schema extension + per-speaker voice_ref.

## Alternatives considered

- Optional diarization in v1: pulls in pyannote (gated models, license friction),
  doubles voice_ref/synthesize/consent complexity — rejected for v1.
