# Issue 06: Configuration system

## Title

Implement layered TOML configuration with env overrides and secret policy enforcement

## Summary

Implement `core/config.py`: pydantic-settings-based schema for all `[tables]` in
DESIGN.md §8.2, four-layer precedence merge (defaults → user TOML → project TOML → env →
CLI), `.env` loading, unknown-key rejection with suggestions, and rejection of secret
material in TOML files.

## Context

Config participates in stage fingerprints (issue 10) and holds every tunable named in
DESIGN.md §5; its shape must be complete and typo-safe before stages land.

## Scope

In: schema, loader/merger, secret policy, `Settings.for_stage(name)` config-subset
accessor. Out: CLI `config show` rendering (issue 31), per-provider param semantics
(provider issues validate their own sub-models).

## Detailed Requirements

1. Pydantic models (frozen) for: `project` (input: Path, source_language: str|"auto",
   target_languages: list[str] min 1, audio_stream: int|None), `providers` (asr,
   translation, tts, separation: str; enabled_plugins: list[str] = []),
   `asr.faster_whisper` (model="large-v3-turbo", device="auto", compute_type="auto"),
   `asr.openai` (model="gpt-4o-mini-transcribe", base_url: str|None = None,
   timestamps: Literal["auto","segment","word"] = "auto"), `translation.openai` (model: str,
   base_url: str|None, max_window_segments=20, temperature=0.3),
   `tts.elevenlabs` (model="eleven_multilingual_v2", stability/similarity floats),
   `tts.chatterbox` (model="multilingual", device="auto", exaggeration=0.5,
   cfg_weight=0.5),
   `tts.openai` (model="gpt-4o-mini-tts", voice="alloy"),
   `tts` root (allow_non_cloned_voice=False),
   `translation` root (cps: dict[str, float] = {} — per-language chars-per-second
   overrides consumed by issue 24),
   `separation` (enabled=True, model="htdemucs", device="auto", segment_s=0),
   `transcribe` (no_speech_prob_max=0.85,
   merge_max_gap_ms=200, merge_min_dur_ms=600, split_max_dur_ms=12000),
   `voice_ref` (target_seconds=60, min_seconds=20),
   `fit` (atempo_min=0.9, atempo_max=1.15, max_bleed_ms=1500, auto_shorten=True),
   `mix` (background_gain_db=-3.0, loudness_lufs=-16.0, true_peak_db=-1.5),
   `export` (keep_original_audio=True, embed_subtitles=False, container: str|None),
   `subtitles` (max_chars_per_line=42, cjk_max_chars_per_line=21, max_lines=2),
   `limits` (max_duration_min=90, max_input_gb=8),
   `cost` (confirm_over_usd=5.0, tables: dict = {}),
   `synthesize` (concurrency=2), `ui` (port=0), `logging` (verbosity=0).
   Language codes: normalize (primary subtag lowercased, region uppercased) then
   validate against `^[a-z]{2,3}(-[A-Z]{2})?$` — `ja`, `en`, `en-US`, `pt-BR` valid
   (region preserved after normalization); `EN` → `en`; `english`, `en_us`,
   `zh-Hans` invalid in v1 with a did-you-mean-style error.
   Note: the reference-voice settings (`voice.mode`, `voice.user_ref_path`) live in
   the **manifest** (DESIGN §4.2), not in config — do not add them here.
2. Sources & precedence (DESIGN.md §8.1): built-in defaults → `platformdirs`
   user config `config.toml` → `<project>/dubstudio.toml` → env `DUBSTUDIO_*` (nested
   via `__`) → explicit overrides dict (CLI layer passes flags).
   `.env` handling (exact): before config parsing, load `<project>/.env` then
   `<platformdirs user config dir>/.env` (first-found value per key wins between the
   two); **never override variables already present in the process environment**;
   loading is plain `KEY=value` lines (no shell interpolation).
3. Unknown keys anywhere → `ConfigError` `DS-CONFIG-001` listing the key path and the
   closest valid key (difflib).
4. Secret policy: after parsing each TOML layer, walk raw keys; any key named
   `api_key|apikey|token|secret|password` (case-insensitive) → `ConfigError`
   `DS-CONFIG-003` with hint "set <ENV_NAME> in the environment or .env instead".
5. Schema-version gate: top-level `config_schema = 1` optional key; higher value →
   `DS-CONFIG-004`.
6. `Settings.for_stage(stage_name) -> dict`: returns the canonical JSON-able subset of
   config that stage declares (single table in this module — feeds fingerprints,
   issue 10). The exact v1 mapping (per-lang stages receive the base name):

   | stage | config subset (model dumps, stable key order) |
   |---|---|
   | ingest | `limits` + `project.audio_stream` |
   | separate | `separation` |
   | transcribe | `transcribe` + `project.source_language` |
   | translate | `translation` root (incl. `cps`) + `translation.openai` |
   | voice_ref | `voice_ref` |
   | synthesize | `tts` root + the selected provider's `tts.<name>` table + `synthesize` |
   | fit | `fit` |
   | mix | `mix` |
   | export | `export` |
   | subtitles | `subtitles` |

   Provider *selection* (`providers.*`) is deliberately excluded — provider identity
   enters fingerprints via the provider stamp (DESIGN §4.4), not the config subset.
7. API-key resolution helper `resolve_api_key(provider_name) -> SecretStr | None`
   reading `DUBSTUDIO_<PROVIDER>_API_KEY` then the provider's canonical env
   (`OPENAI_API_KEY`, `ELEVENLABS_API_KEY`).

## Acceptance Criteria

- [ ] Precedence test: same key set in all four layers resolves in documented order.
- [ ] `DUBSTUDIO_FIT__ATEMPO_MAX=1.2` overrides project TOML value.
- [ ] Unknown key `[fitt]` fails with suggestion `fit`.
- [ ] `api_key = "sk-…"` in any TOML fails with DS-CONFIG-003; the value does **not**
      appear in the error message (redaction interplay).
- [ ] `for_stage("fit")` returns exactly the `[fit]` model dump (stable key order);
      a table-driven test covers every row of the for_stage mapping above.
- [ ] `.env`: project value loads; user-dir value loads when project lacks the key;
      a pre-existing process env var is never overwritten by either.
- [ ] Defaults match every literal default in DESIGN.md §5/§8 (table-driven test).

## Validation

`uv run pytest tests/core/test_config.py`; mypy strict; round-trip: load → dump →
load equality.

## Dependencies

05.

## Non-goals

Keyring storage (v2), config editing commands, provider connectivity validation.

## Design References

DESIGN.md §8, §5 (defaults), §11.2 B6; ADR-004.
