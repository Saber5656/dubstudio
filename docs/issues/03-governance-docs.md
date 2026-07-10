# Issue 03: Governance and policy docs

## Title

Add LICENSE, NOTICE, README, CONTRIBUTING, SECURITY, CODE_OF_CONDUCT, and voice POLICY

## Summary

Create the repository governance set for a public Apache-2.0 OSS project, including the
versioned Voice & Likeness Policy text that the consent gate (issue 11) displays and
records.

## Context

dubstudio is public from day one and ships voice cloning; ADR-005 and ADR-007 require
the policy/licensing scaffolding to exist before any provider or synthesis code lands.

## Scope

In: the seven documents below, in English (README gets a short Japanese companion).
Out: user guides/quickstart details (issue 43), consent gate implementation (issue 11).

## Detailed Requirements

1. `LICENSE`: verbatim Apache-2.0 text.
2. `NOTICE`: `dubstudio\nCopyright 2026 dubstudio contributors` (extend only when
   attribution obligations appear).
3. `README.md` (EN), sections in order: pitch (DESIGN.md §1.1), feature bullets (§2.1),
   status banner ("pre-release, under active development"), quickstart placeholder
   pointing to docs/, **Responsible use** section (summary of POLICY.md: consent
   required, AI-disclosure defaults, watermark-by-default local TTS), links to
   DESIGN/ISSUE_PLAN, license note. Keep under 120 lines.
4. `README.ja.md`: Japanese translation of pitch + feature bullets + responsible-use
   summary, linking to README.md for the rest.
5. `CONTRIBUTING.md`: dev setup (`uv sync --all-extras --dev`, `uv run pytest`,
   ruff/mypy commands), branch/PR rules (PRs to `main`, CI green required), inbound=
   outbound licensing note per ADR-007 (no CLA), issue workflow note that
   `docs/ISSUE_PLAN.md` is authoritative.
6. `SECURITY.md`: report via GitHub private vulnerability reporting; response target
   14 days; supported versions = latest minor; explicit note per DESIGN.md §11.2 B1 that
   processing untrusted media inherits ffmpeg's parser attack surface and should be
   sandboxed by the operator.
7. `CODE_OF_CONDUCT.md`: Contributor Covenant v2.1 with project contact method.
8. `docs/POLICY.md` — Voice & Likeness Policy **v1** (versioned header `policy_version:
   1`): plain-language rules — clone only voices you own or have documented consent
   for; disclose AI-generated audio where required by platform/law; prohibited uses
   (impersonation, fraud, harassment, deception about a real person's speech); a
   consent-affirmation sentence exactly matching DESIGN.md §11.4 (the consent gate
   displays this string verbatim); reporting/abuse contact; note that policy version
   bumps re-trigger the consent gate.

## Acceptance Criteria

- [ ] All seven files exist with the required content and render cleanly on GitHub.
- [ ] POLICY.md contains `policy_version: 1` and the exact §11.4 affirmation sentence.
- [ ] README responsible-use section links to POLICY.md; README.ja.md links back.
- [ ] `gh repo view` shows license detected as Apache-2.0.

## Validation

Manual render check on GitHub; markdown lint (CI lint job) passes; grep the affirmation
sentence to confirm exact match with the string constant later used by issue 11.

## Dependencies

None (repo only). Issue 11 consumes POLICY.md.

## Non-goals

Full user documentation (43), release checklist (42), translations beyond ja summary.

## Design References

DESIGN.md §1, §2.1, §11.4; ADR-005, ADR-007.
