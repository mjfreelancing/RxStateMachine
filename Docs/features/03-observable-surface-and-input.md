# 03 — Observable Surface and Trigger Input

**Builds on:** 02. **Prefix:** `OBS`.

## Feature input (paste into `/speckit-specify`)

Make the machine a first-class reactive citizen. It exposes its current state as a hot observable that
replays the latest value, plus streams for state changes, transitions, transition completions, guard
results, permitted triggers, and errors — all read-only and disposable. It also accepts triggers
directly from any observable, behaving exactly as if `Fire` had been called, so message buses, UI
events, and timers can drive it without adapters. A developer who only uses `Fire` gets full
functionality without knowing Rx.

## Requirements

**Outputs**
- OBS-01 The machine MUST be an observable of its state; it MUST be hot and replay the latest value so
  a late subscriber receives the current state immediately.
- OBS-02 A state-change stream MUST report previous state, current state, when it occurred, and whether
  it was a reentry or internal change.
- OBS-03 A transition stream MUST report each transition when the new position is committed, and a
  transition-completed stream MUST report it after the last entry action, in the order of DD-08.
- OBS-04 A guard-result stream MUST report every guard evaluation with its outcome and description.
- OBS-05 A permitted-triggers stream MUST re-emit as defined in DD-15.
- OBS-06 An errors stream MUST carry rich error values (trigger, state, exception, policy); policy
  behaviour is defined in feature 05.
- OBS-07 All streams MUST be read-only: no caller can push values into them.
- OBS-08 Every stream MUST be disposable, MUST compose with standard Rx operators, and MUST honour the
  Rx grammar (serialized notifications, none after completion).
- OBS-09 Subscribing and unsubscribing MUST leak nothing.
- OBS-10 A failing subscriber MUST NOT stop the machine or other subscribers.

**Inputs**
- OBS-11 Any observable of triggers MUST be able to drive the machine, with the same behaviour as
  `Fire` (DD-02).
- OBS-12 Payload-carrying triggers MUST be accepted from observables via the documented wrapper.
- OBS-13 An input subscription MUST be independently disposable; disposing it stops that input only.
- OBS-14 An upstream that completes or errors MUST NOT stop the machine.
- OBS-15 Failures from observable input have no caller and MUST be published on the errors stream
  (DD-13).

**Scheduling, time, lifecycle**
- OBS-16 An optional scheduler (default `Scheduler.Default`) MUST determine where notifications are
  raised; consumers can still observe on any scheduler themselves.
- OBS-17 Timestamps MUST come from the injected scheduler/clock so tests can use virtual time.
- OBS-18 Disposal MUST behave as in DD-10.

## Edge cases and failure modes
- OBS-E1 A subscriber that arrives after many transitions receives only the current state.
- OBS-E2 Subscribing, or firing a trigger, from inside a notification callback.
- OBS-E3 A slow subscriber must not corrupt ordering seen by others.
- OBS-E4 Subscribing after disposal completes immediately; firing after disposal throws.
- OBS-E5 An upstream that emits on several threads at once while in immediate mode.
- OBS-E6 High-frequency input (thousands per second) preserves order and does not drop triggers.
- OBS-E7 An upstream that emits a trigger not permitted in the current state (handled per policy).
- OBS-E8 Two upstreams feeding the same machine.
- OBS-E9 Disposing the input subscription while a firing is in progress.
- OBS-E10 A reentrant or internal transition still emits a state-change notification with the correct
  flags.
- OBS-E11 A guard-result event for a guard that throws.

## Use-case coverage
UC-1, UC-4, UC-7, UC-9 — see [use-cases.md](../use-cases.md).

## Design decisions relied on
DD-02, DD-08, DD-10, DD-13, DD-15, DD-22.

## Open questions (for `/speckit-clarify`)
- Does the transition record include a duration (DD-12)?
- What happens when observable input arrives concurrently in immediate mode: rejected, queued, or
  serialized?
- Does an internal transition re-emit the current state, or only the state-change stream?

## Out of scope
Error policy behaviour beyond "publish on errors stream" (05), timers (06), queued mode (10).
