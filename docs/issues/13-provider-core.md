# Issue 13: Provider core

## Title

Implement provider protocols, capabilities, registry with explicit plugin activation,
errors, cost estimation, and mock providers

## Summary

Implement `providers/base.py`, `registry.py`, `errors.py`, `cost.py`, and `mock.py`
exactly per DESIGN.md §7.1–7.4/§7.6 and ADR-003, including the security-relevant rule
that third-party entry-point providers load only when explicitly enabled.

## Context

All seven built-in providers (issues 14–20) and the engine's plan-time capability/cost
logic depend on these contracts being final.

## Scope

In: the five modules + retry helper + tests. Out: any real provider implementation.

## Detailed Requirements

1. `base.py`: the four Protocols verbatim from DESIGN.md §7.1 with typed models:
   `ProviderInfo` (name, kind: asr|translation|tts|separation, mode: local|cloud,
   version), `AsrCaps` (word_timestamps: bool, languages: set[str] | Literal["*"]),
   `TtsCaps` (supports_cloning, languages, watermark_builtin, max_chars_per_request),
   `RawTranscript`/`TranslateRequest`/`TranslateResult`/`VoiceReference`
   (`path`, `voice_hash`)/`VoiceHandle` (provider_voice_id|local handle, voice_hash)/
   `SynthAudio` (wav_bytes|path, duration_ms)/`SeparationResult`,
   `Progress = Callable[[float, str], None]`,
   optional protocol methods `estimate_cost(work: CostWork) -> CostEstimate | None`,
   `healthcheck() -> HealthReport` (default impls via runtime `hasattr`).
2. `errors.py`: exception classes per DESIGN.md §7.4 mapping onto issue 05 exit codes;
   `retry_policy(fn)` helper — max 5 attempts, exp backoff ×2 with full jitter, cap
   60 s, honors `retry_after_s` attr on `ProviderQuotaError`; only
   `ProviderQuotaError` and `ProviderRemoteError` retry.
3. `registry.py`:
   - static `BUILTINS: dict[str, factory]` seeded by issues 14–20 (this issue registers
     only mocks);
   - `discover_plugins() -> list[PluginInfo]` reading entry-point group
     `dubstudio.providers` **without importing** (metadata only);
   - `get(kind, name, config) -> Provider`: builtin first; else if name is a discovered
     plugin AND `name in config.providers.enabled_plugins` → import + instantiate;
     else `ProviderNotInstalled DS-PROVIDER-005` (unknown) or `DS-PROVIDER-008`
     "plugin present but not enabled; add to providers.enabled_plugins" — never
     auto-import;
   - `list_all(config)` → rows for CLI/UI: name, kind, mode, builtin/plugin,
     enabled, key-env set?, extra installed?, watermark_builtin (tts).
4. `cost.py`: `CostWork` variants (asr_seconds, mt_chars_in/out, tts_chars,
   none); built-in USD price table constants (approximate, dated comment) overridable
   via `[cost.tables]`; `CostEstimate` (usd: float, breakdown: list[str],
   approximate=True); aggregation helper for the planner.
5. `mock.py` (registered as builtins `mock-asr`, `mock-mt`, `mock-tts`, `mock-sep`):
   deterministic, dependency-free — MockAsr returns the fixture transcript pattern
   (sentence per 3 s, words evenly spaced); MockMt uppercases text and prefixes
   `[lang] `, respecting char_budget by truncation w/ ellipsis; MockTts writes silent
   WAV of `len(text) × 55 ms` (deterministic duration for fit tests),
   `supports_cloning=True`, `watermark_builtin=False`; MockSep splits source into two
   copies at −6 dB. All record their call args onto `self.calls` for assertions.
6. Language normalization helper `normalize_lang("ja")` shared by providers
   (lowercase primary subtag; region preserved).

## Acceptance Criteria

- [ ] mypy strict passes with the Protocols; a static test asserts each mock satisfies
      its Protocol via assignment to the Protocol type.
- [ ] Entry-point plugin fixture package (created inside the test via
      `importlib.metadata` fake) is discovered but NOT imported until enabled — import
      side-effect sentinel proves non-activation; enabling it activates.
- [ ] Retry helper: quota error with `retry_after_s=0.01` retries ≤ 5 then raises;
      auth error does not retry.
- [ ] Cost aggregation sums mixed works and formats a breakdown.
- [ ] Mock determinism: same inputs → byte-identical outputs (hash test).

## Validation

`uv run pytest tests/providers/test_base.py test_registry.py test_cost.py test_mock.py`.

## Dependencies

06.

## Non-goals

Real providers, subprocess-isolated plugins (ADR-003 alternative), pricing accuracy
guarantees (labeled approximate).

## Design References

DESIGN.md §7.1–7.4, §7.6, §11.2 B5; ADR-003.
