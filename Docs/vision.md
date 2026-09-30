# Vision, Problem and Goals

## Summary

RxStateMachine is a **generic, type-safe, reusable state machine library** for .NET that can be shared
across many projects. Unlike most existing .NET state machine libraries — which are built around
**events** and imperative `Fire()` calls — it is designed **around observables** (`IObservable<T>` /
System.Reactive). The state machine is:

- a **producer** — it exposes its current state, transitions, guard results, and errors as hot
  observable streams that any consumer can subscribe to with LINQ operators; and
- a **consumer** — triggers can be pushed imperatively (`Fire(...)`) **and/or** wired directly from
  other observable streams (message buses, UI events, timers, webhooks).

This "hybrid" design gives the best of both worlds: the **readability, async support, and
debuggability** of a classic state machine core, plus the **composition power** of Rx for consumers
(`DistinctUntilChanged`, `Throttle`, `Buffer`, `CombineLatest`, scheduler control, hot/cold semantics,
and painless UI/telemetry/persistence binding). The hybrid architecture is the baseline; see
[observable-model.md](observable-model.md).

The library is informed by the most popular existing C# state machine libraries on NuGet (see
[references.md](references.md)), adopting many of their most-used features. It takes a different
approach: the machine is natively observable, which simplifies reactive usage and supports complex
async scenarios (see [design-rationale.md](design-rationale.md)).

## Vision

> **"Define your state machines in a few lines of type-safe C#; subscribe to state and transitions
> like any other reactive stream; and wire triggers from anything that emits values — or just call
> `Fire()` when that's simpler."**

A single NuGet package (`RxStateMachine`) that any .NET project can reference, with no dependency on
any application framework (UI, DI, messaging, or storage), so it works identically in console apps,
services, Blazor/WPF/MAUI clients, IoT gateways, and backend microservices.

## The problem

Business and technical domains are full of finite-state processes: order lifecycles, payment flows,
approval workflows, device connection lifecycles, job processing, UI wizards, saga orchestration.
Teams typically hand-roll ad-hoc enums and `switch` statements, which quickly become unmaintainable
and un-testable as guards, side effects, and async steps accumulate.

## A different approach

The popular existing libraries are typically built on an **event/subscription** or **callback**
model, or are tied to an external messaging stack.

Modern .NET applications already use **System.Reactive** (Rx.NET) for UI binding, telemetry,
message-bus plumbing, and stream processing. RxStateMachine takes a **different approach**: the state
machine is _natively_ observable, so it composes directly with that ecosystem — no adapters, no
manual `Subject` wiring, no missed updates — while keeping the familiar fluent configuration style
that state machine users expect. The aim is to simplify reactive usage and support complex async
scenarios.

## Goals

- **G1 — Reusable across projects.** One framework-agnostic NuGet package; no app-framework
  dependencies; used the same way in console, web, desktop, mobile, and IoT code.
- **G2 — Observable-first.** The machine is a first-class `IObservable<TState>` producer and supports
  observable inputs, so it composes with Rx without adapters.
- **G3 — Type-safe.** `TState`/`TTrigger` generics, typed transition payloads, nullable annotations,
  and fail-fast configuration validation catch mistakes at compile time or at startup — not in
  production.
- **G4 — Feature parity with the majority of most-used constructs** of the .NET state machine
  ecosystem (entry/exit/transition actions, guards, internal/reentrant transitions, parameterized
  triggers, async, introspection, hierarchical states, persistence hooks).
- **G5 — Low-friction adoption.** Familiar fluent API, minimal ceremony, excellent XML docs, and
  copy-paste samples for the common scenarios.
- **G6 — Production-grade.** Deterministic, testable (scheduler-injectable), thread-safe story,
  documented error/exception policies.

## Objectives (measurable)

How each target is measured is defined in [design-decisions.md](design-decisions.md) (DD-22).

- **O1.** Deliver the complete feature set, in the order given in [roadmap.md](roadmap.md), with the
  hybrid observable API. There is no time constraint; the final deliverable is the only milestone.
- **O2.** Achieve feature coverage of the most-used surface of the .NET state machine ecosystem
  (Permit / PermitIf / InternalTransition / PermitReentry / parameterized triggers / OnEntry / OnExit
  / OnTransitioned / GetPermittedTriggers / external state storage / async variants).
- **O3.** 100% of the public API covered by XML docs; at least 80% unit-test line coverage on the core
  engine.
- **O4.** Transitions are allocation-conscious: a simple synchronous transition with no actions
  completes in microseconds and does not allocate on the hot path (benchmarked).
- **O5.** Provide at least 8 documented, runnable sample scenarios ([use-cases.md](use-cases.md))
  spanning web, IoT, payment, workflow, and UI domains.
- **O6.** Thread-safety: queued mode passes a concurrency stress test (for example, 8 workers × 10,000
  random valid and invalid triggers) with no lost updates or unexpected exceptions.

## Non-goals (explicitly out of scope)

- Not a visual designer or diagramming tool (visualization _export_ is in scope; a visual editor is
  not).
- Not a long-running workflow engine with built-in scheduling or compensation (saga-style usage with
  persistence is supported; see [saga-exploration.md](saga-exploration.md)).
- Not tied to any DI container, ORM, or messaging framework (integration examples only).
- Not a code generator — no source-generated state machines. A compile-time configuration variant is
  not required and may be revisited later.
- Not a replacement for Rx itself; the library builds _on_ System.Reactive and does not re-implement
  it.
