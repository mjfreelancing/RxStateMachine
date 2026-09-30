# 06 — Timers, Timeouts and External State

**Builds on:** 03, 05. **Prefix:** `TES`.

## Feature input (paste into `/speckit-specify`)

Support time-driven behaviour and state that lives outside the machine. Developers can fire a trigger
after a delay, attach a timeout to a state ("after N in this state, fire Y") that arms on entry and
cancels on exit, and build retry/backoff and heartbeat patterns from Rx timers — all deterministic under
virtual time. A machine can also keep its current state in a consumer-owned store (such as a database
entity) instead of in memory.

## Requirements

**Timers**
- TES-01 A trigger MUST be firable after a delay or from any timer-like observable using the ordinary
  observable input path.
- TES-02 A state timeout ("after duration D in state S, fire trigger T") MUST be available.
- TES-03 A state timeout MUST arm on entry and cancel on exit; reentry restarts it; an internal
  transition leaves it untouched (DD-17).
- TES-04 All timers MUST use the injected scheduler so tests can run in virtual time.
- TES-05 Disposing the machine MUST cancel every pending timer.
- TES-06 Pending timers MUST NOT be included in persisted data.
- TES-07 Retry with backoff, max-retry guards, and heartbeat timeouts MUST be achievable with the
  provided pieces and shown in samples.

**External state**
- TES-08 A machine MUST be creatable with a state getter and setter supplied by the consumer.
- TES-09 With external storage the consumer's store is the single owner of state; the machine keeps no
  copy (DD-14).
- TES-10 State MUST be read before each decision and written as part of each transition; if the write
  fails, the transition is not published and the error is handled per policy.

## Edge cases and failure modes
- TES-E1 A timeout and an explicit trigger arrive together (DD-17: first processed wins).
- TES-E2 A timer fires after disposal or after its state has been left (must have no effect).
- TES-E3 Zero, negative, or extremely large durations.
- TES-E4 Several timeouts on one state.
- TES-E5 A timer trigger that is not permitted in the resulting state (handled per policy).
- TES-E6 The external store changes between two reads (another process wrote it).
- TES-E7 The getter or setter throws.
- TES-E8 A state entered by reentry restarts its timeout.
- TES-E9 The scheduler is shut down or paused.
- TES-E10 Large numbers of machines each with timers do not create unbounded resources.

## Use-case coverage
UC-3, UC-4, UC-6, UC-8 — see [use-cases.md](../use-cases.md).

## Design decisions relied on
DD-05, DD-13, DD-14, DD-17, DD-22.

## Open questions (for `/speckit-clarify`)
- Winner rule when a timeout and a trigger race in a way that is not decided by processing order.
- Does the external getter/setter also apply to hierarchy state (feature 07)?
- Are timeouts expressed per state, per trigger, or both?

## Out of scope
Saved snapshots (08), queued mode (10), persistence adapters.
