# ADR-005: Voice-clone consent gate, AI-disclosure metadata, watermark-by-default

- Status: accepted
- Date: 2026-07-10
- Decision drivers: user selection ("consent gate + disclosure guideline" level);
  public-OSS reputational and abuse risk for voice cloning.

## Context

dubstudio's core feature is voice cloning. As public OSS, it will inevitably be pointed
at voices the operator does not own. We cannot technically prevent all misuse of local
software, but we can (a) force an explicit, recorded assertion of consent, (b) make
outputs honest by default, and (c) choose defaults that embed provenance.

## Decision

1. **Consent gate** (DESIGN.md §11.4): cloning paths (`voice_ref`, `synthesize`) hard-fail
   with exit 3 until the versioned Voice & Likeness Policy is accepted; acceptance is
   recorded (user-level) and snapshotted into each project manifest; non-interactive
   acceptance only via explicit env var. Policy version bump re-triggers the gate.
2. **Disclosure**: export writes AI-dubbing metadata tags into output containers;
   subtitle files carry a header note; docs describe platform disclosure duties.
3. **Watermark by default**: the default local TTS is Chatterbox Multilingual
   specifically because PerTh perceptual watermarking is embedded in every output
   (research doc §3); `watermark_builtin` is a first-class provider capability shown in
   `providers list`.
4. **Policy docs**: shipped `docs/POLICY.md` (prohibited uses, reporting), referenced in
   README; separate from SECURITY.md.

## Consequences

- One-time friction for every user (accept once per policy version).
- CI/automation must set `DUBSTUDIO_ACCEPT_VOICE_POLICY=1` explicitly — a deliberate,
  auditable act.
- Non-watermarking providers (ElevenLabs relies on its own provenance tooling) are still
  allowed; the capability difference is surfaced, not hidden.

## Alternatives considered

- **Docs-only**: zero friction, but indefensible posture for a public clone tool.
- **Mandatory watermarking of all outputs in-pipeline**: watermark tech is
  provider-specific and research-grade for generic audio; deferred to v2
  (watermark verification command idea).
