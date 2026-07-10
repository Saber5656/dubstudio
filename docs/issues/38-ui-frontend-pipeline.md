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
   message}}, warnings: [...], lastSummary}`; broadcasts via `EventTarget`.
   Mutation methods (all through the shared `api.js` wrapper, which sets
   `X-Dubstudio-Csrf`; exact issue 36 endpoints):
   - `startRun({until?, only?, lang?}, {acknowledgeCost=false})` → `POST /api/run`
     body `{until?, only?, lang?, acknowledge_cost?}`; 409 responses are surfaced
     as typed results by error code: `DS-COST-001` (carries `estimate`) → cost
     modal; `DS-CONSENT-001` → consent modal; `DS-UI-001` → "job already running"
     toast;
   - `cancel(jobId)` → `POST /api/jobs/{jobId}/cancel`;
   - `resynth(id, lang)` → `POST /api/segments/{id}/resynthesize?lang=…`.
   This store replaces 37's `jobstate.js` poller behind the identical interface.
   Event-data contract consumed here (emitted by the issue 10 runner / stages;
   every field is rendered defensively — missing → "—"):
   `stage_progress.data = {fraction: 0..1, message?}`;
   `segment_completed.data = {cached: bool}`;
   `stage_completed.data = {duration_s, summary?}`;
   `run_completed.data = {stages: {key: {duration_s}}, synth: {cached, new},
   fit_counts: {ok, shortened, warn_overflow}, usage: [str],
   outputs: {lang: [relpath]}}`.
2. Panel layout: card per stage key (grouped per language like `status`, §31),
   showing status glyph/color, duration when completed, error code+message when
   failed, progress bar + segment counters while running (from `stage_progress` /
   `segment_completed` events), `[cloud]` badge per provider info (from
   `/api/project`).
3. Controls: Run all / Run until <stage select> / **single target-language selector**
   (dropdown over manifest targets + an "all targets" option that omits `lang` —
   matches the singular `lang?` field of `POST /api/run`; multi-select is not a v1
   surface); Cancel button while active; controls disabled appropriately (no
   double-submit). 409 DS-COST-001 → modal showing estimate breakdown with
   "Proceed" → retry with `acknowledge_cost: true`; 409 DS-CONSENT-001 → modal with
   the CLI instruction (no acceptance in UI by design, ADR-005 — the affirmative act
   stays in the terminal; rationale line included).
4. Warnings feed: streaming list (max 200, newest first) of warning events with
   segment links — clicking scrolls/focuses the row in the segments view (hash
   `#seg_0042` navigation contract with 37).
5. Outputs section: lists `outputs` from `GET /api/project` (issue 35 provides the
   project-relative artifact paths; refreshed after `run_completed`, which also
   carries the same list in its data) with a copy-path button (no download endpoint
   in v1 — files are local; tooltip explains).
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

Interactive criteria are validated with a scripted stub-run harness
(`tests/ui/manual_harness.py`: launches the server on the fixture project with mock
providers and drives a run via TestClient so a human can observe transitions) plus
recorded evidence in the PR: screen capture of live stage transitions, the cost and
consent modals, an SSE reconnect (server restarted mid-run), and cancel rollback.
Static checks stay automated: `tests/ui/test_static.py` extended (both JS files
served, no innerHTML violations, no external URLs referenced — regex scan for
`https?://` in static/ excluding comments). Browser automation is deliberately out
of v1 scope (same trade-off as issue 37).

## Dependencies

35 (project/outputs payload), 36, 37 (shared api.js + navigation contract) —
matches the ISSUE_PLAN row.

## Non-goals

Log viewer (JSONL is on disk), plan preview UI (CLI `plan` covers it in v1),
UI-side consent acceptance.

## Design References

DESIGN.md §10.4, §6.5, §7.6, §11.4 (consent stays in CLI); ADR-005.
