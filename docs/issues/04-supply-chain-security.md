# Issue 04: Supply-chain hardening

## Title

Add Dependabot, CodeQL, and pip-audit to the repository security baseline

## Summary

Configure automated dependency updates, static code scanning, and dependency
vulnerability auditing per trust boundary B7 (DESIGN.md §11.2).

## Context

Public OSS handling API keys and user media; the release chain must be hardened before
meaningful code accumulates.

## Scope

In: `.github/dependabot.yml`, CodeQL workflow, pip-audit CI job, action-pinning policy
note. Out: PyPI publishing security (issue 42), runtime security controls.

## Detailed Requirements

1. `dependabot.yml`: weekly `pip` updates (grouped minor/patch) and weekly
   `github-actions` updates; PR limit 5 each.
2. `.github/workflows/codeql.yml`: language `python`; triggers: PR to main, push to
   main, weekly schedule, **and `workflow_dispatch`** (so it can be exercised from the
   feature branch); pinned by SHA; `permissions` minimal
   (`security-events: write`, `contents: read`).
3. pip-audit job (in the issue 02 workflow as a separate job):
   - add `pip-audit` to the dev dependency group (issue 01's pyproject; update
     `uv.lock` in this PR);
   - command: `uv export --format requirements-txt --no-emit-project > /tmp/req.txt`
     then `uv run pip-audit -r /tmp/req.txt --strict --disable-pip $(python
     .github/scripts/audit_ignores.py)`;
   - `.github/pip-audit-ignore.txt` format: one advisory per line,
     `<ID>  # expires:YYYY-MM-DD  <reason>`; the helper script
     `.github/scripts/audit_ignores.py` emits `--ignore-vuln <ID>` for unexpired
     entries and **exits nonzero if any entry is expired** (forces re-triage).
4. Add a short `docs/security/supply-chain.md` note documenting: actions pinned by SHA,
   Dependabot cadence, lockfile policy (uv.lock committed; PRs must not regenerate the
   lock without dependency changes), and advisory-acceptance rules.
5. `main` branch protection: verify a repository ruleset exists enforcing — PRs
   required before merge, required status check = CI, force pushes and deletions
   blocked. If absent, configure it (`gh api repos/{owner}/{repo}/rulesets` — settings
   change performed by the maintainer account and evidenced in the PR). Record the
   effective ruleset JSON (secrets-free) in `docs/security/supply-chain.md`.

## Acceptance Criteria

- [ ] Dependabot config validates (GitHub UI shows both ecosystems enabled).
- [ ] CodeQL completes green via `workflow_dispatch` on the feature branch; the
      scheduled/main run is confirmed post-merge in a PR follow-up comment.
- [ ] pip-audit job runs in CI and fails on a known-vulnerable pin (verified once with a
      temporary test pin, then reverted); an expired ignore entry also fails (unit test
      for `audit_ignores.py`).
- [ ] All workflows have explicit `permissions` and SHA-pinned actions.
- [ ] `main` ruleset active: direct push rejected (evidence: attempted push output or
      ruleset JSON in `docs/security/supply-chain.md`), PR + green CI required.

## Validation

Trigger each workflow once (PR + `workflow_dispatch`); attach run links in the PR
description; include the ruleset evidence.

## Dependencies

02.

## Non-goals

SBOM generation, OpenSSF Scorecard, signed commits enforcement (candidates for v2).

## Design References

DESIGN.md §11.2 B7, §11.6.
