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

1. `.github/workflows/release.yml`: trigger `push: tags: ["v*"]`; **this issue also
   adds a `workflow_call` trigger to `.github/workflows/ci.yml`** (issue 02's file
   only has pull_request/push) so the release can reuse it.
   Jobs and least-privilege permissions (top-level `permissions: contents: read`):
   `test` (calls ci.yml via `workflow_call`) → `build` (`uv build`; `twine check
   dist/*`; upload artifacts; `contents: read`) → `publish` (environment `pypi`,
   `permissions: {contents: read, id-token: write}`, `pypa/gh-action-pypi-publish`
   pinned by SHA, no secrets) → `github-release` (`permissions: {contents: write}`
   — the only write-scoped job; creates the Release with generated notes + the
   CHANGELOG section for the tag, attaches dist files).
   Tag↔version guard: fails unless `pyproject.toml` version == tag minus `v`;
   the `publish` job additionally runs **only** for final-format tags
   (`^v\d+\.\d+\.\d+$`) — PEP 440 dev tags (e.g. `v0.0.1.dev1`) are allowed through
   build for TestPyPI dry runs via a separate `testpypi` environment/job gated on
   the dev-tag pattern.
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
5. Package content policy (explicit `[tool.hatch.build]`/equivalent config):
   - **wheel**: the package tree + root `LICENSE` and `NOTICE` via the
     `license-files` metadata; nothing from `docs/` or `tests/` (the consent gate
     embeds its affirmation constant per issue 11 — it does not read POLICY.md at
     runtime);
   - **sdist**: additionally includes `docs/POLICY.md`, `README.md`, `CHANGELOG.md`;
   - workflow smoke: install the built wheel in a clean venv and run
     `python -c "import dubstudio"` + `dubstudio version`.

## Acceptance Criteria

- [ ] Dry run with dev tag `v0.0.1.dev1` publishes to TestPyPI through the
      `testpypi` job while the production `publish` job is skipped (dev-tag gate
      proven); evidence linked in PR.
- [ ] Tag/version mismatch fails the workflow (tested with a bad tag on the fork).
- [ ] Permissions blocks match the least-privilege table above (workflow lint +
      review).
- [ ] `uv build` artifacts pass `twine check`; wheel import-smoke green in workflow.
- [ ] Changelog-check behaves (PR without CHANGELOG change fails unless labeled).
- [ ] Checklist doc complete incl. Trusted Publisher manual setup + U-06 gate; no
      secret/token appears anywhere in workflows (OIDC only).

## Validation

Fork-based dry run + workflow lint (`actionlint` locally); reviewer walks the
checklist against a mock release.

## Dependencies

01, 02 (ci.yml gains `workflow_call` here), 03 (LICENSE/NOTICE/POLICY.md must exist
to package) — matches the ISSUE_PLAN row; 04 for pinning conventions.

## Non-goals

Homebrew tap, conda, Docker images, signed sigstore bundles (v2 candidates),
auto-generated release notes beyond the CHANGELOG section.

## Design References

DESIGN.md §14, §11.6, §11.2 B7; ISSUE_PLAN §1, U-06.
