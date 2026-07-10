# Issue 10: Stage engine

## Title

Implement stage DAG, state machine, fingerprint staleness, runner with events

## Summary

Implement `engine/*`: the dependency graph of DESIGN.md §6.3, the status machine §6.1,
fingerprint-based staleness §6.2/§4.4, the planner §6.4, and the runner with JSONL
events §6.5 and cancellation semantics.

## Context

The engine is the product's core control flow; CLI `run` (issue 32) and the UI job API
(issue 36) are thin callers of this module.

## Scope

In: graph/state/invalidate/runner/plan + a `Stage` interface + tests using stub stages.
Out: real stage implementations (21–30), CLI rendering, cost confirmation UX (engine
exposes estimates; issue 32 prompts).

## Detailed Requirements

1. `Stage` interface (`engine/graph.py`):
   `name: str` (per-lang stages get instance-per-lang with `lang` attr),
   `deps(manifest, config) -> list[str]` (stage keys),
   `config_subset(config) -> dict` (delegates to `Settings.for_stage`),
   `input_artifacts(store) -> list[Path]` (files whose content hashes feed the
   fingerprint), `provider_stamp() -> dict | None`,
   `run(ctx) -> StageResult` where `ctx` carries store, config, providers, emit(event),
   `cancelled: threading.Event`.
2. Graph builder: instantiate stage set from manifest languages exactly per §6.3
   (including the skip rule for `separate` and voice_ref's conditional dep). Cycle check
   at build (defensive).
3. Fingerprint = `hashing.fingerprint({engine: FP_VERSION, inputs: [...], config: ...,
   provider: ...})` (§4.4). `evaluate(manifest) -> dict[key, Evaluation]` classifying
   each stage: `up_to_date | pending | stale | failed | blocked | skipped`, cascading
   staleness to descendants.
4. Planner (`plan.py`): given filters (`until`, `only`, `from_`, `target`), topological
   order of stages to execute + reasons + per-stage `cloud: bool` badge (from provider
   registry info) + aggregated `CostEstimate | None`. Pure (no side effects).
5. Runner (`runner.py`):
   - executes plan sequentially (one stage at a time; per-segment concurrency lives
     inside stages);
   - transitions per §6.1 with manifest writes under the project lock;
   - emits events per §6.5 with a fresh `run_id` (uuid4) to log file
     `logs/run-<UTC ts>.jsonl` and an in-process subscriber list;
   - failure: record `error` on stage, stop the run (later stages remain pending),
     raise the stage's error after emitting `run_failed`;
   - cancellation: `cancel()` sets the event; between/within stages (stages poll),
     current stage's status **rolls back to its pre-run value**, partial per-segment
     outputs are kept on disk, emit `run_cancelled`, raise `CancelledError` (exit 10);
   - SIGINT handler installed only around CLI-invoked runs (hook point; wiring in 32).
6. `invalidate(stage, lang=None, cascade=True)` and `invalidate_input()` APIs (§6.2):
   set statuses to `stale` (or re-hash input), persisting via store.
7. Per-segment cache contract: engine passes stages a `SegmentCache` view
   (`unchanged(segment_id, key_hash) -> bool`) backed by the stage's previous artifact
   doc; stages 26/27 consume it (defined here so the interface is stable).
8. Config flip handling: `separation.enabled` False→True or True→False re-evaluates
   `separate` per §6.1 skipped transitions and cascades.

## Acceptance Criteria

- [ ] Stub-stage tests cover every transition row of the §6.1 table.
- [ ] Editing a fake upstream artifact byte flips exactly the right descendants to
      stale (table-driven: edit transcript → translate:*, synthesize:*, …, subtitles:*).
- [ ] `plan(until="translate", target="en")` on a fresh project lists
      ingest, separate, transcribe, translate:en only, in order.
- [ ] Cancellation mid-stage rolls back status and preserves partial outputs (stub
      stage writes 3 of 5 segment files then blocks on the event).
- [ ] Events JSONL for a two-stage run matches the §6.5 schema (validated via
      `RunEvent`) including run_started/stage_*/run_completed ordering.
- [ ] Re-running a completed project is a no-op (0 stages planned).

## Validation

`uv run pytest tests/engine/` (pure; no ffmpeg/providers — stub stages only);
mypy strict.

## Dependencies

06, 09.

## Non-goals

Parallel stage execution, cross-language parallelism, distributed execution, retries at
stage granularity (provider-level retries only, §7.4).

## Design References

DESIGN.md §6 (all), §4.4, §9 (exit codes 9/10/13), §7.6.
