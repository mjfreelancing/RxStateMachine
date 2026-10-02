# 08 — Snapshot Persistence

**Builds on:** 06, 07. **Prefix:** `PER`.

## Feature input (paste into `/speckit-specify`)

Allow a machine to be saved and restored. A snapshot captures the machine's position (including
hierarchy history, and a reserved place for the pending triggers queued mode adds in feature 10) as
a storage-neutral, versioned description
that the consumer serializes and stores however they like. Restoring is atomic, validates the
snapshot, and does not run actions. Persisting on every change is just an ordinary subscription and
needs no special API. The library does not ship a serializer or storage.

## Requirements
- PER-01 A snapshot MUST capture the current state and the remembered history for hierarchical
  states. Its shape MUST reserve a place for pending queued triggers, in order, but queued mode does
  not exist until feature 10 and this feature may rely only on earlier features (DD-03): that place
  is left empty here and feature 10 fills it (HRD-06).
- PER-02 The snapshot MUST be a storage-neutral data shape that any serializer can handle.
- PER-03 A snapshot MUST be versioned; a snapshot from an incompatible version MUST be refused with a
  clear message.
- PER-04 Restore MUST be atomic (all or nothing), MUST validate structure (an unknown state is
  refused), MUST run no entry or exit actions, and MUST emit no historic transitions (DD-19).
- PER-05 Restore MUST apply to an identically configured machine; a configuration mismatch MUST be
  reported with diagnostics.
- PER-06 Persisting on every change MUST be achievable with an ordinary subscription, and a
  documented example MUST show it.
- PER-07 The library MUST NOT ship a serializer, storage, or serializer dependency.
- PER-08 The interaction with external state storage (feature 06) MUST be defined and documented.
- PER-09 Activation and deactivation hooks MUST be introduced here, because a restore is the one
  moment the machine occupies a state without a transition having run (DD-10). Activation MUST be an
  explicit operation the consumer invokes, MUST be idempotent, MUST run the activation hooks of the
  current state and of every ancestor outermost-first, and MUST NOT emit a transition. Disposal MUST
  NOT run deactivation hooks.

## Edge cases and failure modes
- PER-E1 Restoring into a running machine, or during a transition.
- PER-E2 A corrupted or truncated snapshot.
- PER-E3 A snapshot naming a state that no longer exists after a configuration change.
- PER-E4 A snapshot taken while a firing is in progress.
- PER-E5 Pending triggers keep their order after restore.
- PER-E6 History values for nested states survive a round trip.
- PER-E7 Restore succeeds, but subscribers must learn the restored state (what is emitted and when).
- PER-E8 Two snapshots of an unchanged machine are equal.
- PER-E9 Timers are not restored; consumers re-arm behaviour by re-entering states.

## Use-case coverage
UC-1 (persist-on-change), UC-8 — see [use-cases.md](../use-cases.md).

## Design decisions relied on
DD-05, DD-10, DD-14, DD-17, DD-18, DD-19.

## Open questions (for `/speckit-clarify`)
- What does a subscriber see immediately after a restore?
- How much configuration identity is verified on restore (names only, or full shape)?
- Which serializer, if any, does a sample use to show persisting a snapshot?
- Is activation (PER-09) also offered without a restore — for example to run the current state's
  hooks at startup, which is the other half of CORE-12 — and is deactivation exposed at all in this
  feature?

## Out of scope
Any serializer, database, or file storage; distributed coordination.
