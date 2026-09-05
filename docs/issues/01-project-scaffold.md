# Issue 01: Project scaffold

## Title

Scaffold Python package with uv, src layout, Typer entry point, and dev tooling

## Summary

Create the installable `dubstudio` Python package skeleton exactly matching DESIGN.md
§3.2/§3.3: pyproject with core deps and extras, src layout with empty typed modules, a
working `dubstudio version` command, and ruff/mypy/pytest/pre-commit configuration.

## Context

Everything else builds on this skeleton. Module paths defined here are referenced by all
subsequent issues and must match DESIGN.md §3.2 exactly.

## Scope

In: pyproject.toml, uv.lock, package skeleton, tooling config, pre-commit, placeholder
test. Out: any feature logic, CI workflows (issue 02), repo governance docs (issue 03).

## Detailed Requirements

1. `pyproject.toml`:
   - `[project]` name `dubstudio`, dynamic-free version `0.1.0.dev0`, `requires-python
     = ">=3.12"`, license `Apache-2.0`, description from DESIGN.md §1.1 one-line pitch.
   - dependencies: `typer`, `pydantic>=2`, `pydantic-settings`, `httpx`, `rich`,
     `fastapi`, `uvicorn`, `platformdirs`.
   - `[project.optional-dependencies]`: `local-asr = ["faster-whisper"]`,
     `separate = ["demucs"]`, `local-tts = ["chatterbox-tts"]`,
     `local = ["dubstudio[local-asr,separate,local-tts]"]`.
   - `[dependency-groups]` dev: `pytest`, `pytest-cov`, `respx`, `ruff`, `mypy`,
     `pre-commit`.
   - `[project.scripts] dubstudio = "dubstudio.cli.app:main"`.
   - `[project.entry-points."dubstudio.providers"]` left empty (group reserved; see
     DESIGN.md §7.3).
2. Create exactly this tree under `src/dubstudio/` (matches DESIGN.md §3.2). Files
   marked (real) get working content per requirements 3–4; **every other module is
   docstring-only**, the docstring naming its owning DESIGN section (e.g.
   `"""Stage engine runner — DESIGN.md §6."""`); package `__init__.py` files are
   docstring-only too:

   ```
   __init__.py (real)   __main__.py (real)   py.typed (empty marker)
   cli/__init__.py app.py(real) init.py run.py status.py segments.py voice.py
       providers_cmd.py consent_cmd.py doctor.py ui_cmd.py invalidate.py config_cmd.py
   core/__init__.py errors.py logging.py config.py consent.py hashing.py procs.py
   media/__init__.py ffmpeg.py probe.py audio.py
   model/__init__.py manifest.py segments.py translation.py synthesis.py
       fitreport.py events.py
   project/__init__.py store.py lock.py
   engine/__init__.py graph.py state.py invalidate.py runner.py plan.py
   stages/__init__.py ingest.py separate.py transcribe.py translate.py voice_ref.py
       synthesize.py fit.py mix.py export.py subtitles.py
   providers/__init__.py base.py registry.py errors.py cost.py mock.py
   providers/asr/__init__.py faster_whisper.py openai_asr.py
   providers/mt/__init__.py openai_llm.py
   providers/tts/__init__.py chatterbox.py elevenlabs.py openai_tts.py
   providers/sep/__init__.py demucs.py
   ui/__init__.py server.py auth.py sse.py jobs.py
   ui/api/__init__.py
   ui/static/index.html ui/static/app.css   (placeholder shell; JS lands in issues 37/38)
   ```
3. `src/dubstudio/__init__.py` exposes `__version__` read from package metadata
   (`importlib.metadata`, fallback `"0.0.0"` for source runs).
4. `cli/app.py`: Typer root app named `dubstudio`; `main()` callable; subcommand
   `version` printing `dubstudio <__version__>`; `python -m dubstudio` works via
   `__main__.py`.
5. Tooling:
   - ruff: lint + format, line length 100, `select = ["E","W","F","I","UP","B","S"]`
     (S = bandit rules, W covers trailing-whitespace/EOF-newline) with `S` relaxed in
     `tests/`.
   - mypy: `strict = true` for `dubstudio.core`, `dubstudio.model`, `dubstudio.engine`,
     `dubstudio.providers.base`; normal elsewhere; plugin `pydantic.mypy`.
   - pytest: `testpaths = ["tests"]`, `addopts = "-q -m 'not live'"` (single exact
     string — `live` tests are deselected by default and re-enabled by an explicit
     CLI `-m live`, which overrides the addopts expression); marker declarations:
     `media` (needs ffmpeg), `live` (needs API keys / local models).
   - `.pre-commit-config.yaml`: a single `repo: local` block so hook versions are
     locked by `uv.lock` (supply-chain rule §11.2 B7, no floating `rev`):
     `ruff-check` (`uv run ruff check --force-exclude`) and `ruff-format`
     (`uv run ruff format --check --force-exclude`), both `language: system`,
     `types: [python]`.
6. `tests/test_scaffold.py`: imports `dubstudio`, asserts `__version__` non-empty, and
   runs the Typer CLI `version` via `typer.testing.CliRunner` asserting exit 0.
7. Commit `uv.lock`.

## Acceptance Criteria

- [ ] `uv sync --all-extras --dev` succeeds on macOS arm64 and Linux x86_64.
- [ ] `uv run dubstudio version` and `uv run python -m dubstudio version` print the
      version and exit 0.
- [ ] `uv run ruff check .`, `uv run ruff format --check .`, `uv run mypy src` all pass.
- [ ] `uv run pytest` passes.
- [ ] Package tree matches DESIGN.md §3.2 (no missing/extra top-level modules).
- [ ] No file imports torch or any optional-extra package at core import time.

## Validation

Run the AC commands verbatim in CI-like clean env (`uv sync` from scratch). Verify
`uv pip show dubstudio` metadata (license, python requirement).

## Dependencies

None.

## Non-goals

CI workflows, README content, any pipeline/provider/UI logic, packaging release config
(issue 42 owns publish workflow).

## Design References

DESIGN.md §3.2, §3.3, §14; ADR-001.
