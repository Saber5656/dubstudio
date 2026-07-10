# Issue 08: Domain models and JSON schemas

## Title

Implement manifest, transcript, translation, synthesis, and fit-report models

## Summary

Implement `model/*` pydantic v2 models exactly matching the schemas in DESIGN.md §4.2,
§4.3 and event records §6.5, with schema_version gates, unknown-field preservation, and
canonical JSON serialization.

## Context

These models are the contract between stages, store, CLI, and UI; every artifact file
on disk validates through them (trust boundary B6).

## Scope

In: models + (de)serialization + JSON-schema export + tests. Out: file IO/atomicity
(issue 09), staleness semantics (issue 10).

## Detailed Requirements

1. `model/manifest.py`: `Manifest` per DESIGN.md §4.2 — `schema_version: Literal[1]`,
   project_id (uuid4 str), created_at (UTC), `InputRef` (path, sha256, size_bytes,
   media: MediaSummary), `Languages` (source: str, targets: list[str]),
   `VoiceConfig` (mode: "auto"|"user", user_ref_path: str|None),
   `stages: dict[str, StageRecord]` where key matches
   `^(ingest|separate|transcribe|voice_ref)$` or
   `^(translate|synthesize|fit|mix|export|subtitles):[a-z]{2,3}(-[A-Za-z0-9]+)?$`
   (validator), `StageRecord` (status: StageStatus enum per §6.1, fingerprint: str|None,
   started_at/finished_at: datetime|None, error: ErrorRef|None
   {code, message}), `consent_snapshot: ConsentSnapshot|None` {policy_version:int,
   accepted_at:datetime}.
2. `model/segments.py`: `Transcript` per §4.3 (language, provider: ProviderStamp
   {name, model}, segments: list[Segment]); `Segment` (id: `^seg_\d{4}$` validated,
   start_ms/end_ms ints with end>start, text non-empty str, words: list[Word]|None
   {w, start_ms, end_ms}, asr: AsrStats|None {avg_logprob, no_speech_prob}).
   Ordering validator: segments sorted by start_ms, ids strictly increasing.
3. `model/translation.py`: `TranslationDoc` per §4.3 — source/target language,
   provider stamp, segments: list[TranslatedSegment] (id, source_text, text,
   status: Literal["draft","edited","approved"], char_budget: int, notes: str|None).
4. `model/synthesis.py`: `SynthDoc` — header {provider stamp incl. params_hash,
   voice: {voice_hash, provider_voice_id: str|None,
   kind: Literal["cloned","preset"]}}, segments:
   list[SynthSegment] (id, audio: relative POSIX path str, duration_ms, text_hash,
   created_at). (Header-form provider/voice dedup matches DESIGN.md §4.3.)
5. `model/fitreport.py`: `FitReport` — segments: list[FitEntry] (id, slot_ms,
   available_ms, synth_ms, atempo: float, pad_ms, overrun_ms,
   result: Literal["ok","shortened","warn_overflow"], audio: relative path).
6. `model/events.py`: `RunEvent` per §6.5 (ts, run_id,
   level: Literal["debug","info","warning","error"], event enum, stage|None,
   lang|None, segment_id|None, message, data: dict|None).
7. Common behaviors:
   - `model_config = ConfigDict(extra="allow")` + round-trip test proving unknown fields
     survive load→dump (forward compat, §4.2).
   - `schema_version` int field on every *document* model; loader helper
     `parse_document(model_cls, data)` raising `ConfigError DS-CONFIG-004` when
     `schema_version` > supported.
   - `to_canonical_json(model) -> str`: UTF-8, 2-space indent, sorted keys **off**
     (declaration order), trailing newline — the byte format issue 09 writes.
   - Relative-path fields validated **syntactically** here: POSIX separators only
     (backslash rejected), no `..` component, no leading `/`, no empty components.
     Semantic containment (resolve + `is_relative_to(project_root)`) is the store's
     job (issue 09) at every path consumption — this split is deliberate (§11.2 B6);
     document it in the module docstring.
8. Export JSON Schemas for Transcript/TranslationDoc/Manifest to
   `docs/schemas/*.schema.json` via a `scripts/export_schemas.py` (checked-in output;
   test asserts freshness).

## Acceptance Criteria

- [ ] Every example JSON block in DESIGN.md §4.2/§4.3 parses through its model
      unchanged (fixtures copied verbatim into tests).
- [ ] Stage-key regex accepts `translate:en`, rejects `translate:EN` and `translate:`.
- [ ] Unknown-field round-trip preserved; schema_version=2 rejected with DS-CONFIG-004.
- [ ] Path validator rejects `../x.wav`, `/abs/x.wav`, `a\\b.wav`, `a/../b.wav`,
      and `a//b.wav`; accepts `segments/seg_0001.wav`.
- [ ] `docs/schemas/` outputs committed and up to date (CI test).

## Validation

`uv run pytest tests/model/`; mypy strict; schema-freshness test.

## Dependencies

05.

## Non-goals

Migrations (identity stubs only), speaker fields (ADR-006), file IO.

## Design References

DESIGN.md §4.2, §4.3, §4.6, §6.5, §11.2 B6.
