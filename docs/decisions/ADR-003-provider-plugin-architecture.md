# ADR-003: Provider abstraction via Protocols + entry points with explicit activation

- Status: accepted
- Date: 2026-07-10
- Decision drivers: user selection ("abstract and support both API and local");
  security posture for a public OSS tool.

## Context

Every AI step (ASR, translation, TTS, separation) must support both cloud APIs and local
models, and the model landscape shifts quickly (see
`docs/research/2026-07-audio-model-landscape.md` — e.g. OpenAI's 2026 releases are
realtime-focused and unusable for batch dubbing today, but that may change).

## Decision

Typed `Protocol` interfaces per slot with explicit capability objects
(`supports_cloning`, `word_timestamps`, `watermark_builtin`, language sets) checked at
plan time (DESIGN.md §7). Built-in providers are statically registered. Third-party
providers are discovered via the `dubstudio.providers` entry-point group but are **not
activated unless explicitly named** in `providers.enabled_plugins` — installing a package
must not silently add code to the pipeline (trust boundary B5, DESIGN.md §11.2).

## Consequences

- New models (e.g. a future OpenAI cloning API, Qwen3-TTS) are additive provider issues,
  not core changes.
- Capability checks move failures to plan time instead of mid-pipeline.
- Slightly more boilerplate per provider (info, caps, cost estimate, healthcheck).

## Alternatives considered

- **Auto-activated entry points** (pytest-style): convenient but lets any installed
  package inject code into a tool that handles API keys and user media — rejected.
- **Subprocess-isolated plugins**: stronger isolation, disproportionate complexity for v1.
