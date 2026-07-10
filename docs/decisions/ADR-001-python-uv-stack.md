# ADR-001: Python 3.12+ with uv, Typer, FastAPI

- Status: accepted
- Date: 2026-07-10
- Decision drivers: user selection (2026-07-10 requirements interview); implementability
  by lower-capability agents; audio/ML ecosystem fit.

## Context

dubstudio needs local ML inference (faster-whisper, demucs, Chatterbox — all
Python/PyTorch), cloud SDK access, a CLI, and a thin local web UI. Implementation will be
executed by lower-capability agents from granular issues, so the stack must be mainstream
and boring.

## Decision

Python ≥ 3.12, uv-managed (committed `uv.lock`), src layout. CLI: Typer. Validation and
schemas: Pydantic v2 (+pydantic-settings for config). HTTP client: httpx. Web UI: FastAPI
+ uvicorn serving no-build static frontend. Console output: rich.

## Consequences

- Local model extras (`[local-asr]`, `[separate]`, `[local-tts]`) isolate heavy torch
  dependencies from the light core install.
- Single language across engine and UI backend; no Node toolchain in the repo.
- Distribution via PyPI / `uvx dubstudio` (DESIGN.md §14).

## Alternatives considered

- **TypeScript/Node**: better frontend DX, but local audio ML bindings are weak; native
  dependency management for whisper.cpp-class libs raises implementation risk.
- **Rust core**: performance is irrelevant (bottleneck is model inference and ffmpeg);
  development cost and agent difficulty are much higher.
