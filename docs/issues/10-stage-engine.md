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
   `name: str` (per-lang stages get instance-per-lang with `lang: str | None`; stage
   key = `name` or `f"{name}:{lang}"`),
   `deps(manifest, config) -> list[str]` (stage keys),
   `config_subset(config) -> dict` (delegates to `Settings.for_stage`),
   `input_artifacts(store) -> list[Path]` (files whose content hashes feed the
   fingerprint, **hashed in the declared list order** — order is part of the
   canonical fingerprint payload), `provider_stamp() -> dict | None`,
   `run(ctx) -> StageResult`.
   `StageContext` (frozen dataclass): `store: ProjectStore`, `config: Settings`,
   `providers: ProviderSet` (resolved instances for this run), `lang: str | None`,
   `emit: Callable[[RunEvent], None]`, `cancelled: threading.Event`,
   `interactive: bool`, `segment_cache: SegmentCache | None`.
   `StageResult` (frozen dataclass): `warnings: list[str]` and optional
   `summary: dict` (lands in the stage_completed event `data`) — nothing else:
   stages write their own artifacts via the store; the **runner** owns manifest
   updates (status, fingerprint, timestamps, error) so a stage cannot corrupt
   engine state.
   Cancellation contract: stages must poll `ctx.cancelled` between units of work AND
   pass it as `cancel_event` into every `procs.run`/provider call that supports it
   (issue 07) so in-flight subprocesses get SIGTERM → 5 s → SIGKILL per §6.1.
2. Graph builder: instantiate stage set from manifest languages exactly per §6.3
   (including the skip rule for `separate` and voice_ref's conditional dep). Cycle check
   at build (defensive).
3. Fingerprint = `hashing.fingerprint({engine: FP_VERSION, inputs: [...], config: ...,
   provider: ...})` (§4.4). `evaluate(manifest) -> dict[key, Evaluation]` classifying
   each stage: `up_to_date | pending | stale | failed | blocked | skipped`, cascading
   staleness to descendants. Dependency-state table (deterministic):

   | upstream state | this stage evaluates as |
   |---|---|
   | any dep pending/running/failed/blocked | blocked |
   | all deps completed/skipped, no own record | pending |
   | own fingerprint mismatch | stale (cascades) |
   | dep completed but a declared input artifact is missing on disk | that dep is re-marked stale at evaluate time (cascades here) |
   | own record failed | failed (runnable) |
4. Planner (`plan.py`): produces the topological execution list + reasons + per-stage
   `cloud: bool` badge + aggregated `CostEstimate | None`. Pure (no side effects).
   Provider metadata comes through an engine-owned `ProviderMeta` protocol
   (`mode: local|cloud`, `estimate(work) -> CostEstimate | None`) injected by the
   caller — issue 13's registry satisfies it later; engine tests use stubs (keeps
   this issue independent of 13).
   Filter semantics (exact):
   - no filter: every stage evaluated pending/stale/failed, in topo order;
   - `until S`: that set intersected with S and its ancestors;
   - `only S`: S alone; if any dep is not completed/skipped → `StageError
     DS-STAGE-001` listing the unmet deps;
   - `from S`: S is force-marked stale first, then plan as the no-filter case
     (i.e. S + all descendants);
   - `target L…`: per-lang stage instances restricted to those langs; shared stages
     included whenever any selected lang needs them; S referring to a per-lang stage
     without `target` means all targets;
   - filters name base stages (`translate`), never keyed instances.
5. Runner (`runner.py`):
   - executes plan sequentially (one stage at a time; per-segment concurrency lives
     inside stages);
   - transitions per §6.1 with manifest writes under the project lock;
   - emits events per §6.5 with a fresh `run_id` (uuid4) to log file
     `logs/run-<UTC ts>.jsonl` and an in-process subscriber list; **every event
     passes through the issue 05 redaction filter before both the file write and
     subscriber delivery** (no key material in `message`/`data` — tested);
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
- [ ] A fake `ELEVENLABS_API_KEY` value planted in a stub stage's warning message is
      redacted in both the JSONL file and a test subscriber's received event.
- [ ] Cancellation of a stub stage blocked in a `procs.run` subprocess (sleep child)
      terminates the child (SIGTERM path asserted) within ~6 s.
- [ ] Filter semantics table-driven test covers: until/only (unmet deps →
      DS-STAGE-001)/from/target combinations and the shared-stage inclusion rule.
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
