# Issue 11: Voice-clone consent gate

## Title

Implement voice-clone consent gate module and `consent` CLI command

## Summary

Implement `core/consent.py` and `cli/consent_cmd.py` per DESIGN.md §11.4 / ADR-005:
a versioned, recorded user-level acceptance of the Voice & Likeness Policy, required
before any voice-reference building or synthesis, with an explicit non-interactive
path and per-project snapshotting.

## Context

This is the product's central ethical control. Stages 25/26 hard-depend on it; the
policy text itself ships in issue 03 (`docs/POLICY.md`).

## Scope

In: consent record IO, gate check API, interactive prompt, CLI command, env acceptance.
Out: policy text authoring (03), export disclosure metadata (29).

## Detailed Requirements

1. Record file: `<platformdirs user_config>/consent.json` —
   `{schema_version: 1, policy_version: int, accepted_at: UTC ISO, method:
   "interactive" | "env"}`. Atomic write; permissions 0600.
2. `CURRENT_POLICY_VERSION = 1` constant co-located with the exact affirmation string,
   which must be byte-identical to the sentence in `docs/POLICY.md` (test compares the
   file and the constant).
3. `check_consent() -> ConsentStatus` (`accepted | outdated | missing`).
4. `require_consent(interactive: bool)`:
   - `accepted` → returns `ConsentSnapshot` for the manifest;
   - `missing`/`outdated` + interactive TTY → display policy summary (verbatim
     affirmation + pointer to POLICY.md), require typing `yes` (localized input not
     accepted; exact literal) → record + return;
   - non-interactive → if env `DUBSTUDIO_ACCEPT_VOICE_POLICY=1`, record with
     `method: "env"` and log a NOTICE line; else raise `ConsentError DS-CONSENT-001`
     (exit 3) with hint naming `dubstudio consent --accept` and the env var.
5. CLI `dubstudio consent`: `--status` (default; prints state, version, date, method,
   `--json` support), `--accept` (runs the interactive flow; with
   `DUBSTUDIO_ACCEPT_VOICE_POLICY=1` allowed non-interactively), `--revoke` (deletes
   record after confirm; prints that existing projects keep their snapshots).
6. Snapshot API `snapshot() -> ConsentSnapshot {policy_version, accepted_at}` — stages
   copy it into `manifest.consent_snapshot` at synth time (used by 25/26).
7. Policy-version bump behavior: `outdated` behaves exactly like `missing` (gate
   re-triggers) but the prompt says the policy changed and links the diff section in
   POLICY.md.

## Acceptance Criteria

- [ ] Fresh env: `dubstudio consent --status` exits 0 reporting `missing`.
- [ ] `require_consent` in non-TTY without env var raises DS-CONSENT-001 → exit 3.
- [ ] With `DUBSTUDIO_ACCEPT_VOICE_POLICY=1`, acceptance is recorded with
      `method:"env"` and a NOTICE log.
- [ ] Interactive flow (pexpect or Typer CliRunner input) requires literal `yes`;
      `y`/`YES` re-prompts once then aborts with exit 3.
- [ ] Bumping the constant to 2 in a test makes a previous record report `outdated`.
- [ ] Affirmation string equality test against `docs/POLICY.md` passes.
- [ ] consent.json written 0600; concurrent accept is atomic.

## Validation

`uv run pytest tests/core/test_consent.py tests/cli/test_consent_cmd.py`.

## Dependencies

06 (05 transitively). Policy text: 03.

## Non-goals

Per-voice/per-subject consent records (v2 candidate), watermarking (provider-level,
issue 18), export metadata (29).

## Design References

DESIGN.md §11.4, §9 (exit 3); ADR-005.
