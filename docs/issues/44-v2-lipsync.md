# Issue 44: [v2] Lip-sync stage

## Title

[v2] Research and integrate an optional lip-sync stage

## Summary

Post-v1: add an optional `lipsync` pipeline stage that retimes the speaker's mouth
region to match the dubbed audio, as a new stage between `mix` and `export`, behind a
dedicated extra and explicit config opt-in. This issue is a v2 placeholder created at
the user's request; it starts with a research spike, not implementation.

## Context

Requirements interview (2026-07-10): lip-sync is explicitly v2 — high technical risk,
heavy compute, unstable quality (DESIGN.md §2.2; Linly-Dubbing's complexity noted in
`docs/research/2026-07-dubbing-tools-landscape.md`). The v1 architecture reserves a
clean insertion point: video is stream-copied everywhere (§5.9), so a lip-sync stage
is the first and only video-modifying stage.

## Scope

Phase A (research spike, timeboxed): evaluate current open lip-sync models
(2025–2026 generation — e.g. MuseTalk/LatentSync successors current at execution
time) on: license compatibility (Apache-2.0 distributable), ja/en visual quality,
GPU requirements, face-detection dependency chain, batch throughput. Deliverable:
`docs/research/lipsync-landscape.md` + an ADR proposing the model and integration
shape, reviewed before Phase B is planned.
Phase B (only after ADR approval): implementation issues to be drafted then (stage,
provider-style abstraction, UI preview, safety notes).

## Detailed Requirements

1. Phase A produces the research doc with a comparison table (model, license, VRAM,
   fps, identity preservation, failure modes on glasses/beards/profile faces) and a
   recommendation with evidence clips generated from the issue 40 fixture + one real
   consenting-face sample (maintainer's own footage).
2. Integration constraints Phase B must honor (recorded now so v1 code doesn't
   preclude them):
   - stage key `lipsync:<lang>` between `mix:<lang>` and `export:<lang>`; skipped by
     default (`lipsync.enabled=false`);
   - video re-encoding becomes unavoidable → export's stream-copy invariant
     (I29 AC) becomes conditional on lip-sync disabled; the md5 test gains a branch;
   - face/identity processing raises the safety bar: consent gate text needs a
     policy_version bump adding visual-likeness language (ADR-005 mechanism);
   - disclosure metadata gains `lipsync` marker (§11.4).
3. Non-negotiable safety AC for Phase B: lip-sync refuses to run when the consent
   policy version predates the visual-likeness clause.

## Acceptance Criteria

- [ ] (Phase A) research doc + ADR merged; go/no-go decision recorded with the user.
- [ ] v1 codebase audit confirms no additional coupling was introduced that blocks
      the integration constraints above.

## Validation

Research doc review by maintainer; evidence clips attached to the ADR PR.

## Dependencies

v1 complete (issues 01–43); explicitly excluded from v1 completion (ISSUE_PLAN §1).

## Non-goals

Full-face reenactment/avatar generation, realtime lip-sync, v1 delivery.

## Design References

DESIGN.md §2.2–2.3, §5.9, §11.4; ADR-005; ISSUE_PLAN §7.
