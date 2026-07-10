# Issue 02: CI pipeline

## Title

Add GitHub Actions CI: lint, type-check, and test matrix on Linux and macOS

## Summary

Create `.github/workflows/ci.yml` running ruff, mypy, and pytest (with coverage gate)
on ubuntu-latest and macos-latest for Python 3.12 and 3.13, with uv caching, ffmpeg
installed for `media`-marked tests, pinned actions, and least-privilege permissions.

## Context

CI is the enforcement mechanism for every later issue's Validation section (DESIGN.md
§13) and part of the supply-chain posture (§11.2 B7).

## Scope

In: one CI workflow + coverage config. Out: release/publish workflow (issue 42), CodeQL
and Dependabot (issue 04).

## Detailed Requirements

1. Workflow `ci.yml`: triggers `pull_request` and `push` to `main`; concurrency group
   `ci-${{ github.ref }}` with `cancel-in-progress: true`; top-level
   `permissions: contents: read`.
2. Jobs:
   - `lint`: `ruff check .` and `ruff format --check .` (ubuntu, py3.12 only).
   - `typecheck`: `mypy src` (ubuntu, py3.12 only).
   - `test`: matrix `os ∈ {ubuntu-latest, macos-latest}` × `python ∈ {3.12, 3.13}`;
     installs ffmpeg (`apt-get install -y ffmpeg` / `brew install ffmpeg`); a step
     prints `ffmpeg -version` and `ffprobe -version` and fails if either major
     version < 6 (DESIGN §3.3); runs
     `uv run pytest --cov=dubstudio --cov-report=xml -m "not live"`.
     Coverage is **collected** from day one; the enforcement gate starts at
     `fail_under = 0` with a `# ratchet` comment — issue 40 raises it to the DESIGN
     §13 targets (80% overall, 90% for core/engine/model) once the test base exists.
3. Use `astral-sh/setup-uv` with built-in cache; `uv sync --dev` (core tests must not
   require torch extras; `media`-marked tests skip gracefully when ffmpeg absent but CI
   always installs ffmpeg).
4. All third-party actions pinned to full commit SHA with a version comment.
5. Coverage configured in `pyproject.toml` (`[tool.coverage.run] source=["dubstudio"]`,
   omit `*/ui/static/*`; `[tool.coverage.report] fail_under = 0  # ratchet: issue 40
   raises to DESIGN §13 targets`).
6. Add a CI status badge placeholder line to README if README exists (do not create
   README here).

## Acceptance Criteria

- [ ] CI runs and is green on a PR containing only the scaffold (issue 01).
- [ ] A deliberately mis-formatted file fails `lint` (verified once, then reverted).
- [ ] Matrix runs 4 test jobs; ffmpeg AND ffprobe available in all of them, with the
      version-gate step failing on major < 6 (verified once by faking an old version
      string in the step's parser test or by inspection).
- [ ] All actions referenced by commit SHA; `permissions` block present.
- [ ] Coverage collected and reported in CI logs; `fail_under = 0` present with the
      ratchet comment pointing at issue 40 (gate values are 40's deliverable).

## Validation

Open a draft PR with the workflow; confirm all jobs green; confirm the failure case for
lint; check job logs show uv cache hit on second run.

## Dependencies

01.

## Non-goals

Publishing, CodeQL, pip-audit, Windows runners (known unknown U-05).

## Design References

DESIGN.md §13, §11.2 B7, §11.6.
