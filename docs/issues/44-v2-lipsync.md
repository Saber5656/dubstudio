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

Phase A (research spike, **timeboxed to 3 working days**): evaluate ≥ 4 candidate
open lip-sync models from ≥ 2 sources (Hugging Face, GitHub; 2025–2026 generation —
e.g. MuseTalk/LatentSync successors current at execution time) on: license
compatibility (Apache-2.0 distributable), supply-chain posture per §11.2 B4
(pinned official source/revision, safetensors availability, sha256/TOFU
verifiability, no `trust_remote_code` requirement — a model failing B4 is
disqualified), ja/en visual quality, GPU requirements, face-detection dependency
chain, batch throughput.
Go/no-go rubric (all must hold for "go"): license + B4 pass; runs within 16 GB
consumer VRAM; subjective identity preservation acceptable on both test clips;
throughput ≥ 0.2× realtime on the reference GPU. Otherwise record "no-go/defer"
with evidence.
Deliverable: `docs/research/lipsync-landscape.md` + an ADR proposing the model and
integration shape, reviewed before Phase B is planned.
Phase B (only after ADR approval): implementation issues to be drafted then (stage,
provider-style abstraction, UI preview, safety notes).

## Detailed Requirements

1. Phase A produces the research doc with a comparison table (model, license, B4
   posture, VRAM, fps, identity preservation, failure modes on
   glasses/beards/profile faces) and a recommendation with evidence clips from
   **two maintainer-provided, consent-cleared face clips** (the maintainer's own
   footage: one frontal talking-head, one with glasses/off-angle). The issue 40
   synthetic fixture has no face — it is used only for pipeline smoke, never as
   quality evidence.
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

- [ ] (Phase A) research doc + ADR merged; go/no-go decision (per the rubric)
      recorded with the user.
- [ ] v1 integration-readiness audit completed against this exact checklist, with
      findings noted in the ADR: (a) stage DAG accepts an insertion between
      `mix:<lang>` and `export:<lang>` without core changes; (b) export's video
      stream-copy invariant/md5 test is branchable on a config flag; (c) the config
      schema accepts a new `[lipsync]` table without breaking unknown-key
      validation; (d) the consent policy_version bump mechanism (ADR-005) supports
      adding visual-likeness language; (e) the disclosure metadata writer accepts an
      additional marker; (f) no other stage assumes video is never re-encoded.

## Validation

Research doc review by maintainer; evidence clips attached to the ADR PR.

## Dependencies

v1 complete (issues 01–43); explicitly excluded from v1 completion (ISSUE_PLAN §1).

## Non-goals

Full-face reenactment/avatar generation, realtime lip-sync, v1 delivery.

## Design References

DESIGN.md §2.2–2.3, §5.9, §11.4; ADR-005; ISSUE_PLAN §7.
