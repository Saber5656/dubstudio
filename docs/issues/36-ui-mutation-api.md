# Issue 36: UI mutation and job API

## Title

Implement UI mutation endpoints: segment PATCH, resynthesize, run/jobs, SSE events

## Summary

Implement the mutating routes of DESIGN.md §10.3: translation edits with precise
staleness, single-segment re-synthesis, job-controlled pipeline runs with a
single-concurrency guard, cancellation, and the SSE event stream bridging engine
events to the frontend.

## Context

These routes make the UI a real HITL surface. They reuse engine/runner APIs (10) and
mirror CLI semantics (32/33) so both surfaces stay behaviorally identical.

## Scope

In: `PATCH /api/segments/{id}`, `POST /api/segments/{id}/resynthesize`,
`POST /api/run`, `GET /api/jobs/{id}`, `POST /api/jobs/{id}/cancel`, `GET /api/events`.
Out: server foundation (34), read routes (35), frontend consumption (37/38).

## Detailed Requirements

1. `PATCH /api/segments/{id}?lang=L` body `{text?: str, status?: enum}` (≥ 1 field):
   - validation identical to `segments import` single-segment rules (33/B6): id
     exists, text non-empty, status enum;
   - text change → status `edited` (unless body sets `approved` explicitly), doc
     written atomically under the project lock, per-segment invalidation (§6.2);
   - 409 with `{error:{code:"DS-LOCK-001"}}` when a job/run holds the lock;
   - response = the refreshed merged row (35 shape).
2. Job model (`ui/jobs.py`): at most **one** active job per server (§10.3);
   `POST /api/run` body `{until?, only?, lang?}` (validated like CLI filters) →
   `{job_id}` 202, or 409 `DS-UI-001` when active. Jobs run the engine runner on a
   worker thread; job registry keeps `{id, state: queued|running|completed|failed|
   cancelled, started_at, finished_at, error?, summary?}` for the server lifetime.
3. Consent/cost interplay (no interactive prompts in a server): planning result
   requiring consent → 409 `DS-CONSENT-001` with hint to run `dubstudio consent
   --accept`; cost estimate above threshold → 409 `DS-COST-001` unless body
   `{"acknowledge_cost": true}` (frontend shows the estimate from the 409 payload
   `{estimate}` and retries with the flag).
4. `POST /api/segments/{id}/resynthesize?lang=L`: shorthand creating a job that runs
   synthesize+fit restricted to that segment (engine per-segment invalidation +
   filtered run); same 409 rules.
5. `POST /api/jobs/{id}/cancel`: `runner.cancel()` semantics (§6.1 rollback); idempotent.
6. `GET /api/events` (SSE, §10.3): subscribes to the in-process event bus; emits
   `event: <RunEvent.event>` + `data: <RunEvent JSON>`; 15 s `: heartbeat` comments;
   on connect, replays the current job's `stage_started`-to-now tail (last 100
   events) so late-joining clients render state; client disconnect detection stops
   the generator. CSRF not required (GET), cookie required (34 middleware).
7. All mutations emit events (segment_updated custom event added to §6.5 enum —
   extend `model/events.py` accordingly with a doc note).

## Acceptance Criteria

- [ ] PATCH matrix: text-only (→edited), status-only, both, empty body (422), bad id
      (422), unknown lang (422), during active job (409 lock).
- [ ] PATCH → exactly one segment stale in synthesize (engine assertion, mirrors 33).
- [ ] Second `POST /api/run` while active → 409; after completion → accepted.
- [ ] Consent-missing run → 409 with DS-CONSENT-001 payload; cost 409 carries
      estimate; retry with acknowledge_cost runs (stub estimates).
- [ ] Cancel mid-run (stub blocking stage): job state `cancelled`, manifest rolled
      back, subsequent run accepted.
- [ ] SSE: TestClient stream receives replayed tail + live events in order + a
      heartbeat; CSRF-free GET verified; disconnected client cleans up (no thread
      leak — assert bus subscriber count returns to baseline).

## Validation

`uv run pytest tests/ui/test_mutation_api.py test_jobs.py test_sse.py` (stub engine +
mock providers; no real media needed).

## Dependencies

34, 35, 10, 33 (shared validation rules), 11.

## Non-goals

Multi-job queueing, websockets, transcript editing, auth changes.

## Design References

DESIGN.md §10.3, §6.1–6.2, §6.5, §7.6, §11.2 B3/B6.
