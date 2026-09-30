# 02 — Core Transitions

**Builds on:** 01. **Prefix:** `CORE`.

## Feature input (paste into `/speckit-specify`)

Provide the synchronous heart of the library: a generic, type-safe state machine whose states and
triggers can be any .NET type, configured with a fluent API. A developer defines states, permitted
triggers, optional guards (with human-readable descriptions), entry/exit/transition actions, internal
and reentrant transitions, and typed trigger payloads; then drives the machine with `Fire`, which
returns the resulting transition. Configuration mistakes fail fast at configuration time with
actionable messages. This feature is synchronous only and covers the immediate (single-threaded,
reentrancy-protected) firing mode.

## Requirements

**Definition**
- CORE-01 The machine MUST be generic over a state type and a trigger type, each any .NET type (enum,
  record, string, integer, class), with non-null constraints where appropriate.
- CORE-02 A machine MUST be created with an initial state; a missing initial state is a
  configuration error.
- CORE-03 A fluent configuration API MUST let a developer configure each state independently.

**Transitions**
- CORE-04 A state MUST be able to permit a trigger, moving to a specified destination.
- CORE-05 A permit MUST be able to carry a guard and an optional human-readable description.
- CORE-06 Multiple guarded permits for one trigger MUST be evaluated one at a time in registration
  order; the first passing guard decides; an optional `otherwise` destination applies when none
  pass (DD-06).
- CORE-07 A destination MUST be selectable dynamically at firing time.
- CORE-08 An internal transition MUST run its action without leaving the state and without running
  exit or entry actions.
- CORE-09 A reentrant transition MUST run exit then entry actions for the same state.

**Actions**
- CORE-10 A state MUST support entry actions, exit actions, and entry actions specific to a trigger.
- CORE-11 Transition-level actions and transition-completed callbacks MUST be supported.
- CORE-12 Activation and deactivation hooks MUST be accepted and follow the lifecycle in DD-10.
- CORE-13 Actions and notifications MUST run in the order defined in DD-08.

**Payloads and results**
- CORE-14 A trigger MUST be able to carry a typed payload available to guards and actions without
  casting; a payload of the wrong type is refused with a clear message.
- CORE-15 `Fire` MUST return the resulting transition (source, destination, trigger, payload, kind,
  time) as described in DD-11 and DD-12.

**Validation and semantics**
- CORE-16 Configuration errors — unknown destination, duplicate registration for the same state and
  trigger, a guard without a destination, missing initial state — MUST fail fast at configuration
  time with messages naming the state and trigger.
- CORE-17 Configuration MUST be closed after the first trigger is fired.
- CORE-18 Firing an unhandled trigger under the default `Throw` policy MUST throw a message that names
  the current state and trigger (other policies: feature 05).
- CORE-19 A trigger fired from inside an action MUST be queued behind the current firing (DD-09).
- CORE-20 A failure in a guard or action MUST leave the machine consistent (DD-07).
- CORE-21 Immediate mode MUST be single-threaded and reentrancy-protected.
- CORE-22 The machine MUST hold no static mutable state and no dependence on ambient time or culture.

## Edge cases and failure modes
- CORE-E1 A state with no transitions (terminal) accepts no trigger without error handling.
- CORE-E2 A trigger with no registration in the current state is "unhandled", distinct from "blocked
  by guards".
- CORE-E3 A guard that always fails, and a guard that throws, are different outcomes (blocked vs
  fault).
- CORE-E4 Two permits for the same trigger with identical guards are a duplicate registration.
- CORE-E5 A dynamic destination that returns an unconfigured state fails clearly at firing time.
- CORE-E6 Null triggers, null states, and null payloads are rejected or defined explicitly.
- CORE-E7 State types with custom equality (records, classes) behave consistently; mutable state
  objects are documented as unsafe.
- CORE-E8 A permit whose destination is its own source: whether this is reentry or an error must be
  stated explicitly (see open questions).
- CORE-E9 An exit action that fires a trigger, an entry action that fires a trigger, and a guard that
  fires a trigger.
- CORE-E10 An action that throws after the position is committed keeps the new position (DD-07).
- CORE-E11 Firing from a subscriber callback while a transition is in progress.
- CORE-E12 Extremely long trigger chains started from actions do not overflow the stack.

## Use-case coverage
UC-1, UC-5 (reentry, internal), UC-7, UC-10 — see [use-cases.md](../use-cases.md).

## Design decisions relied on
DD-06, DD-07, DD-08, DD-09, DD-10, DD-11, DD-12.

## Open questions (for `/speckit-clarify`)
- Is a permit to the same state a reentry or a configuration error?
- The final closed set of transition kinds (DD-11).
- Shape of the payload-carrying trigger wrapper (DD-02).
- Are activation/deactivation hooks required at this stage or only from feature 07?

## Out of scope
Async actions and guards (05), observable streams and observer input (03), introspection queries
(04), hierarchy (07), timers (06), queued mode (10).
