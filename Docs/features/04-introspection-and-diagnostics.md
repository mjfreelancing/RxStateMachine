# 04 — Introspection and Diagnostics

**Builds on:** 02, 03. **Prefix:** `INS`.

## Feature input (paste into `/speckit-specify`)

Let developers and tools ask a machine questions without firing anything: which triggers are permitted
right now (honouring guards), what is the full configured shape of the machine, why was a particular
trigger blocked, and is the machine in a given state. These answers back diagnostics, UIs that
enable/disable buttons, tests, and the diagram export in a later feature.

## Requirements
- INS-01 A query MUST return the triggers permitted in the current state, evaluating guards.
- INS-02 The query MUST be shaped so an async variant can be added without changing it. That variant
  is delivered with async guards in feature 05 (ASY-15), because this feature precedes it and may
  rely only on earlier features (DD-03).
- INS-03 A machine-description query MUST return an immutable description of states, transitions,
  triggers, guards (with descriptions), and action metadata, and the initial state.
- INS-04 Guard descriptions MUST appear in exceptions and in the guard-result stream so "why was this
  blocked?" is answerable.
- INS-05 An "is in state" query MUST report whether the machine is in a given state (containment
  semantics are extended in feature 07).
- INS-06 A query MUST NOT run entry, exit, or transition actions or change any state.
- INS-07 A guard that throws during a query MUST be handled per the error policy (DD-13); until
  feature 05 adds `Observe`, the default `Throw` policy is the only behaviour.
- INS-08 The machine description MUST be the single data source for diagram export (feature 09).

## Edge cases and failure modes
- INS-E1 Terminal state: permitted-triggers is an empty set, not an error.
- INS-E2 A guard with side effects is evaluated by the query; this is documented.
- INS-E3 A dynamic destination trigger is reported as permitted without resolving the destination.
- INS-E4 A query on a disposed machine.
- INS-E5 A query made from inside an action while a transition is in progress.
- INS-E6 The description of a machine with a single state and no transitions.
- INS-E7 Descriptions remain stable after further configuration is rejected (configuration is
  closed).

## Use-case coverage
UC-6, UC-7 (permitted triggers drive UI) — see [use-cases.md](../use-cases.md).

## Design decisions relied on
DD-06, DD-13, DD-15.

## Open questions (for `/speckit-clarify`)
- Should the description include payload types and action names?
- Is the description a live view or a point-in-time snapshot?
- Do guard evaluations performed by a query appear on the guard-result stream (OBS-04)? Proposed:
  no — that stream reports evaluations made while firing, so a UI that polls permitted triggers does
  not flood it; a caller who needs the detail gets it in the query's own result.

## Out of scope
Export formats (09), hierarchy containment (07).
