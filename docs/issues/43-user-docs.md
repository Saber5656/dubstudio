# Issue 43: User documentation

## Title

Write user documentation: quickstart (EN/JA), provider guides, workflow, troubleshooting

## Summary

Write the end-user docs set under `docs/guide/` plus the finalized README quickstart,
covering install→first-dub in minutes, per-provider setup with security notes, the
review/edit workflow (CLI files and UI), configuration reference, and troubleshooting
mapped to error codes.

## Context

Persona 1 (§1.2) judges the product by time-to-first-dub. Docs are also where the
§11.2 B2 privacy disclosures and §11.4 responsible-use guidance become user-visible.
Written last so every command/flag documented actually exists (deps 31–33, 39).

## Scope

In: the doc set below + README quickstart replacement (03 placeholder) + `docs/guide/`
nav from README. Out: API/reference docs for plugin authors (v2), translations beyond
the ja quickstart.

## Detailed Requirements

1. `docs/guide/quickstart.md` (EN) + `docs/guide/quickstart.ja.md` (JA, full
   translation): prerequisites (ffmpeg install per-OS, Python/uv), `uvx dubstudio`
   path and `pipx` path, 6-step first dub (init → doctor → consent → run → review →
   export outputs), copy-pasteable with expected output snippets (from the mock or a
   real tiny run), total reading time target < 5 min.
2. `docs/guide/providers.md`: per provider — what it does, local vs cloud, **exactly
   what data leaves your machine** (B2 table), key setup (`export OPENAI_API_KEY=…`
   / `.env`), extras install line, model download size/location, cost model pointer,
   capability notes (watermark for Chatterbox per ADR-005; ElevenLabs IVC tier
   requirement; timestamps caveat from U-01 as verified in I15; ja-quality note from
   U-02 as evaluated in I18).
3. `docs/guide/workflow.md`: the review loop — `run --until translate`, editing via
   `segments export/import` (schema documented with an annotated example), the UI
   flow (launch, edit, re-synth, run), staleness mental model (§6.2 in user terms:
   "edit anything → only affected work re-runs"), `--require-approved` gate,
   voice reference management (`voice set/auto/show`, what makes a good reference).
4. `docs/guide/configuration.md`: generated-style reference of every config key
   (table: key, default, effect, stage it invalidates) — source of truth is issue
   06's models; a doc-freshness test compares the documented key set against the
   pydantic schema (missing/extra keys fail CI).
5. `docs/guide/troubleshooting.md`: every `DS-*` code from the issue 05 catalog with
   one-paragraph fixes (doc-freshness test: catalog codes ⊆ documented codes);
   plus the top non-error problems: ffmpeg missing per-OS, GPU/MPS notes, long-video
   memory (limits), dub sounds rushed (fit tuning: atempo_max, cps table),
   voice sounds wrong (reference quality checklist).
6. README: replace quickstart placeholder with the real 5-command version; verify
   the responsible-use section links (03) still hold; badges (CI, PyPI once
   published).
7. Every command/flag/output in docs must be real: a docs-accuracy pass runs each
   quoted command against the built package (manual checklist in PR + the two
   freshness tests wired into CI).

## Acceptance Criteria

- [ ] Quickstart executed verbatim by the implementer on a clean macOS and Linux env
      (mock or real tiny video) — evidence (terminal transcript) in PR.
- [ ] JA quickstart is a faithful full translation (not summary).
- [ ] Config-reference and troubleshooting freshness tests green and wired into CI.
- [ ] Providers page includes the B2 data-disclosure table for all 7 providers.
- [ ] No documented command differs from `--help` output (spot-check script or manual
      checklist).

## Validation

CI freshness tests + PR checklist with transcripts; one external-reader review pass
(maintainer) for the quickstart.

## Dependencies

31, 32, 33, 39 (12, 11 content), 03.

## Non-goals

Plugin-author docs, video tutorials, hosted docs site (plain GitHub markdown in v1).

## Design References

DESIGN.md §1.2, §2.1, §8, §11.2 B2, §11.4, §12; ISSUE_PLAN §6.6.
