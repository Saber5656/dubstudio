# Issue 38: UI frontend — pipeline panel

## Title

Implement the pipeline control panel with live SSE progress

## Summary

Implement the second frontend view per DESIGN.md §10.4: stage cards with statuses,
run controls (full/until/only per language), live progress and warnings via SSE,
cost-estimate acknowledgment flow, and links to outputs/fit report.

## Context

Pairs with the segments table (37); shares `api.js` and introduces the SSE-driven
`jobs.js` store both views consume.

## Scope

In: `static/pipeline.js`, `static/jobs.js` (SSE client + shared job/state store),
panel styles. Out: server SSE (36), segments view (37).

## Detailed Requirements

1. `jobs.js`: `EventSource('/api/events')` with auto-reconnect (backoff 1→15 s);
   normalizes RunEvents into a store `{jobActive, stages: {key: {status, progress,
   message}}, warnings: [...], lastSummary}`; broadcasts via `EventTarget`; exposes
   `startRun(filters, {acknowledgeCost})`, `cancel(jobId)`, `resynth(id, lang)` —
   the single mutation path both views use (37's poller is replaced by this store
   when both land; keep interfaces identical).
2. Panel layout: card per stage key (grouped per language like `status`, §31),
   showing status glyph/color, duration when completed, error code+message when
   failed, progress bar + segment counters while running (from `stage_progress` /
   `segment_completed` events), `[cloud]` badge per provider info (from
   `/api/project`).
3. Controls: Run all / Run until <stage select> / target language checkboxes
   (manifest targets); Cancel button while active; controls disabled appropriately
   (no double-submit). 409 DS-COST-001 → modal showing estimate breakdown with
   "Proceed" → retry with `acknowledge_cost: true`; 409 DS-CONSENT-001 → modal with
   the CLI instruction (no acceptance in UI by design, ADR-005 — the affirmative act
   stays in the terminal; rationale line included).
4. Warnings feed: streaming list (max 200, newest first) of warning events with
   segment links — clicking scrolls/focuses the row in the segments view (hash
   `#seg_0042` navigation contract with 37).
5. Outputs section: after export/subtitles complete, list artifact file names
   (relative paths from §4.1) with a copy-path button (no download endpoint in v1 —
   files are local; tooltip explains).
6. Completion summary card from `run_completed` event data (durations, synth
   cached/new, fit counts, usage/cost actuals).
7. Same CSP/`textContent`/a11y rules as 37 (`role=status` on progress, reduced-motion
   respect).

## Acceptance Criteria

- [ ] With a stub run (TestClient-driven mock pipeline), cards transition
      pending→running→completed live; progress bar reflects stage_progress; warnings
      append with working segment links.
- [ ] Cost modal flow: 409 → modal shows breakdown → proceed reruns with flag (stub).
- [ ] Consent modal renders CLI instruction; no accept button exists (grep test).
- [ ] SSE reconnect: killing the stream (server restart in test) → client
      re-subscribes and replays tail without duplicating stage cards.
- [ ] Cancel rolls the running card back to its pre-run status per §6.1.
- [ ] Outputs list appears only when files exist (progressive).

## Validation

Manual checklist + screenshots in PR (mock run); `tests/ui/test_static.py` extended
(both JS files served, no innerHTML violations, no external URLs referenced —
regex scan for `https?://` in static/ excluding comments).

## Dependencies

36, 37 (shared api.js + navigation contract), 35.

## Non-goals

Log viewer (JSONL is on disk), plan preview UI (CLI `plan` covers it in v1),
UI-side consent acceptance.

## Design References

DESIGN.md §10.4, §6.5, §7.6, §11.4 (consent stays in CLI); ADR-005.
