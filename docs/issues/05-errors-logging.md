# Issue 05: Error taxonomy and logging with secret redaction

## Title

Implement core error taxonomy, exit-code mapping, and redacting console/JSONL logging

## Summary

Implement `core/errors.py` (typed error hierarchy with codes, hints, exit codes) and
`core/logging.py` (rich console + JSONL file logging with a global secret-redaction
filter) exactly per DESIGN.md §12, §9 (exit codes), §6.5 (event/log record shape).

## Context

Every subsequent module raises these errors and logs through this layer; the redaction
filter is a security acceptance criterion (§11.2 B2).

## Scope

In: the two modules + tests. Out: stage event *semantics* (issue 10 emits events through
this layer), CLI rendering of errors (issues 31–33).

## Detailed Requirements

1. `DubstudioError(Exception)` fields: `code: str` (format `DS-<AREA>-<NNN>`),
   `message: str` (one-line summary), `detail: str | None` (multi-line cause detail —
   e.g. stderr tail, offending values — passed through the redaction filter),
   `hint: str | None`, `area: ErrorArea` enum
   (`CONFIG, MEDIA, PROVIDER, STAGE, CONSENT, COST, LOCK, UI`), `exit_code: int`.
2. Subclasses with fixed exit codes per DESIGN.md §9: `UsageError`→2,
   `ConsentError`→3, `ConfigError`→4, `MediaError`→5, `ProviderAuthError`→6,
   `ProviderQuotaError`→7, `ProviderError`→8, `StageError`→9, `CancelledError`→10,
   `CostConfirmationRequired`→11, `EnvironmentError`→12 (name it `EnvMissingError` to
   avoid builtin shadowing), `LockHeldError`→13.
3. Registry: module-level `ERROR_CATALOG: dict[str, ErrorSpec]` seeded with the full
   code set used across the v1 issue plan (message + default hint each):
   - CONFIG: 001 unknown config key, 002 invalid target dir / not a project,
     003 secret material in TOML, 004 unsupported schema version, 005 non-cloning TTS
     without `tts.allow_non_cloned_voice`, 006 segments-import validation failure;
   - MEDIA: 000 ffmpeg/ffprobe missing, 001 unreadable/corrupt media, 002 input hash
     mismatch, 003 missing/invalid audio stream, 004 over size/duration limits,
     005 mux container/codec incompatibility;
   - STAGE: 001 prerequisite stage not completed, 002 no speech detected, 003 voice
     reference unusable/insufficient, 004 output duration drift, 005 subprocess
     timeout, 006 corrupted artifact JSON, 007 segment text exceeds provider limit,
     008 partial per-segment synthesis failures, 009 approval gate failed;
   - PROVIDER: 002 auth, 003 quota/rate limit, 004 remote provider server error
     (5xx), 005 provider/extra not installed, 006 capability/language unsupported,
     007 invalid provider response, 008 plugin discovered but not enabled,
     009 request exceeds provider limit, 010 local resource exhaustion (OOM);
   - CONSENT: 001 consent missing/declined; COST: 001 confirmation required;
   - LOCK: 001 lock held by another process, 002 manifest mutation attempted without
     holding the project lock; UI: 001 job already active.
   The catalog is **append-only**: any future issue introducing a new code must
   register it here (rule stated in the module docstring). `ErrorSpec` carries
   `{code, area, exception_class, message, hint}` — the exception class fixes the
   exit code, and the mapping is explicit per code (notably **DS-MEDIA-000 →
   `EnvMissingError`, exit 12**, not MediaError/5). Constructing a catalogued error by
   code (`DubstudioError.from_code`) instantiates that class with defaults; unknown
   codes raise (no typo'd codes), enforced by an import-time test.
4. `core/logging.py`:
   - `configure_logging(verbosity: int, jsonl_path: Path | None)`.
   - Console: rich handler, INFO default, DEBUG at `-v`.
   - JSONL handler (always DEBUG when path set): one JSON object/line. Two record
     kinds share the §6.5 field names: (a) engine `RunEvent`s serialized verbatim
     (`ts, run_id, level, event, stage, lang, segment_id?, message, data?`); (b)
     generic log records rendered as `{ts, level, event: "log", message, data?}` —
     `run_id/stage/lang/segment_id` are simply omitted when absent, and any
     structured logging extras go under `data` (never as ad-hoc top-level keys).
   - `RedactionFilter` applied to **all** handlers: at configure time, snapshot values
     of env vars whose names match `re.fullmatch(r".*_(API_KEY|TOKEN|SECRET)")` (len ≥ 8);
     replace exact occurrences in rendered messages/args with `***`; additionally mask
     `(?i)(authorization:\s*bearer\s+)\S+` and any `sk-[A-Za-z0-9_-]{8,}`-shaped token.
   - Redaction also wraps exception formatting (tracebacks pass through the filter).
5. `render_error(err) -> str` helper producing the §12 layout: `error DS-…: <message>`
   + optional indented `detail` block (redacted) + optional `hint: …` + docs anchor
   line (anchor = lowercase code).

## Acceptance Criteria

- [ ] Every exit code in DESIGN.md §9 has exactly one error class; test asserts the full
      mapping table.
- [ ] A log record containing the literal value of `ELEVENLABS_API_KEY` (set to a test
      value) is emitted with `***` in console and JSONL outputs.
- [ ] Tracebacks containing a bearer token are redacted.
- [ ] JSONL lines parse as JSON and contain `ts/level/message`.
- [ ] Constructing `DubstudioError.from_code("DS-MEDIA-001")` yields catalogued
      message/hint/exit code.

## Validation

`uv run pytest tests/core/test_errors.py tests/core/test_logging.py` — includes the
negative redaction tests above. mypy strict clean.

## Dependencies

01.

## Non-goals

Sentry/telemetry (never), i18n of messages, CLI presentation.

## Design References

DESIGN.md §12, §9 (exit codes), §6.5, §11.2 B2.
