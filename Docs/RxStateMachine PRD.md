# RxStateMachine — Product Requirements Document

> **Status:** Draft </br>
> **Last Updated:** 2026-08-14 </br>
> **Target frameworks:** `netstandard2.0` + `net10.0` </br>
> **Core dependency:** System.Reactive (Rx.NET)

---

## 1. Executive Summary

We want a **generic, type-safe, reusable state machine library** for .NET that can be shared across many
projects. Unlike most existing .NET state machine libraries — which are built around **events** and
imperative `Fire()` calls — this framework is designed **around observables** (`IObservable<T>` /
System.Reactive). The state machine is:

- a **producer** — it exposes its current state, transitions, guard results, and errors as hot
  observable streams that any consumer can subscribe to with LINQ operators; and
- a **consumer** — triggers can be pushed imperatively (`Fire(...)`) **and/or** wired directly from
  other observable streams (message buses, UI events, timers, webhooks).

This "hybrid" design (see §5) gives us the best of both worlds: the **readability, async support, and
debuggability** of a classic state machine core, plus the **composition power** of Rx for consumers
(`DistinctUntilChanged`, `Throttle`, `Buffer`, `CombineLatest`, scheduler control, hot/cold semantics,
and painless UI/telemetry/persistence binding).

The library will be informed by the most popular existing C# state machines on NuGet —
[Stateless](https://www.nuget.org/packages/Stateless), [Appccelerate.StateMachine](https://www.nuget.org/packages/Appccelerate.StateMachine),
[Automatonymous/MassTransit](https://www.nuget.org/packages/Automatonymous), and
[WorkflowCore](https://www.nuget.org/packages/WorkflowCore) — adopting their most-used features and
filling the observable gap (see §6).

---

## 2. Vision

> **"Define your state machines in a few lines of type-safe C#; subscribe to state and transitions
> like any other reactive stream; and wire triggers from anything that emits values — or just call
> `Fire()` when that's simpler."**

A single NuGet package (`RxStateMachine`) that any .NET project can reference, with no
dependency on any application framework (UI, DI, messaging, or storage), so it works identically in
console apps, services, Blazor/WPF/MAUI clients, IoT gateways, and backend microservices.

---

## 3. Background & Problem Statement

### 3.1 The problem

Business/technical domains are full of finite-state processes: order lifecycles, payment flows,
approval workflows, device connection lifecycles, job processing, UI wizards, saga orchestration.
Teams typically hand-roll ad-hoc enums + `switch` statements, which quickly become unmaintainable and
un-testable as guards, side effects, and async steps accumulate.

### 3.2 Why not just use an existing library?

The popular existing libraries are excellent but are built on an **event/subscription** model:

- **Stateless** exposes `OnTransitioned(...)`, `OnTransitionCompleted(...)` and requires manual
  bookkeeping to observe state.
- **Appccelerate** uses an `IExtension` interface with many lifecycle callbacks.
- **Automatonymous** is tied to the MassTransit messaging stack.

All of them treat *observing* the machine as a secondary concern bolted on after the fact. Modern .NET
applications already use **System.Reactive** (Rx.NET) for UI binding, telemetry, message-bus plumbing,
and stream processing. A state machine that is *natively* observable composes directly with that
ecosystem: no adapters, no manual `Subject` wiring, no missed updates.

### 3.3 Why observables (and why hybrid)?

There are two extreme designs:

| | **Purely Reactive** (state derived via `Scan`) | **Hybrid** (classic FSM core + observable API) |
|---|---|---|
| State storage | Derived from the event stream history | Explicit, owned by the machine |
| Complex async side effects (I/O, retries, DB) | Hard — `SelectMany` spaghetti, race-prone | Natural `async`/`await` in entry/exit/transition actions |
| Time-based operators (debounce, throttle, timer) | Exceptional, built-in | Available to *consumers* of the streams; helpers for timers |
| Debugging | Deep Rx pipelines, scheduler issues | Standard C# stack traces in the core |
| Guard logic & error handling | Awkward inside stream operators | First-class, with clear policies |
| UI / telemetry / persistence binding | Seamless | Seamless (that's the point of the observable API) |
| Ideal for | High-frequency input streams, games, IoT telemetry | Domain services, workflows, payments, most line-of-business |

**Decision (recommended):** a **hybrid** architecture. The transition engine is a proper state machine
(guards, entry/exit/transition actions, async support, validation, introspection), and **everything
observable flows through native `IObservable<T>` streams**. This is the most *reusable* choice because
it doesn't force consumers into pure-Rx patterns they may not want, while fully enabling them when
they do. §5.2 shows the three options and the recommendation.

---

## 4. Goals & Objectives

### 4.1 Goals (what we want to be true)

- **G1 — Reusable across projects.** One framework-agnostic NuGet package; no app-framework
  dependencies; used the same way in console, web, desktop, mobile, and IoT code.
- **G2 — Observable-first.** The machine is a first-class `IObservable<TState>` producer and supports
  observable inputs, so it composes with Rx without adapters.
- **G3 — Type-safe.** `TState`/`TTrigger` generics, typed transition payloads, nullable annotations,
  and fail-fast configuration validation catch mistakes at compile time or at startup — not in
  production.
- **G4 — Feature parity with the "80% most-used" constructs** from Stateless/Appccelerate
  (entry/exit/transition actions, guards, internal/reentrant transitions, parameterized triggers,
  async, introspection, hierarchical states, persistence hooks).
- **G5 — Low-friction adoption.** Familiar fluent API, minimal ceremony, excellent XML docs, and
  copy-paste samples for the common scenarios.
- **G6 — Production-grade.** Deterministic, testable (scheduler-injectable), thread-safe story,
  documented error/exception policies.

### 4.2 Objectives (measurable)

- **O1.** Ship v1 with the "Core" milestone features (§9) and the hybrid observable API.
- **O2.** Achieve feature coverage of Stateless' most-used surface (Permit / PermitIf /
  InternalTransition / PermitReentry / parameterized triggers / OnEntry / OnExit / OnTransitioned /
  GetPermittedTriggers / external state storage / async variants) — v1.
- **O3.** 100% of public API covered by XML docs; ≥ 80% unit-test line coverage on the core engine.
- **O4.** Transitions are allocation-conscious: a simple synchronous transition with no actions should
  complete in microseconds and not allocate on the hot path (benchmarked with BenchmarkDotNet).
- **O5.** Provide ≥ 8 documented, runnable sample scenarios (§8) spanning web, IoT, payment, workflow,
  and UI domains.
- **O6.** Thread-safety: the "queued/active" mode passes a concurrency stress test (e.g., 8 workers ×
  10k random valid/invalid triggers) with no lost updates or exceptions.

### 4.3 Non-goals (explicitly out of scope)

- Not a visual designer / diagramming tool (visualization *export* is in scope later; a visual editor
  is not).
- Not a long-running workflow engine with built-in scheduling/compensation like WorkflowCore
  (though saga-style usage with persistence is supported).
- Not tied to any DI container, ORM, or messaging framework (integration examples only).
- Not a code generator (no source-generated state machines — not currently required; see §14.4).
- Not a replacement for Rx itself; we build *on* System.Reactive, not re-implement it.

---

## 5. The Observable Model (Producer / Consumer)

This is the heart of the design and the decision that needed the most thought. Here is how we reason
about it.

### 5.1 Core mental model

An `IObservable<T>` is a **push-based, potentially-lazy stream**: a **producer** pushes `OnNext`
values, `OnError`, and `OnCompleted` to **consumers** (observers), who react with LINQ-style operators.

A state machine sits in the middle of two flows:

```mermaid
flowchart LR
    subgraph Upstream["Upstream (producers) — anything that emits triggers"]
        U1["User input / clicks"]
        U2["Message bus / webhooks"]
        U3["Timers / timeouts / heartbeats"]
        U4["Explicit code: Fire()"]
    end
    subgraph SM["StateMachine&lt;TState,TTrigger&gt;"]
        C["Consumer side: accepts triggers<br/>(Fire / IObserver input)"]
        E["Transition engine<br/>guards · actions · async · validation"]
        P["Producer side: emits observables"]
    end
    subgraph Downstream["Downstream (consumers) — subscribe with LINQ"]
        D1["UI (Blazor/WPF/MAUI)"]
        D2["Telemetry / logging"]
        D3["Persistence (persist on change)"]
        D4["Other services / sagas"]
    end
    U1 --> C
    U2 --> C
    U3 --> C
    U4 --> C
    C --> E --> P
    P --> D1
    P --> D2
    P --> D3
    P --> D4
```

- **Producer side (outputs):** the machine exposes streams for *current state*, *transitions*, *guard
  results*, *permitted triggers*, and *errors*.
- **Consumer side (inputs):** triggers arrive either imperatively (`Fire(...)`) or by piping an
  upstream observable into the machine (e.g., `messageBus.Observe<OrderEvent>().Subscribe(machine)`).

### 5.2 The three candidate designs

| | **A. Outputs only** | **B. Fully reactive** | **C. Hybrid (recommended)** |
|---|---|---|---|
| State/transition outputs | `IObservable<T>` | `IObservable<T>` | `IObservable<T>` |
| Trigger input | `Fire(...)` only | Machine implements `IObserver<TTrigger>`; state via `Scan` | Both `Fire(...)`/`FireAsync(...)` **and** `IObserver` input |
| Async side effects | Natural | Awkward (`SelectMany` nesting) | Natural (`async`/`await`) |
| Time-based composition for consumers | Via streams | Exceptional | Via streams |
| Learning curve | Low | High | Low–Medium |
| Reusability / flexibility | High | Medium (opinionated) | **Highest** |
| Risk | Observers miss updates if wired by hand | Debugging & error-handling complexity | Low (classic core + observable API) |

**Recommendation: Option C — Hybrid.** The transition engine is a conventional, well-tested state
machine (like Stateless) so business logic stays readable and async-friendly; the machine *natively*
implements `IObservable<TState>` (producer) and also accepts observable trigger streams (consumer).
Nothing forces users into pure-Rx — but everything is available if they want it.

> ⚠️ **Open decision (see §14):** confirm Option C after reviewing the worked examples in §5.4.
> This is the one architectural choice we'd like explicit sign-off on before Milestone 1.

### 5.3 Streams the machine exposes (v1)

| Stream | Type | Semantics |
|---|---|---|
| Current state | `IObservable<TState>` (machine itself) | Hot, replays last (`BehaviorSubject`-backed) so late subscribers get the current state |
| State changed | `IObservable<StateChange<TState>>` | `Previous`, `Current`, `Timestamp`, `IsReentry`, `IsInternal` |
| Transition | `IObservable<Transition<TState,TTrigger>>` | `Source`, `Destination`, `Trigger`, payload, kind, duration |
| Transition completed | `IObservable<Transition<TState,TTrigger>>` | Fired after last entry action completes (Stateless parity) |
| Guard result | `IObservable<GuardResult<TState,TTrigger>>` | Every guard evaluation (for logging, tests, and "why blocked?" UI) |
| Permitted triggers | `IObservable<PermittedTriggers<TState,TTrigger>>` | Re-emitted when the state changes; also queryable via `GetPermittedTriggers()` |
| Errors | `IObservable<StateMachineError>` | Unhandled triggers, guard/action exceptions, configuration errors (depending on policy) |

### 5.4 Worked examples (to aid the product decision)

**Example 1 — Producer only (most common):** a checkout UI wants to drive a progress bar and disable
buttons. The consumer needs **no** trigger wiring.

```csharp
var machine = new StateMachine<OrderState, OrderTrigger>(OrderState.Draft);

machine.Configure(OrderState.Draft)
    .Permit(OrderTrigger.Submit, OrderState.Submitted);

machine.Configure(OrderState.Submitted)
    .PermitIf(OrderTrigger.Pay, OrderState.Paid, () => _payment.Succeeded, "payment succeeded")
    .Permit(OrderTrigger.Cancel, OrderState.Cancelled);

// Producer usage: pure LINQ over state
machine
    .Where(s => s is OrderState.Paid or OrderState.Cancelled)
    .Subscribe(_ => NotifyAccounting());

machine.Subscribe(state => progressBar.Value = ProgressFor(state)); // BehaviorSubject replays current state

machine.Fire(OrderTrigger.Submit);      // imperative input
```

**Example 2 — Fully reactive input (same machine, no manual Fire):** wire a payment gateway's events
straight into the machine.

```csharp
paymentGateway.PaymentSucceeded        // IObservable<PaymentReceipt>
    .Select(receipt => new TriggerWithParameters<OrderTrigger, PaymentReceipt>(OrderTrigger.Pay, receipt))
    .Subscribe(machine);               // machine is IObserver<...>

// A timeout that fires a trigger if the customer stalls
Observable.Timer(TimeSpan.FromMinutes(10))
    .Select(_ => OrderTrigger.Cancel)
    .Subscribe(machine);
```

**Example 3 — The "pure Rx Scan" alternative** (what we are *not* building as the core, for
comparison):

```csharp
_state = _events
    .Scan(OrderState.Draft, (state, e) => (state, e) switch
    {
        (OrderState.Draft, OrderTrigger.Submit) => OrderState.Submitted,
        _ => state
    })
    .DistinctUntilChanged().Replay(1).RefCount();
```

This is elegant but breaks down when transitions must run **async I/O, retries, or database calls**
inside guards/actions, and it has no place for entry/exit side effects, guard descriptions, or
"why is this blocked?" introspection. Our hybrid keeps all of that while still exposing the same
`IObservable<TState>` to consumers.

---

## 6. Reference Landscape (Existing NuGet State Machines)

Used as **guidelines** for feature completeness, not as dependencies.

| Feature | Stateless | Appccelerate | Automatonymous (MassTransit) | WorkflowCore | **This framework (target)** |
|---|---|---|---|---|---|
| Downloads (approx.) | ~33.6M | ~2.2M | ~52M | ~4.8M | — |
| Generic states/triggers (any type) | ✅ | ✅ | enum-based | string/JSON | ✅ (any type) |
| Fluent config (`Permit`, `PermitIf`) | ✅ | ✅ (`.If().Goto()`) | ✅ | ✅ | ✅ |
| Entry/exit actions (sync + async) | ✅ | ✅ | ✅ | ✅ | ✅ |
| Transition actions | ✅ (internal) | ✅ | ✅ | ✅ | ✅ |
| Guards / guard descriptions | ✅ | ✅ | ✅ | ✅ | ✅ |
| Parameterized triggers (typed payloads) | ✅ | ✅ (parametrized actions) | ✅ | ✅ | ✅ |
| Internal transitions (no exit/entry) | ✅ | ✅ | ~ | ✅ | ✅ |
| Reentrant transitions | ✅ | ~ | ~ | ~ | ✅ |
| Dynamic destination (`destinationStateSelector`) | ✅ | ✅ | ~ | ✅ | ✅ |
| Hierarchical (super/sub) states | ✅ | ✅ | ✅ | ✅ | ✅ (M2) |
| History (shallow/deep) | ~ | ✅ | ~ | ~ | ✅ (M2) |
| Async (`FireAsync`, async guards/actions) | ✅ | ✅ | ✅ | ✅ | ✅ |
| Introspection / permitted triggers | ✅ | ✅ | ✅ | ✅ | ✅ |
| Machine info / report (text, DOT, Mermaid, D2, CSV, yEd) | DOT + Mermaid | text/CSV/yEd + custom | — | — | ✅ (Mermaid + D2, M3) |
| External state storage (accessor/mutator) | ✅ | ✅ (persist whole machine) | ✅ (saga DB) | ✅ | ✅ (M2) |
| Threading model | Immediate / Queued | Passive / Active (thread-safe) | queue-based | queue-based | Immediate / Queued (M2) |
| Timer / timeout driven transitions | ❌ (manual) | ❌ (manual) | ✅ (saga schedules) | ✅ | ✅ via Rx (M2) |
| Extension points / lifecycle hooks | events | `IExtension` | message handlers | events | observable streams + optional observer base |
| **Observable-first API** | ❌ | ❌ | ❌ | ❌ | ✅ (our differentiator) |

### 6.1 Feature ideas we borrow from these libraries

1. **Stateless** → `FiringMode.Immediate` vs `.Queued`; `OnTransitioned` / `OnTransitionCompleted`;
   external state storage via `Func<TState>`/`Action<TState>`; `StateMachineInfo` introspection;
   graph export (Stateless pioneered DOT/Mermaid; **we scope to Mermaid + D2** — product decision);
   guard descriptions for rich error messages; `TriggerWithParameters<TArg>`.
2. **Appccelerate** → hierarchical states with `HistoryType.None / Shallow / Deep`; **active**
   (thread-safe, worker-thread) vs **passive** machines; extension callbacks around the full lifecycle
   (entering/entered state, firing/fired event, guard/action exceptions); persistence of current state
   + queued events + history; reporting.
3. **Automatonymous/MassTransit** → saga-style usage; scheduling/timeouts (`Schedule`/`Publish`);
   state-machine-as-persistence-subject; correlation.
4. **WorkflowCore** → persistence-first design, long-running workflows; we deliberately keep this
   lighter, but support snapshot persistence.

---

## 7. Requirements

### 7.1 Functional requirements (v1 = Core, phased by §9)

#### Configuration & core semantics
- **FR-1** Generic over `TState` and `TTrigger` — any .NET type (enum, record, string, int, custom
  class). 
- **FR-2** Fluent builder: `Configure(TState)` returning a configuration object with
  `Permit`, `PermitIf`, `PermitReentry`, `InternalTransition`, and dynamic destination selectors.
- **FR-3** Entry actions (`OnEntry`), exit actions (`OnExit`), trigger-specific entry actions
  (`OnEntryFrom(trigger, ...)`), activation/deactivation actions (Stateless parity).
- **FR-4** Transition actions (`OnTransitioned`, `OnTransitionCompleted`).
- **FR-5** Guards with optional human-readable descriptions; multiple guards evaluated in
  registration order; an `otherwise`/default transition concept.
- **FR-6** Parameterized transitions: `TriggerWithParameters<TTrigger, TPayload>` carrying a typed
  payload through the transition (available to actions/guards and the transition stream).
- **FR-7** Internal transitions (side effect, no exit/entry, no state change).
- **FR-8** Reentrant transitions (re-run exit+entry without leaving, Stateless `PermitReentry`).
- **FR-9** Async variants of all of the above (`OnEntryAsync`, `PermitIfAsync`, `FireAsync`).
- **FR-10** Configuration-time validation: unknown destination, duplicate transition registration,
  guard without a destination, missing initial state, invalid hierarchy — fail fast with actionable
  messages.

#### Observable surface
- **FR-11** Machine implements `IObservable<TState>`; hot, replays last (late subscribers get current
  state immediately).
- **FR-12** Rich streams from §5.3: `StateChange<TState>`, `Transition<TState,TTrigger>`,
  `GuardResult`, `PermittedTriggers`, `StateMachineError`.
- **FR-13** All streams are cold-safe (subscribe/unsubscribe cleanly), disposable, and compose with
  standard Rx operators.
- **FR-14** Machine accepts triggers as `IObserver` (`.Subscribe(machine)` from any observable) **and**
  imperatively via `Fire`/`FireAsync` (hybrid).
- **FR-15** Scheduler integration: an optional scheduler (default `Scheduler.Default`) determines the
  thread on which transition notifications are raised; consumers can also `.ObserveOn(...)` any
  stream themselves.

#### Introspection & diagnostics
- **FR-16** `GetPermittedTriggers()` / async variant, honoring guards.
- **FR-17** `GetInfo()` / `StateMachineInfo` describing states, transitions, guards, and action
  metadata (Stateless parity) — the basis for diagnostics and graph export.
- **FR-18** Guard descriptions surfaced in exceptions and in a `GuardResult` stream (answers "why is
  this transition blocked?").

#### Error handling & policies
- **FR-19** Unhandled-trigger policy: `Throw` (default), `Ignore`, or `Observe` (push to error
  stream). Same policy set for exceptions thrown in guards and actions.
- **FR-20** `OnTransitioned`-style lifecycle events with unsubscribe support.

#### External state / persistence (M2)
- **FR-21** External state storage via `Func<TState> stateAccessor` / `Action<TState> stateMutator`
  (Stateless parity) so state can live in an ORM entity.
- **FR-22** Snapshot persistence: serialize current state, active substate history, and (in queued
  mode) pending triggers (Appccelerate parity) via a small `IStateMachinePersistence` interface.
- **FR-23** "Persistence as an observer": because state changes are observable, persisting is simply
  `machine.Subscribe(state => repository.Save(instanceId, state))` — no special API required.

#### Timers & timeouts (M2, Rx-native)
- **FR-24** Helpers to trigger transitions from timers/delays (`Observable.Timer(...).Subscribe(machine)`),
  plus convenience API for common patterns: "after N in state X, fire trigger Y" with automatic
  subscription cancellation when the state is left.

#### Hierarchical states (M2)
- **FR-25** `SubstateOf(superstate)`, `InitialTransitionTarget`, history types `None/Shallow/Deep`
  (Appccelerate parity).
- **FR-26** `IsInState(superstate)` returns true when in any substate; superstate exit/entry actions
  run at the correct points in the nesting.

#### Visualization & reporting (M3)
- **FR-27** Export configuration to **Mermaid** and **D2** for docs/PRs (product decision: no DOT).
- **FR-28** Optional textual report of states/transitions/actions (Appccelerate-style).

### 7.2 Non-functional requirements

- **NFR-1** No dependencies beyond `System.Reactive` (and standard BCL). No app-framework, DI, or
  messaging dependencies.
- **NFR-2** Nullable reference types enabled; `notnull` constraints where appropriate; `[Obsolete]`
  policy for API evolution.
- **NFR-3** Thread-safety: immediate mode is single-threaded and reentrancy-protected; queued/active
  mode is safe for concurrent `Fire` from multiple threads (M2).
- **NFR-4** Async-first: never block on async work (no `.Result`/`.Wait()` in library code).
- **NFR-5** Performance: zero or minimal allocation on the simple-transition hot path; configuration
  is one-time; streams should not allocate per-subscription unnecessarily.
- **NFR-6** Determinism & testability: injectable scheduler, no ambient time dependence, helpers for
  virtual time (Rx `TestScheduler`).
- **NFR-7** Documentation: XML docs on all public members; API docs site (DocFX or similar); ≥ 8
  runnable samples.
- **NFR-8** Compatibility targets: **netstandard2.0** + **net10.0** (decided; net8.0 omitted — nearing end-of-life).
  (§14.2) for maximum reuse across older projects.
- **NFR-9** Semantic versioning; clean public API surface with an explicit public/`internal` boundary.
- **NFR-10** Cancellation: `CancellationToken` support on long-running/async transition operations.

---

## 8. Example Use Cases (Product-Owner Confidence)

These scenarios span the domains our consumers actually build. Each maps to concrete FRs. The PRD
treats these as the **definition of "most, if not all, business requirements"** — if a new requirement
looks like one of these, the framework covers it.

| # | Use case | Domain | Key framework features exercised |
|---|---|---|---|
| UC-1 | **Order / checkout lifecycle** | E‑commerce | Basic permits, guards, entry actions, state observable → progress UI, persist-on-change, cancel paths, Mermaid/D2 diagram export (see §15.2) |
| UC-2 | **Payment & refund processing** | Payments / fintech | Async entry actions (gateway call), guarded transitions, timeout → auto-cancel, parameterized payload (receipt), error stream |
| UC-3 | **Approval / document workflow** | Content / HR | Guard "N approvals collected", escalation via timer trigger, `IsInState` for draft-of-published, history |
| UC-4 | **Device lifecycle & provisioning** | IoT | Connection states, heartbeat timeouts (observable triggers), reconnection with backoff (timer), telemetry throttling via Rx |
| UC-5 | **Socket / session connection FSM** | Networking | Reentrant transitions (reconnect), `InternalTransition` (heartbeat ping), queued firing under load, thread-safety |
| UC-6 | **Background job / pipeline processing** | DevOps / data | Retry-with-backoff (timer triggers), max-retry guard, async actions (worker calls), permitted-triggers UI |
| UC-7 | **UI wizard / multi-step form** | Web/desktop | State observable drives step rendering; guard enables "Next"; `OnEntryFrom` runs per-step logic; parameterized payload |
| UC-8 | **Saga orchestration (MassTransit-style)** | Distributed systems | External state storage + snapshot persistence, async actions, error policies, timeouts; survives restarts |
| UC-9 | **Game / player state machine** | Games | High-frequency observable input, debounce/throttle, immediate firing, testable with virtual time |
| UC-10 | **Media player / playback control** | Media | Parameterized transitions (seek position), internal transitions (volume), guarded transitions (buffered?) |

### 8.1 Two detailed examples (how a product owner reads confidence)

**UC-2 — Payment processing** shows async + timeout + persistence:

```csharp
var machine = new StateMachine<PaymentState, PaymentTrigger>(PaymentState.Initiated);

machine.Configure(PaymentState.Initiated)
    .OnEntry(_ => _ = AuthorizeAsync())                    // async side effect on entry
    .Permit(PaymentTrigger.Authorized, PaymentState.Authorized)
    .Permit(PaymentTrigger.Failed, PaymentState.Failed);

machine.Configure(PaymentState.Authorized)
    .Permit(PaymentTrigger.Capture, PaymentState.Captured)
    .Permit(PaymentTrigger.Refund, PaymentState.Refunded);

// Timeout: if still Authorizing after 30s, treat as Failed
Observable.Timer(TimeSpan.FromSeconds(30))
    .Select(_ => PaymentTrigger.Failed)
    .Subscribe(machine);

// Telemetry + persistence are just observers
machine.Subscribe(state => _metrics.StateChanged(state));
machine.Subscribe(state => _repo.Save(transactionId, state));
```

**UC-5 — Connection FSM** shows reentry, internal transitions, and thread-safe queued firing:

```csharp
machine.Configure(ConnectionState.Connecting)
    .Permit(ConnectionTrigger.Connected, ConnectionState.Open)
    .Permit(ConnectionTrigger.Failed, ConnectionState.Retrying);

machine.Configure(ConnectionState.Open)
    .InternalTransition(ConnectionTrigger.Heartbeat, _ => _lastSeen = DateTimeOffset.UtcNow) // no exit/entry
    .Permit(ConnectionTrigger.Lost, ConnectionState.Retrying);

machine.Configure(ConnectionState.Retrying)
    .PermitReentry(ConnectionTrigger.Retry)               // re-run entry (backoff reset)
    .Permit(ConnectionTrigger.Connected, ConnectionState.Open);
```

---

## 9. Feature Complexity Analysis & Delivery Plan

The product owner asked which features carry the most complexity so we can phase sensibly. High-level
guidance:

- **Low complexity (core, ship first):** basic permits, entry/exit/transition actions, internal &
  reentrant transitions, parameterized payloads, observable outputs, `Fire`, config validation.
- **Medium complexity:** guards + guard streams, async guards/actions, queued firing, introspection,
  external state storage, scheduler integration, error policies, timer helpers.
- **High complexity (do last):** hierarchical states + history semantics, thread-safe active mode,
  snapshot persistence of the full machine, visualization export, deep extension hooks.

### 9.1 Milestones

| Milestone | Theme | Delivered (FRs) | Complexity | Target |
|---|---|---|---|---|
| **M0** | Foundations | Project scaffolding, package skeleton, CI, docs site, benchmarks harness | Low | Sprint 1 |
| **M1 — Core** | The "widely used 80%" | FR-1..8, FR-10..14 (sync), FR-16..17, FR-19 (throw policy), samples UC-1/5/7/9 | Low–Med | Sprint 2–3 |
| **M2 — Async & power** | Async, payloads, policies, scheduling, timers, storage | FR-9, FR-15, FR-18, FR-19 (all policies), FR-20, FR-21, FR-24, FR-14 (observer input fully), queued firing | Medium | Sprint 4–6 |
| **M3 — Hierarchy & persistence** | Hierarchical states + snapshot persistence | FR-22, FR-23, FR-25, FR-26, FR-28, Mermaid/D2 export | Med–High | Sprint 7–9 |
| **M4 — Hardening** | Thread-safe active mode, perf, stress tests, docs, final samples | NFR-3, NFR-5, NFR-6, remaining | High | Sprint 10–12 |

### 9.2 Why this order

1. **M1 targets the most-used features** so the library is useful as early as possible and provides
   feedback for the riskier M2/M3 designs.
2. **Async + observer input (M2)** builds directly on the M1 core and is where most real-world
   integrations live (webhooks, gateways, buses).
3. **Hierarchy/history (M3)** is the single largest source of subtle bugs in every reference library;
   it benefits from a battle-tested M1/M2 core and a solid test suite first.
4. **Hardening (M4)** focuses on thread-safety and performance only after the semantics are stable.

---

## 10. Architecture & Technical Design (High-Level Sketch)

### 10.1 Package & namespaces

```
RxStateMachine/                            (library, the deliverable of this PRD)
├── StateMachine{TState,TTrigger}.cs       (implements IObservable<TState> + IObserver<TTrigger>)
├── StateMachineConfiguration.cs           (fluent builder; Configure(...).Permit(...))
├── Transition.cs                          (record: Source, Destination, Trigger, Payload, Kind, Timestamp)
├── StateChange.cs                         (record: Previous, Current, Timestamp, Kind)
├── GuardResult.cs
├── PermittedTriggers.cs
├── StateMachineError.cs
├── TriggerWithParameters.cs
├── StateMachineOptions.cs                 (Scheduler, FiringMode, UnhandledTriggerPolicy, ExceptionPolicy)
├── Reactive/
│   ├── ObservableStateMachineExtensions.cs (DistinctUntilChanged-style helpers, timer helpers)
│   └── StateMachineObserver.cs            (optional convenience base for lifecycle observers)
├── Persistence/  (M2)  IStateMachinePersistence, SnapshotPersistence
├── Diagnostics/  (M3)  MermaidFormatter, D2Formatter, TextReport
└── Validation/   StateMachineValidator (fail-fast config checks)
```

### 10.2 Core type sketch (illustrative API)

```csharp
public sealed class StateMachine<TState, TTrigger> : IObservable<TState>, IObserver<TriggerWithParameters<TState, TTrigger>>
    where TState : notnull
    where TTrigger : notnull
{
    public StateMachine(TState initialState, StateMachineOptions? options = null);
    public StateMachine(Func<TState> accessor, Action<TState> mutator, StateMachineOptions? options = null); // FR-21

    public StateMachineConfiguration<TState, TTrigger> Configure(TState state); // fluent

    public TState State { get; }
    public IObservable<TState> States { get; }                       // == this
    public IObservable<StateChange<TState>> StateChanges { get; }
    public IObservable<Transition<TState, TTrigger>> Transitions { get; }
    public IObservable<Transition<TState, TTrigger>> TransitionsCompleted { get; }
    public IObservable<GuardResult<TState, TTrigger>> GuardResults { get; }
    public IObservable<PermittedTriggers<TState, TTrigger>> PermittedTriggers { get; }
    public IObservable<StateMachineError> Errors { get; }

    public Transition<TState, TTrigger> Fire(TTrigger trigger);
    public Transition<TState, TTrigger> Fire<TArg>(TriggerWithParameters<TTrigger, TArg> trigger, TArg arg);
    public Task<Transition<TState, TTrigger>> FireAsync(TTrigger trigger, CancellationToken ct = default);
    public Task<Transition<TState, TTrigger>> FireAsync<TArg>(TriggerWithParameters<TTrigger, TArg> trigger, TArg arg, CancellationToken ct = default);

    public IEnumerable<TTrigger> GetPermittedTriggers();
    public Task<IEnumerable<TTrigger>> GetPermittedTriggersAsync(CancellationToken ct = default);
    public StateMachineInfo GetInfo();
    public bool IsInState(TState state);                            // M3: honors hierarchy

    // IObserver input (hybrid) — lets any observable drive the machine
    void IObserver<TriggerWithParameters<TState, TTrigger>>.OnNext(TriggerWithParameters<TState, TTrigger> value);
}
```

### 10.3 Configuration surface (illustrative)

```csharp
machine.Configure(OrderState.Draft)
    .Permit(OrderTrigger.Submit, OrderState.Submitted)
    .InternalTransition(OrderTrigger.Touch, _ => OnTouch())
    .OnExit(_ => NotifyLeaving());

machine.Configure(OrderState.Submitted)
    .PermitIf(OrderTrigger.Pay, OrderState.Paid, () => _payment.Succeeded, "payment confirmed")
    .PermitIfAsync(OrderTrigger.Ship, OrderState.Shipping, p => _stock.ReserveAsync(p), "stock reserved")
    .OnEntryFrom(OrderTrigger.Pay, _ => StartShippingTimer())
    .PermitReentry(OrderTrigger.RefreshStatus);
```

### 10.4 Design principles

- **Observable-pure core:** the engine pushes to internal subjects; the public API exposes them
  `AsObservable()`-style (read-only, no external `OnNext`).
- **Validation at configuration time:** a `StateMachineValidator` runs after build to catch most
  errors before runtime.
- **Single source of truth:** the current state lives in one place; observable replays it; no parallel
  copies to drift.
- **No blocking:** all async flows return `Task`; schedulers are injected, never ambient.

---

## 11. Testing & Quality

- **Unit tests** (xUnit) on the engine: transition correctness, guard ordering, reentry, internal
  transitions, async actions/guards, error policies, config validation.
- **Reactive tests** using Rx `TestScheduler` for virtual-time determinism (timers, timeouts, stream
  emission ordering, hot/cold semantics, unsubscribe).
- **Concurrency stress tests** for queued/active mode (M2/M4).
- **Property-based tests** (FsCheck) for state-transition invariants (e.g., "never transition to an
  unconfigured state", "permitted set matches configured transitions given guards").
- **Benchmarks** (BenchmarkDotNet) for the hot path (NFR-5).
- **Sample projects** double as integration tests; each UC in §8 gets a runnable sample.

---

## 12. Risks & Mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| Observable API is over-engineered / confusing | Adoption friction | Hybrid keeps `Fire()` simple; streams are additive. Docs + samples first. |
| Hierarchy/history bugs (all reference libs struggle here) | Correctness | Defer to M3; model on Appccelerate's proven semantics; heavy test coverage. |
| netstandard2.0 costs time (shims, `#if`) | Schedule | Resolved (§14.2): included; gate only modern niceties behind `#if`. |
| Thread-safety surprises in queued mode | Production incidents | Ship passive/immediate first (single-threaded, documented); active mode behind a clear opt-in. |
| Scope creep (full workflow engine) | Schedule | Non-goals (§4.3) kept visible; WorkflowCore-style features explicitly out. |
| Rx dependency perceived as heavy | Adoption | System.Reactive is already ubiquitous; we depend on it directly, no wrapper bloat. |

---

## 13. Success Metrics / KPIs

- Adoption in ≥ 3 distinct projects within 2 quarters of GA.
- Feature-parity checklist against Stateless' most-used surface completed for M1 (§4.2 O2).
- Zero reported race conditions in queued/active mode after the M4 stress suite.
- ≥ 80% core coverage; all samples build and pass in CI.
- Docs site live with API reference + the §8 samples by M4.

---

## 14. Decisions & Open Items

> All previously-open items in §14 are now **decided**. No blockers remain before Milestone 1.

### 14.1 (DECIDED) Observable model — **Option C (Hybrid)**
The machine implements `IObservable<TState>` (producer) **and** accepts observable trigger input
plus `Fire()`/`FireAsync()` (consumer). Confirmed Option C — see §5 for rationale and examples.

### 14.2 (DECIDED) Target frameworks — **netstandard2.0 + net10.0**
- **Definite:** `netstandard2.0` (max reach across projects) and `net10.0` (latest).
- **`net8.0` is omitted** — it is nearing end-of-life and `net10.0` already covers modern consumers.
- Multi-targeting costs: C# feature shims (`IsExternalInit` for `init`/records, `RequiredMemberAttribute`
  for `required`) and a few `#if` gates — no functionality, testability, or API-quality sacrifice.
- **Time & testing (resolved):** `TimeProvider` is not a blocker — keep it out of the public API;
  implement time-based behavior on Rx `IScheduler` and test with `TestScheduler` virtual time;
  use `TimeProvider` only for cosmetic timestamps gated behind `#if NET8_0_OR_GREATER` (defined on
  net10.0 too; fallback `DateTimeOffset.UtcNow`) or the `Microsoft.Bcl.TimeProvider` backport.
- **Async streams (resolved):** `IAsyncEnumerable` is not required — all async APIs are `Task`-based
  and the reactive surface is `IObservable<T>`. If interop is ever wanted, System.Reactive 6.x ships
  `ToObservable`/`ToAsyncEnumerable` and `Microsoft.Bcl.AsyncInterfaces` backports it to
  netstandard2.0.

### 14.3 (DECIDED) API naming & identity
- Package name: **`RxStateMachine`** (not `Rx.StateMachine`, not `ReactiveStateMachine`).
- `Fire`/`FireAsync` **return the resulting `Transition`** — sync `Transition<TState,TTrigger>`,
  async `Task<Transition<TState,TTrigger>>` — for fluent/assertive tests and callers that need the
  transition result (see §10.2).

### 14.4 (DECIDED) Misc
- **No source generation** — a source-generated (compile-time) configuration variant is not currently
  required; remains a non-goal and may be revisited later.
- The `Errors` stream exposes rich **`StateMachineError`** (trigger, state, exception, policy).
  Default error policy: **Throw**; **`Observe` is opt-in** via `StateMachineOptions`.

---

## 15. Appendix

### 15.1 Glossary
- **State machine (FSM):** a model with a finite set of states, one current state, and guarded
  transitions between them triggered by triggers.
- **Trigger:** the input that may cause a transition (called an *event* in Appccelerate/Stateless).
- **Guard:** a predicate that must return true for a transition to be permitted.
- **Observable (`IObservable<T>`):** a push-based, lazily-composed stream of values.
- **Hot vs cold:** a hot observable pushes regardless of subscribers and typically replays/loses
  values; a cold observable produces a fresh sequence per subscriber.
- **Hybrid:** a machine that is both a classic imperative FSM core and a native observable producer +
  observable-input consumer.
- **Reentry:** a transition that exits and re-enters the same state, re-running entry/exit actions.
- **Internal transition:** a side-effect transition that does not change state and runs no exit/entry
  actions.

### 15.2 Example exports (Mermaid & D2)

*Informational only — illustrative output that the M3 formatters (`MermaidFormatter`,
`D2Formatter`) will produce from a machine's configuration graph. Exact syntax and annotation
detail are subject to implementation.*

Using the order lifecycle from UC-1:

```csharp
machine.Configure(OrderState.Draft)
    .Permit(OrderTrigger.Submit, OrderState.Submitted);
machine.Configure(OrderState.Submitted)
    .Permit(OrderTrigger.Pay, OrderState.Paid)
    .Permit(OrderTrigger.Cancel, OrderState.Cancelled);
machine.Configure(OrderState.Paid)
    .Permit(OrderTrigger.Ship, OrderState.Shipped);
machine.Configure(OrderState.Shipped)
    .Permit(OrderTrigger.Deliver, OrderState.Delivered);
```

**Mermaid** (`stateDiagram-v2`) — renders natively in GitHub, GitLab, and VS Code:

```mermaid
stateDiagram-v2
    [*] --> Draft
    Draft --> Submitted : Submit
    Submitted --> Paid : Pay
    Submitted --> Cancelled : Cancel
    Paid --> Shipped : Ship
    Shipped --> Delivered : Deliver
    Delivered --> [*]
```

**D2** (`.d2`) — rendered with the [D2 CLI](https://d2lang.com) (e.g., `d2 order.d2 order.svg`):

```d2
direction: right

start: {
    shape: circle
}
end: {
    shape: circle
}

start -> Draft
Draft -> Submitted: Submit
Submitted -> Paid: Pay
Submitted -> Cancelled: Cancel
Paid -> Shipped: Ship
Shipped -> Delivered: Deliver
Delivered -> end
```

Notes:
- Guards, entry/exit actions, and parameterized payloads can be annotated as edge labels/details
  (Mermaid `:label` and D2 edge labels) — the detail level is a formatter decision.
- No DOT export will be produced (product decision, see FR-27).

### 15.3 References
- Stateless — https://github.com/dotnet-state-machine/stateless
- Appccelerate.StateMachine — https://github.com/appccelerate/statemachine
- Automatonymous / MassTransit — https://masstransit.io/documentation/configuration/sagas/automatonymous
- WorkflowCore — https://github.com/danielgerlag/workflow-core
- System.Reactive (Rx.NET) — https://github.com/dotnet/reactive
