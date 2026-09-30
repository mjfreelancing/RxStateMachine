# 05 — Async, Cancellation and Error Policies

**Builds on:** 02, 03, 04. **Prefix:** `ASY`.

## Feature input (paste into `/speckit-specify`)

Add asynchronous behaviour and failure handling. Guards, entry/exit/transition actions, and firing
itself gain awaitable variants that never block, accept cancellation, and keep the machine consistent
when something fails or is cancelled. A configurable error policy — `Throw` (default), `Observe`, or
`Ignore` — governs unhandled triggers and exceptions from consumer code, and failures with no caller
are surfaced on the errors stream. Developers can subscribe to lifecycle notifications and unsubscribe
cleanly.

## Requirements

**Async**
- ASY-01 Async variants MUST exist for entry, exit, and transition actions, guards, conditional
  permits, and firing.
- ASY-02 Async APIs MUST return `Task`, MUST NOT block on async work, and MUST NOT expose
  `IAsyncEnumerable`.
- ASY-03 Async operations MUST accept a cancellation token; cancellation takes precedence over the
  error policy (DD-16).
- ASY-04 Async guards MUST be evaluated one at a time, in order, and async firing MUST follow DD-08.
- ASY-05 Synchronous `Fire` against a configuration that requires async work MUST be rejected with a
  clear message rather than blocking.
- ASY-06 An awaited firing MUST report only its own trigger's outcome.

**Error policies**
- ASY-07 The default policy MUST be `Throw`; `Observe` and `Ignore` MUST be opt-in via options.
- ASY-08 `Throw` MUST let the failure reach the caller.
- ASY-09 `Observe` MUST publish a rich error (trigger, state, exception, policy) on the errors stream.
- ASY-10 `Ignore` MUST absorb the failure and keep a bounded record plus a running count.
- ASY-11 The same policy set MUST govern unhandled triggers and exceptions from guards and actions.
- ASY-12 Failures from observable input MUST always be published on the errors stream (DD-13).
- ASY-13 A handled failure MUST NOT complete the errors stream or disconnect the trigger source.
- ASY-14 After a failure or cancellation the machine's position and all streams MUST be consistent
  (DD-07).

**Lifecycle notifications**
- ASY-15 Developers MUST be able to subscribe to transition lifecycle notifications and unsubscribe.

## Edge cases and failure modes
- ASY-E1 A token already cancelled before firing starts.
- ASY-E2 Cancellation during an entry action, after the position is committed.
- ASY-E3 A subscriber to the errors stream that itself throws.
- ASY-E4 Ignore's record reaches its bound; oldest entries are dropped and counting continues.
- ASY-E5 Two `FireAsync` calls overlap in immediate mode.
- ASY-E6 An async guard that never completes (only cancellation ends it).
- ASY-E7 Multiple exceptions in one firing (for example an exit action and then a compensating action).
- ASY-E8 A synchronization context is present; the library does not capture or resume on it.
- ASY-E9 An exception in a transition-completed callback.
- ASY-E10 Cancelling a firing whose trigger was queued behind another.

## Use-case coverage
UC-2, UC-6 — see [use-cases.md](../use-cases.md).

## Design decisions relied on
DD-07, DD-08, DD-11, DD-13, DD-16.

## Open questions (for `/speckit-clarify`)
- Behaviour of overlapping `FireAsync` calls in immediate mode: reject, queue, or serialize?
- Default size of Ignore's bounded record, and whether it is configurable.
- What exactly does a cancelled awaited firing return or throw once the position is committed?

## Out of scope
Timers (06), queued/concurrent firing (10).
