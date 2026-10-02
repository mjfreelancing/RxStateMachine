# 10 — Concurrency, Performance and Release Readiness

**Builds on:** all previous features. **Prefix:** `HRD`.

## Feature input (paste into `/speckit-specify`)

Finish the library. Add the opt-in queued firing mode that is safe under concurrent callers, prove it
with reproducible stress tests, prove the hot path is fast and allocation-free with a benchmark
comparison and an allocation test, and make the project releasable: complete XML documentation, a full
sample catalogue that runs in CI, and a final readiness check for the first stable release. This feature delivers the final
deliverable.

## Requirements

**Queued mode**
- HRD-01 Queued mode MUST be an explicit opt-in and MUST never be enabled implicitly (DD-04).
- HRD-02 In queued mode `Fire`/`FireAsync` MUST be safe from many threads; triggers are processed one
  at a time in arrival order with no lost or duplicated triggers.
- HRD-03 A call that only enqueues a trigger MUST return a transition whose kind is `Queued`, and an
  awaitable form MUST let a caller wait for its own trigger's outcome.
- HRD-04 Queue depth and idleness MUST be observable.
- HRD-05 Shutdown MUST be clean: behaviour on dispose (drain or discard) is defined and documented.
- HRD-06 Pending queued triggers MUST be cancellable and MUST fill the place the snapshot shape
  reserves for them (PER-01), in order.

**Verification**
- HRD-07 A stress suite MUST run many workers firing random valid and invalid triggers started
  together, asserting no lost updates, no gaps in the transition history (each source equals the
  previous destination), and no unexpected exceptions; failures MUST be replayable from a logged seed
  (DD-21).
- HRD-08 Property-based tests MUST assert state-transition invariants across the whole feature set.
- HRD-09 Every timed behaviour MUST be reproducible under virtual time.
- HRD-10 Zero steady-state allocation for a simple synchronous transition MUST be enforced by a
  deterministic test, and a benchmark run on one machine MUST show microsecond-scale cost for it. A
  before/after comparison is recorded for any change in this feature that touches the hot path; there
  is no stored baseline or CI latency gate (DD-22).

**Release readiness**
- HRD-11 Every use case in [use-cases.md](../use-cases.md) known at release MUST have a documented,
  runnable sample, all built and run in CI. The set is open-ended: there is no fixed number or domain
  list, and new use cases add samples.
- HRD-12 The API reference MUST be complete (100% XML documentation). A published documentation site
  is not required.
- HRD-13 The public API baseline and README MUST be up to date and consistent with the shipped
  surface.
- HRD-14 Before the first stable version is published, the maintainer MUST work through a short
  manual checklist confirming CI is green on the release commit and the README matches the shipped
  surface. This is a checklist, not an automated gate.

## Edge cases and failure modes
- HRD-E1 Bounded versus unbounded queue and behaviour when a bound is reached.
- HRD-E2 An exception in the drain loop must not stop later triggers.
- HRD-E3 A trigger fired during shutdown, and disposal while draining.
- HRD-E4 A queued trigger fired from inside an action of the queue's own drain.
- HRD-E5 Timers and the queue interplay: a timer fires while the queue is busy.
- HRD-E6 A synchronization context is present; the library must not deadlock or capture it.

## Use-case coverage
UC-5 (queued firing under load); all samples — see [use-cases.md](../use-cases.md).

## Design decisions relied on
DD-04, DD-09, DD-19, DD-21, DD-22.

## Open questions (for `/speckit-clarify`)
- Default queue bound and behaviour at the bound.
- Drain versus discard on dispose.
- Whether to publish the package to a public feed immediately after the readiness check.

## Out of scope
Distributed queues, cross-process coordination, and orchestration (see [saga-exploration.md](../saga-exploration.md)).
