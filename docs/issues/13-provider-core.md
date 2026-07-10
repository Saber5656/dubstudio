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

1. `base.py`: the four Protocols verbatim from DESIGN.md §7.1 with fully specified
   pydantic models (all frozen; language codes pre-normalized per issue 06 rule):
   - `ProviderInfo`: `name: str`, `kind: Literal["asr","translation","tts",
     "separation"]`, `mode: Literal["local","cloud"]`, `version: str`.
   - `AsrCaps`: `word_timestamps: bool`, `languages: set[str] | Literal["*"]`.
   - `TtsCaps`: `supports_cloning: bool`, `languages: set[str] | Literal["*"]`,
     `watermark_builtin: bool`, `max_chars_per_request: int (> 0)`.
   - `RawTranscript`: `language: str`, `language_confidence: float | None (0..1)`,
     `segments: list[RawSegment]`; `RawSegment`: `start_ms: int`, `end_ms: int`,
     `text: str`, `words: list[RawWord] | None` (`RawWord`: `w, start_ms, end_ms`),
     `avg_logprob: float | None`, `no_speech_prob: float | None`. (Raw = pre
     post-processing; may contain overlaps/empties that issue 23 cleans.)
   - `SegmentIn`: `id: str`, `text: str`, `char_budget: int (≥ 1)`.
   - `StyleHints`: `register: Literal["spoken"] = "spoken"` (v1 fixed; extension
     point), `extra: dict[str, str] = {}`.
   - `TranslateRequest`: `segments: list[SegmentIn] (non-empty)`, `source_lang: str`,
     `target_lang: str`, `context_summary: str = ""` (≤ 400 chars),
     `style: StyleHints`.
   - `TranslateResult`: `texts: dict[str, str]` (id → translation; id set must equal
     the request's — validator), `overruns: dict[str, int] = {}` (id → chars over
     budget).
   - `VoiceReference`: `path: Path` (existing file), `voice_hash: str`.
   - `VoiceHandle`: `voice_hash: str`, `kind: Literal["cloned","preset"]`,
     `provider_voice_id: str | None` (cloud voice id), `local_ref: Path | None`
     (exactly one of provider_voice_id/local_ref set for cloned; both None for
     preset — validator).
   - `SynthAudio`: `path: Path` (WAV written by the provider into the dir the stage
     passed) OR `wav_bytes: bytes` (exclusive — validator), `duration_ms: int`
     measured by the provider from actual audio.
   - `SeparationResult`: `vocals: Path`, `background: Path`.
   - `TtsParams`: frozen pydantic model with `extra="allow"` — the selected TTS
     provider's config-table dump (issue 06) passed through verbatim; each provider
     documents the keys it reads (e.g. chatterbox: exaggeration/cfg_weight;
     elevenlabs: stability/similarity_boost/style) and ignores unknown keys. Its
     canonical JSON participates in the synth cache key (issue 26).
   - `HealthReport`: `ok: bool`, `detail: str`, `checked: list[str]`.
   - `Progress = Callable[[float, str], None]` (fraction 0.0–1.0 monotonic, short
     message).
   `cleanup_voice(handle)` is a **required** TtsProvider method (DESIGN §7.1);
   providers with no remote state implement a documented no-op. Optional protocol
   methods (detected via `hasattr`): `estimate_cost(work: CostWork) ->
   CostEstimate | None`, `healthcheck() -> HealthReport`.
2. `errors.py`: **re-exports** the provider exception classes that issue 05 defines
   and catalogs in `core/errors.py` (single source of truth for classes/codes/exit
   codes — no duplicate class definitions here). The exact class↔code map (as
   catalogued by issue 05): `ProviderAuthError`=DS-PROVIDER-002 (exit 6),
   `ProviderQuotaError`=DS-PROVIDER-003 (exit 7, carries `retry_after_s: float |
   None`), `ProviderRemoteError`=DS-PROVIDER-004 (5xx, exit 8),
   `ProviderNotInstalled`=DS-PROVIDER-005 (exit 12), `ProviderUnsupported`=
   DS-PROVIDER-006 (exit 8), `ProviderInvalidResponse`=DS-PROVIDER-007 (exit 8),
   `PluginNotEnabled`=DS-PROVIDER-008 (exit 4), `RequestTooLarge`=DS-PROVIDER-009
   (exit 8), `ResourceExhausted`=DS-PROVIDER-010 (exit 8).
   This module adds the provider-layer helper only:
   `retry_policy(fn)` — max 5 attempts, exp backoff ×2 with full jitter, cap
   60 s, honors `retry_after_s`; only `ProviderQuotaError` and
   `ProviderRemoteError` retry.
3. `registry.py`:
   - static `BUILTINS: dict[str, factory]` seeded by issues 14–20 (this issue registers
     only mocks);
   - `discover_plugins() -> list[PluginInfo]` reading entry-point group
     `dubstudio.providers` **without importing** — `PluginInfo`: `name` (the entry
     point name = provider name), `dist_name`, `dist_version`, `ep_value`
     (`pkg.module:factory`). Kind/mode/capabilities are *unknown until import*:
     `list_all` renders disabled plugins as `plugin (not loaded)` rows with only
     these metadata fields. Factory contract once enabled: `factory(config: dict) ->
     provider instance` (config = that provider's config table). Name collisions:
     plugin shadowing a builtin is ignored with a WARNING; two plugins with the same
     name → keep the first by sorted dist_name, WARNING for the rest;
   - `get(kind, name, config) -> Provider`: builtin first; else if name is a discovered
     plugin AND `name in config.providers.enabled_plugins` → import + instantiate;
     else `ProviderNotInstalled DS-PROVIDER-005` (unknown) or `DS-PROVIDER-008`
     "plugin present but not enabled; add to providers.enabled_plugins" — never
     auto-import;
   - `list_all(config)` → rows for CLI/UI: name, kind, mode, builtin/plugin,
     enabled, key-env set?, extra installed?, watermark_builtin (tts).
4. `cost.py`:
   - `CostWork` variants (each carries the estimator identity):
     `AsrWork(provider: str, seconds: float)`, `MtWork(provider: str,
     chars_in: int, chars_out_est: int)`, `TtsWork(provider: str, chars: int)`.
   - Estimation helper providers call:
     `line(table, *, provider, price_key, qty, unit_label) -> CostEstimate | None`
     — returns None (with the breakdown line `"<provider>: unknown pricing"`
     handled by `aggregate`) when `price_key` is absent.
   - Price table: flat dict keyed `"<provider>.<unit>"` with USD-per-unit floats,
     module constant `DEFAULT_PRICES` with a dated "approximate, update freely"
     comment (seed keys: `openai-asr.audio_min`, `openai.tok1k_in`,
     `openai.tok1k_out` (tokens estimated as chars/4), `elevenlabs.char`,
     `openai-tts.char`); `[cost.tables]` config entries override by identical key
     (unknown keys allowed — forward compat).
   - `CostEstimate`: `usd: float`, `breakdown: list[str]`, `approximate: bool =
     True`. Breakdown line format (exact): `"<provider>: <qty> <unit> ≈ $<usd>"`
     with qty rendered `%g` and usd `%.2f`.
   - `aggregate(estimates: Iterable[CostEstimate | None]) -> CostEstimate | None`:
     sums usd (float), concatenates breakdowns, returns None only when every input
     is None; missing price key → that work contributes None + a breakdown line
     `"<provider>: unknown pricing"`.
5. `mock.py` (registered as builtins `mock-asr`, `mock-mt`, `mock-tts`, `mock-sep`):
   deterministic, dependency-free — MockAsr returns the fixture transcript pattern
   (sentence per 3 s, words evenly spaced); MockMt uppercases text and prefixes
   `[lang] `, respecting char_budget by truncation w/ ellipsis; MockTts writes silent
   WAV of `len(text) × 55 ms` (deterministic duration for fit tests),
   `supports_cloning=True`, `watermark_builtin=False`; MockSep splits source into two
   copies at −6 dB. All record their call args onto `self.calls` for assertions.
6. Language normalization helper `normalize_lang` shared by providers — identical
   semantics to issue 06's rule (primary subtag lowercased, region **uppercased**,
   same validation regex; `en-us` → `en-US`); the two modules share one
   implementation (config imports it from here or vice versa — single definition).
7. `list_all(config)` row schema matches issue 31's JSON convention: for discovered
   but disabled plugins, `kind/mode/watermark_builtin/languages` are `null`
   ("unknown until import") and `origin="plugin", enabled=false`.

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

`uv run pytest tests/providers/test_base.py tests/providers/test_registry.py
tests/providers/test_cost.py tests/providers/test_mock.py`.

## Dependencies

06.

## Non-goals

Real providers, subprocess-isolated plugins (ADR-003 alternative), pricing accuracy
guarantees (labeled approximate).

## Design References

DESIGN.md §7.1–7.4, §7.6, §11.2 B5; ADR-003.
