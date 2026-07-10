# Issue 42: Packaging and release automation

## Title

Implement PyPI release automation with Trusted Publishing and CHANGELOG discipline

## Summary

Implement the release pipeline per DESIGN.md §14: tag-triggered GitHub Actions workflow
building sdist+wheel and publishing to PyPI via Trusted Publishing (OIDC), GitHub
Release creation, keep-a-changelog `CHANGELOG.md`, and the release checklist including
the U-06 name check.

## Context

`uvx dubstudio` is the promised install path (§2.1); supply-chain rules B7/§11.6 apply
to the publish channel. **PyPI Trusted Publisher registration is a manual step the
maintainer performs in the PyPI UI** — this issue documents it; agents must not handle
credentials.

## Scope

In: release workflow, changelog, version policy, release checklist doc. Out: CI test
workflow (02), Homebrew/other channels (v2).

## Detailed Requirements

1. `.github/workflows/release.yml`: trigger `push: tags: ["v*"]`;
   jobs: `test` (reuse CI via `workflow_call` on issue 02's workflow) → `build`
   (`uv build`; `twine check dist/*`; upload artifacts) → `publish` (environment
   `pypi`, `permissions: id-token: write`, `pypa/gh-action-pypi-publish` pinned by
   SHA, no secrets) → `github-release` (creates the Release with generated notes +
   the CHANGELOG section for the tag, attaches dist files). Tag↔version guard: job
   fails if `pyproject.toml` version ≠ tag (strip `v`).
2. Versioning: SemVer 0.x; version lives only in `pyproject.toml`
   (`dubstudio.__version__` reads metadata, issue 01); `0.1.0` is the v1-complete
   release per ISSUE_PLAN §1.
3. `CHANGELOG.md`: keep-a-changelog format, `Unreleased` section required; CI (02)
   gains a lightweight check: PRs touching `src/` must touch `CHANGELOG.md`
   (skippable with label `no-changelog`).
4. `docs/dev/release-checklist.md`: ordered steps — CHANGELOG finalize; version bump
   PR; `-m live` smoke (41) results recorded; dogfood exit test (ISSUE_PLAN §6.6);
   **U-06: verify PyPI name `dubstudio` is available/owned and run a basic trademark
   search — if taken/conflicted, STOP and escalate to the maintainer (rename is a
   user decision)**; tag + push; verify PyPI page, `uvx dubstudio version`, GitHub
   Release. Includes the one-time Trusted Publisher setup instructions (maintainer
   manual step, with exact PyPI UI fields: owner, repo, workflow file, environment).
5. sdist/wheel content policy: exclude `tests/`, `docs/` (except LICENSE/NOTICE/
   POLICY.md which ship in the wheel — POLICY.md is read by the consent gate? No:
   issue 11 embeds the affirmation constant; POLICY.md ships for reference only) —
   define `[tool.hatch.build]`/equivalent includes explicitly; wheel imports cleanly
   without dev files (`python -c "import dubstudio"` from wheel in the workflow).

## Acceptance Criteria

- [ ] Dry run on a fork/test tag (`v0.0.1.dev1` to TestPyPI via a temporary parallel
      environment config) succeeds end-to-end; evidence linked in PR.
- [ ] Tag/version mismatch fails the workflow (tested with a bad tag on the fork).
- [ ] `uv build` artifacts pass `twine check`; wheel import-smoke green in workflow.
- [ ] Changelog-check behaves (PR without CHANGELOG change fails unless labeled).
- [ ] Checklist doc complete incl. Trusted Publisher manual setup + U-06 gate; no
      secret/token appears anywhere in workflows (OIDC only).

## Validation

Fork-based dry run + workflow lint (`actionlint` locally); reviewer walks the
checklist against a mock release.

## Dependencies

01, 02 (04 for pinning conventions).

## Non-goals

Homebrew tap, conda, Docker images, signed sigstore bundles (v2 candidates),
auto-generated release notes beyond the CHANGELOG section.

## Design References

DESIGN.md §14, §11.6, §11.2 B7; ISSUE_PLAN §1, U-06.
