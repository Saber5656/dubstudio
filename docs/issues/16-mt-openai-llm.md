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
3. Request construction per window: the provider is **stateless** — it uses
   `TranslateRequest.context_summary` exactly as supplied by the translate stage
   (issue 24 builds it deterministically; the provider never maintains rolling
   state). Exact HTTP contract: `POST {base_url or https://api.openai.com/v1}
   /chat/completions`, JSON body `{model, temperature, messages: [{role:"system",
   content:<prompt file>}, {role:"user", content:<payload JSON: source_lang,
   target_lang, context_summary, segments [{id, text, char_budget}]>}]}` +
   `response_format={"type":"json_object"}` when the endpoint supports it (config
   `json_mode: auto|on|off`, default auto = on for api.openai.com, off otherwise);
   `Authorization: Bearer <key>` header **only when a key is resolved**; request
   timeout 120 s; TLS verification always on (no config to disable — §11.2 B2).
4. Response handling: strip a single Markdown code fence if present, then extract the
   first top-level JSON object (that is the **only** local repair permitted); parse;
   validate id set equality with the request window, non-empty texts; on violation
   retry **once** with an appended corrective user message; second failure →
   `ProviderInvalidResponse DS-PROVIDER-007` naming the window's first/last ids.
5. Over-budget outputs are accepted (soft limit) but flagged in
   `TranslateResult.overruns` (id → chars over) for stage warnings.
6. Retry/backoff via issue 13 policy for 429/5xx; 401 → auth error.
7. `estimate_cost`: (chars_in + estimated chars_out) → token estimate (chars/4)
   × price table for the configured model; unknown model → None (planner shows
   "unknown").
8. `healthcheck()`: GET `/models` (or `base_url` root for non-OpenAI), 10 s timeout.
9. Privacy & secrecy (§11.2 B2): provider docstring + user-docs stub state that
   source-segment texts and the context block are sent to the configured endpoint
   (and to OpenAI by default); API key never appears in logs/exceptions (test with a
   planted fake key); TLS verify is not configurable.

## Acceptance Criteria

- [ ] respx tests: window of 20 segments builds one request with exact payload schema
      (golden JSON); ids round-trip; budget passed through.
- [ ] Malformed JSON then valid JSON on retry → succeeds; twice-malformed → DS-PROVIDER-007.
- [ ] Missing id / extra id / empty text each trigger the corrective retry.
- [ ] `base_url=http://127.0.0.1:11434/v1` with no key does not raise auth error.
- [ ] Overruns reported for texts exceeding budget.
- [ ] Code-fenced JSON response parses (repair path); prose-wrapped JSON beyond one
      object → corrective retry.
- [ ] Fake `OPENAI_API_KEY` never appears in logs/exceptions (grep test); request to
      api.openai.com carries Bearer header exactly once.
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
