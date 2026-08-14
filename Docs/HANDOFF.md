# RxStateMachine — Session Handoff

> Handoff notes for continuing work in a fresh agent/discussion. The authoritative spec is
> **`RxStateMachine PRD.md`** in this folder — read it fully before implementing.

## What this is
**RxStateMachine**: a generic, type-safe, reusable state machine library for .NET, designed around
observables (System.Reactive). The machine is a producer (`IObservable<TState>` state, transitions,
guard results, permitted triggers, errors) **and** a consumer (triggers via `Fire()`/`FireAsync()` or
piped from any observable).

## Status
- PRD **complete and decisions locked**. Implementation **not started**.
- Next step: scaffold **M0** — `RxStateMachine` class library targeting `netstandard2.0;net10.0`,
  dependency `System.Reactive`, plus a sample console app.

## Key decisions (locked)
- **Hybrid observable model (Option C)**: `IObservable<TState>` producer + observable-input consumer + `Fire()`/`FireAsync()`.
- **Package name:** `RxStateMachine` (not `Rx.StateMachine`, not `ReactiveStateMachine`).
- **Target frameworks:** `netstandard2.0` + `net10.0` (net8 omitted — nearing EOL).
- **`Fire`/`FireAsync` return the resulting `Transition`** (`Task<Transition>` for async).
- **No source generation** currently required (non-goal; may revisit later).
- **`Errors` stream exposes `StateMachineError`** (trigger, state, exception, policy). Default policy **Throw**; `Observe` opt-in via `StateMachineOptions`.
- **Diagram export: Mermaid + D2 only** (no DOT). Example exports: PRD §15.2.
- **Time-based behavior on Rx `IScheduler`; tests with `TestScheduler`** — no `TimeProvider` in the
  public API (timestamps only, gated `#if NET8_0_OR_GREATER` with `DateTimeOffset.UtcNow` fallback).

## API sketch (PRD §10.2)
`StateMachine<TState,TTrigger>` implements `IObservable<TState>` and
`IObserver<TriggerWithParameters<TState,TTrigger>>`; fluent `Configure(TState).Permit/PermitIf/
InternalTransition/PermitReentry/OnEntry/OnExit`; streams `StateChanges`, `Transitions`,
`TransitionsCompleted`, `GuardResults`, `PermittedTriggers`, `Errors`; `GetPermittedTriggers()`,
`GetInfo()`, `IsInState()`.

## Reference libraries (feature guidelines; PRD §6)
Stateless · Appccelerate.StateMachine · Automatonymous/MassTransit · WorkflowCore.

## Roadmap (PRD §9)
M0 foundations → M1 core (widely-used 80%) → M2 async & power → M3 hierarchy & persistence →
M4 hardening. Use-case catalog UC-1..UC-10 (PRD §8) doubles as sample scope.

## Testing approach
Rx `TestScheduler` for deterministic virtual-time tests (NFR-6); unit tests on the engine; property
tests for transition invariants; BenchmarkDotNet for the hot path; the §8 use cases become runnable
samples that double as integration tests.
