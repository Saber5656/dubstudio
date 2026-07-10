# ADR-007: License is Apache-2.0

- Status: accepted
- Date: 2026-07-10
- Decision drivers: user selection (2026-07-10), after MIT/Apache comparison.

## Context

Public OSS in a patent-active domain (AI speech synthesis, voice cloning). External
contributions are expected.

## Decision

Apache-2.0 for all repository code. `LICENSE` + `NOTICE` files at repo root; SPDX
headers not required per-file in v1. Contributions are accepted under Apache-2.0 §5
(inbound=outbound, no CLA).

## Consequences

- Explicit patent grant + retaliation clause protects users; contribution terms are
  defined without a CLA.
- GPLv2-only code cannot be vendored (GPLv3-compatible only) — relevant when evaluating
  audio libraries.
- Dependencies and **model weights** must be license-checked against redistribution and
  commercial-use expectations: IndexTTS-2 / Fish Speech / XTTS-v2 are excluded from
  defaults for exactly this reason (research doc §3); Chatterbox (MIT), faster-whisper
  (MIT), Demucs (MIT) are compatible. Model weights are downloaded at runtime, never
  vendored into the repo or wheels.
