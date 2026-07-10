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
   main, weekly schedule; pinned by SHA; `permissions` minimal
   (`security-events: write`, `contents: read`).
3. Extend CI (issue 02 workflow or separate job) with `pip-audit` against the resolved
   environment (`uv export --format requirements-txt | uv run pip-audit -r
   /dev/stdin --strict` or equivalent); failures block. Provide
   `.github/pip-audit-ignore.txt` mechanism (documented inline) for accepted advisories
   with expiry comments.
4. Add a short `docs/security/supply-chain.md` note documenting: actions pinned by SHA,
   Dependabot cadence, lockfile policy (uv.lock committed; PRs must not regenerate the
   lock without dependency changes), and advisory-acceptance rules.

## Acceptance Criteria

- [ ] Dependabot config validates (GitHub UI shows both ecosystems enabled).
- [ ] CodeQL run completes green on main.
- [ ] pip-audit job runs in CI and fails on a known-vulnerable pin (verified once with a
      temporary test pin, then reverted).
- [ ] All workflows have explicit `permissions` and SHA-pinned actions.

## Validation

Trigger each workflow once (PR + manual dispatch where applicable); attach run links in
the PR description.

## Dependencies

02.

## Non-goals

SBOM generation, OpenSSF Scorecard, signed commits enforcement (candidates for v2).

## Design References

DESIGN.md §11.2 B7, §11.6.
