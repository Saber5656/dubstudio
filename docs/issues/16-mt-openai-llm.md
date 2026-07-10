# Issue 16: Translation provider — OpenAI-compatible LLM

## Title

Implement length-aware dubbing translation provider over OpenAI-compatible chat API

## Summary

Implement `providers/mt/openai_llm.py`: windowed, char-budget-aware, spoken-register
segment translation through any OpenAI-compatible Chat Completions endpoint (OpenAI,
Ollama, LM Studio via `base_url`), returning strict JSON validated per segment.

## Context

The only cloud-or-local translation path in v1 (DESIGN.md §5.4, §7.5). Prompt content is
part of the fingerprint, so it lives in-repo and is versioned.

## Scope

In: provider, prompt templates, JSON-strict parsing/repair, tests. Out: the translate
stage's edit-preservation logic (issue 24 — stage decides *which* segments to send).

## Detailed Requirements

1. Config `[translation.openai]`: `model` (required; no default hardcoding of a dated
   model — `init` template suggests one in a comment), `base_url` (None → OpenAI),
   `max_window_segments=20`, `temperature=0.3`. Key via `resolve_api_key("openai")`;
   **when `base_url` points to localhost, a missing key is allowed** (local server).
2. Prompt files `providers/mt/prompts/dubbing_v1.md` (system) — requirements from
   DESIGN.md §5.4: spoken register suitable for dubbing; preserve meaning, no
   additions/omissions; per-segment char budget as soft limit; keep numbers, units,
   proper nouns; return **only** JSON `{"segments": [{"id": "...", "text": "..."}]}`.
   `PROMPT_VERSION = "dubbing_v1"` exported; participates in `ProviderInfo.version`
   (→ fingerprint via issue 10).
3. Request construction per window (≤ max_window_segments): system prompt + user
   payload JSON containing source/target langs, rolling `context_summary` (last ≤ 400
   chars of previous window's translations, provider-maintained), segments
   `[{id, text, char_budget}]`. Use `response_format={"type":"json_object"}` when the
   endpoint supports it; otherwise rely on prompt + parser.
4. Response handling: parse JSON; validate id set equality with the request window,
   non-empty texts; on violation retry **once** with an appended corrective user
   message; second failure → `ProviderInvalidResponse DS-PROVIDER-007` naming the
   window's first/last ids.
5. Over-budget outputs are accepted (soft limit) but flagged in
   `TranslateResult.overruns` (id → chars over) for stage warnings.
6. Retry/backoff via issue 13 policy for 429/5xx; 401 → auth error.
7. `estimate_cost`: (chars_in + estimated chars_out) → token estimate (chars/4)
   × price table for the configured model; unknown model → None (planner shows
   "unknown").
8. `healthcheck()`: GET `/models` (or `base_url` root for non-OpenAI), 10 s timeout.

## Acceptance Criteria

- [ ] respx tests: window of 20 segments builds one request with exact payload schema
      (golden JSON); ids round-trip; budget passed through.
- [ ] Malformed JSON then valid JSON on retry → succeeds; twice-malformed → DS-PROVIDER-007.
- [ ] Missing id / extra id / empty text each trigger the corrective retry.
- [ ] `base_url=http://127.0.0.1:11434/v1` with no key does not raise auth error.
- [ ] Overruns reported for texts exceeding budget.
- [ ] Prompt file hash change changes `ProviderInfo.version` (fingerprint test).

## Validation

`uv run pytest tests/providers/mt/test_openai_llm.py` (HTTP fully mocked); manual live
check ja→en on the fixture transcript documented in PR.

## Dependencies

13.

## Non-goals

Glossaries (v2), DeepL (v2), auto model selection, translation memory.

## Design References

DESIGN.md §5.4, §7.5, §7.6, §11.2 B2.
