# S-00 — Decisions & API Suggestions

## 1. Metadata

- **Sprint ID:** S-00
- **Title:** Decisions & API Suggestions
- **Status:** Draft (awaiting review)
- **PRD references:** FR-1..FR-31 (scan), NFR-1..NFR-11, UC-1..UC-10, PRD §5, §9, §10, §14
- **Prerequisites:** PRD
- **Depends-on (future sprints that need this):** S-01..S-14
- **Owner:** TBD

---

## 2. Outcome

This sprint produces the **starting assumptions** for the entire build: a decision register resolving
the PRD's open questions (D-01..D-10), a **suggested (provisional)** v1 API surface, and environment
defaults — each marked with **how it will be validated** by tests and samples. **No code is written.**
Per the project philosophy, nothing here is a hard contract; every item is a suggestion to be
validated and evolved as we build.

### 2.1 Definition of Done

- [ ] Every blocking decision D-01..D-10 has a recorded working assumption with rationale,
      alternatives, and a "validate via" pointer.
- [ ] Proposed v1 API surface is captured and **explicitly marked provisional**.
- [ ] Environment defaults proposed (repo layout, SDK, CI, package target, test tooling).
- [ ] PRD cross-references completed — no contradictions introduced.
- [ ] The API readability check (T4) against UC-2 and UC-5 is recorded.
- [ ] Decision register in the README is populated and in sync.
- [ ] Reviewed & accepted by the owner (status → Ready; register statuses updated).

---

## 3. Concepts & Vocabulary

> This is the foundational vocabulary for the whole series. Later sprints add their own terms and
> assume these. Written for a developer **not fluent in state machine terminology**.

### State machine (FSM)

- **What it is:** a model with a finite set of **states**, one **current state**, and guarded
  **transitions** between them, triggered by **triggers**.
- **Example:** an order is `Draft` → `Submitted` → `Paid` → `Shipped`. Only one of these is "current"
  at any time.

### State

- **What it is:** a named condition/phase the machine can be in.
- **Example:** `OrderState.Paid`.

### Trigger

- **What it is:** an input that may cause a transition (other libraries call it an _event_).
- **Example:** `OrderTrigger.Pay`.

### Transition

- **What it is:** moving from a source state to a destination state when a trigger is fired.
- **Example:** firing `Pay` while in `Submitted` moves to `Paid`.

### Guard

- **What it is:** a predicate (a `bool`-returning check) that must be `true` for a transition to be
  allowed.
- **Example:** "only allow `Pay` if `payment.Succeeded == true`."

### Entry / Exit / Transition actions

- **What it is:** side-effect code run when entering a state (`OnEntry`), leaving a state (`OnExit`),
  or when a transition happens (`OnTransitioned`). They don't decide anything — they _do_ things
  (log, save, call an API).
- **Example:** `OnEntry(PaymentState.Authorized, _ => ChargeCard())`.

### Internal transition

- **What it is:** a trigger handled without leaving/re-entering the state and without changing state —
  just a side effect.
- **Example:** a heartbeat ping that updates `_lastSeen` while staying `Open`.

### Reentry

- **What it is:** a transition that exits and re-enters the _same_ state, re-running exit/entry
  actions.
- **Example:** `PermitReentry(Retry)` to reset a backoff counter while staying in `Retrying`.

### Parameterized trigger / payload

- **What it is:** a trigger that carries extra data (a payload) through the transition, available to
  guards, actions, and the transition stream.
- **Example:** `Pay` carries a `PaymentReceipt`.

### Observable (`IObservable<T>`)

- **What it is:** a push-based stream. A producer pushes values (`OnNext`), errors (`OnError`), and
  completion (`OnCompleted`) to subscribers, who react with LINQ-style operators.
- **Example:** `machine.Subscribe(state => progressBar.Value = ...)`.

### Observer (`IObserver<T>`)

- **What it is:** the consumer side of an observable — an object with `OnNext/OnError/OnCompleted`.
  A machine can _be_ an observer so you can `.Subscribe(machine)` any stream into it.
- **Example:** `messageBus.Observe<OrderEvent>().Subscribe(machine)`.

### Hot vs cold

- **What it is:** a **hot** observable pushes values regardless of subscribers (and usually replays
  or loses them); a **cold** one produces a fresh sequence per subscriber.
- **Example:** a `BehaviorSubject` is hot and _replays the latest value_ to late subscribers — that's
  how the machine gives you the current state immediately.

### Scheduler

- **What it is:** Rx's abstraction for _where/when_ work runs (thread, timer). Injectable so tests can
  use virtual time.
- **Example:** `Scheduler.Default` (thread pool) in production, `TestScheduler` in tests.

### Firing mode

- **What it is:** how concurrent triggers are handled — `Immediate` (synchronous, single-threaded) vs
  `Queued` (serialized queue).
- **Example:** see D-04.

### Hierarchy (superstate / substate)

- **What it is:** nesting states inside a "superstate"; being in a substate means you're also "in" the
  superstate. **Not implemented until S-11.**
- **Example:** `Idle` superstate containing `Connected` and `Disconnected`.

### History

- **What it is:** remembering which substate was active when leaving a superstate, so re-entering can
  restore it (`None` / `Shallow` / `Deep`). **S-11.**

### Snapshot persistence

- **What it is:** saving the machine's current state (and, in queued mode, pending triggers) so it can
  be restored after a restart. **S-12.**

### Fail-fast validation

- **What it is:** catching configuration mistakes (e.g., a transition to an unconfigured state) at
  startup instead of at runtime.

---

## 4. Tasks

### T1 — Decision register: resolve D-01..D-10

- **Goal:** turn every open PRD question into a working assumption that later sprints can start from.
- **Work to perform:** For each decision below, record: working assumption (suggestion), rationale,
  alternatives considered, and a "validate via" pointer. Update the table in README §3.1 statuses as
  assumptions are agreed.
- **Files / locations:** this document (§4.1), plus `Docs/Sprint Planning/README.md` §3.1.
- **Acceptance criteria:** every decision has all four fields filled; none left blank.
- **Tests / Samples:** n/a (pointers are recorded, tests come in later sprints).

### T2 — Propose the v1 API surface (suggestions)

- **Goal:** capture a suggested public API for review, clearly marked provisional.
- **Work to perform:** Write the proposed surface in §4.2 based on PRD §10.2/§10.3, folding in the
  T1 assumptions. Annotate each type with where it will be validated.
- **Files / locations:** this document (§4.2).
- **Acceptance criteria:** the surface covers FR-1..FR-31 surface-relevant items; every public type
  marked provisional; no contradictions with PRD §10.
- **Tests / Samples:** validated from S-02 onward.

### T3 — Propose environment defaults

- **Goal:** settle the toolchain so S-01 can build immediately.
- **Work to perform:** Record proposed defaults in §4.3 (SDK, layout, CI, package target, test
  tooling, docs tooling). Owner confirms or adjusts.
- **Files / locations:** this document (§4.3); reflected in README §8.
- **Acceptance criteria:** defaults are specific enough for S-01 to scaffold without further
  questions.
- **Tests / Samples:** n/a.

### T4 — API readability check (spike, no code)

- **Goal:** validate the suggested API _on paper_ before building it — the first "iterate via
  examples" pass.
- **Work to perform:** Hand-compile PRD §8.1 examples **UC-2 (payment)** and **UC-5 (connection FSM)**
  against the proposed API (§4.2). Note every awkwardness, missing member, or typo. Record the result
  in §4.4.
- **Files / locations:** this document (§4.4).
- **Acceptance criteria:** a written list of findings (or "reads cleanly") for both scenarios.
- **Tests / Samples:** the two PRD examples transcribed against the proposed API.

### T5 — Decision register sync

- **Goal:** keep the roadmap authoritative.
- **Work to perform:** Copy agreed assumptions into the README change-control register / status
  tables; mark statuses (Open → Provisional → Agreed).
- **Files / locations:** `Docs/Sprint Planning/README.md`.
- **Acceptance criteria:** README §3.1 statuses match this document; register clean.
- **Tests / Samples:** n/a.

### T6 — Review & accept (owner)

- **Goal:** get sign-off before S-01.
- **Work to perform:** Owner reviews T1–T5 outputs; confirms or amends defaults/assumptions.
- **Files / locations:** n/a (meeting/review).
- **Acceptance criteria:** §2.1 DoD all checked; S-00 status → Ready.
- **Tests / Samples:** n/a.

---

## 4.1 Decision register (working assumptions)

| #    | Topic                                   | Working assumption (suggestion)                                                                                                                                                                                                                                                                                                             | Rationale                                                                         | Alternatives                                                                      | Validate via                                   |
| ---- | --------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------- | --------------------------------------------------------------------------------- | ---------------------------------------------- |
| D-01 | Public API contract                     | Capture the proposed surface in §4.2; it is **provisional** and evolves through samples/tests                                                                                                                                                                                                                                               | Avoids freezing a bad shape early                                                 | Freeze now (rejected — philosophy)                                                | Every sprint's samples                         |
| D-02 | Async sequencing                        | `FireAsync` order: evaluate guards (awaiting async ones) → run exit actions → transition actions → entry actions. `Transitions` emits when the transition is **accepted** (guards passed), **before** entry actions; `TransitionsCompleted` emits **after** the last entry action completes. No entry actions ⇒ completed fires right after | Matches "completed" semantics (FR-4/§5.3); gives consumers a "committed" signal   | Emit `Transitions` after all actions (rejected — loses the "about to run" signal) | S-07 tests (UC-2 payment async)                |
| D-03 | Mid-transition exception / rollback     | If a guard or action throws, **no state change occurs** (machine stays in the source state); the exception routes per policy — `Throw` (default) propagates, `Observe` pushes a `StateMachineError` to `Errors`, `Ignore` swallows. State invariant always preserved                                                                        | Prevents undefined/partial states (PRD §10.4 single source of truth)              | Commit destination then roll back (rejected — complex, non-atomic)                | S-05/S-07 error tests                          |
| D-04 | FiringMode                              | `Immediate` = default: synchronous, single-threaded, reentrancy-protected (nested `Fire` from inside an action: rule decided in S-03). `Queued` = serialized, thread-safe → S-09. Worker-thread "active" mode deferred to S-14 (O6)                                                                                                         | Matches PRD §9 complexity split (queued = medium, active = high)                  | Ship Queued first (rejected — bigger risk before core is stable)                  | S-03 (immediate), S-09 (queued), S-14 (stress) |
| D-05 | Mixed sync/async guard ordering         | Guards evaluate strictly in **registration order**; async guards awaited in that order; transition permitted iff **all** pass; each evaluation emits on `GuardResults` in order                                                                                                                                                             | Deterministic (NFR-6)                                                             | Parallel guard evaluation (rejected — non-deterministic side effects)             | S-05, S-07                                     |
| D-06 | `otherwise` / default transition (FR-5) | `PermitIf(trigger, destination, guard, description)` plus an `Otherwise(destination)` fallback (or an overload with `otherwiseDestination`) that applies when all other guards for that trigger fail                                                                                                                                        | Matches Stateless's widely-used pattern; gives the "why blocked / fallback" story | No fallback (rejected — FR-5 requires it)                                         | S-05 (UC-7 wizard "Next" gating)               |
| D-07 | Dynamic destination (FR-2)              | `PermitDynamic(trigger, selector)` where `selector` computes the destination at fire time (after guards); parameterized variant uses the payload                                                                                                                                                                                            | Covers routing-by-value cases (UC-10 seek)                                        | Config-time destinations only (rejected — FR-2 names dynamic selectors)           | S-05/S-06                                      |
| D-08 | `PermittedTriggers` stream              | Re-emits when the **state changes** and when a **guard-dependent re-evaluation** happens (guards can change without a state change). `GuardResults` gives per-guard detail                                                                                                                                                                  | Avoids stale "permitted" UI (UC-7)                                                | Only on state change (rejected — async guards can change permittedness in-place)  | S-05 (permitted-triggers UI, UC-7)             |
| D-09 | Dispose / lifetime                      | `StateMachine<TState,TTrigger> : IDisposable`. Disposing completes all exposed streams (`OnCompleted`) and disposes internal subjects. `Fire`/`FireAsync` after dispose throw `ObjectDisposedException`. `Subscribe` after dispose → completes immediately (validate)                                                                       | Clean resource story for long-lived hosts                                         | No dispose (rejected — leaks subjects/timers)                                     | S-03/S-08 stream-lifetime tests                |
| D-10 | Activation / deactivation (FR-3)        | API surfaces `OnActivated`/`OnDeactivated` now, but behavior ships in S-11 (hierarchy). Until then they are **no-ops** (not config-validation errors)                                                                                                                                                                                       | They only make sense with superstates; keeps M1 honest                            | Reject at validation in M1 (rejected — churn)                                     | S-11                                           |

---

## 4.2 Proposed v1 API surface (PROVISIONAL — evolves via tests & samples)

> Based on PRD §10.2/§10.3. **Every item is a suggestion**, to be validated/refined by the sprint
> noted. Do not treat as frozen.

**Namespaces:** `RxStateMachine` root; sub-namespaces `RxStateMachine.Reactive` (helpers),
`RxStateMachine.Persistence` (S-12), `RxStateMachine.Diagnostics` (S-13).

```csharp
// Core machine — producer + consumer (S-03, S-08)
public sealed class StateMachine<TState, TTrigger> : IObservable<TState>,
        IObserver<TriggerWithParameters<TState, TTrigger>>, IDisposable          // D-09
    where TState : notnull where TTrigger : notnull
{
    public StateMachine(TState initialState, StateMachineOptions? options = null);
    public StateMachine(Func<TState> accessor, Action<TState> mutator,            // FR-21 (S-10)
        StateMachineOptions? options = null);

    public StateMachineConfiguration<TState, TTrigger> Configure(TState state);   // S-02

    public TState State { get; }
    public IObservable<TState> States { get; }                                    // == this
    public IObservable<StateChange<TState>> StateChanges { get; }                 // S-03
    public IObservable<Transition<TState, TTrigger>> Transitions { get; }         // S-03
    public IObservable<Transition<TState, TTrigger>> TransitionsCompleted { get; }// S-04
    public IObservable<GuardResult<TState, TTrigger>> GuardResults { get; }       // S-05
    public IObservable<PermittedTriggers<TState, TTrigger>> PermittedTriggers { get; } // S-05
    public IObservable<StateMachineError> Errors { get; }                         // S-05

    public Transition<TState, TTrigger> Fire(TTrigger trigger);                   // S-03 (FR-30)
    public Transition<TState, TTrigger> Fire<TArg>(TriggerWithParameters<TTrigger, TArg> trigger, TArg arg); // S-06
    public Task<Transition<TState, TTrigger>> FireAsync(TTrigger trigger, CancellationToken ct = default);   // S-07
    public Task<Transition<TState, TTrigger>> FireAsync<TArg>(
        TriggerWithParameters<TTrigger, TArg> trigger, TArg arg, CancellationToken ct = default);             // S-07

    public IEnumerable<TTrigger> GetPermittedTriggers();                          // S-05 (FR-16)
    public Task<IEnumerable<TTrigger>> GetPermittedTriggersAsync(CancellationToken ct = default);             // S-07
    public StateMachineInfo GetInfo();                                            // S-02/S-10 (FR-17)
    public bool IsInState(TState state);                                          // S-11 (FR-26)

    void IObserver<TriggerWithParameters<TState, TTrigger>>.OnNext(...);          // S-08 (FR-14)
}
```

**Records (S-03/S-05/S-06):**

- `Transition<TState,TTrigger>` — `Source`, `Destination`, `Trigger`, `Payload`, `Kind`
  (`Normal`/`Internal`/`Reentry`), `Timestamp`, `Duration`.
- `StateChange<TState>` — `Previous`, `Current`, `Timestamp`, `Kind`.
- `GuardResult<TState,TTrigger>` — `State`, `Trigger`, `GuardDescription`, `Passed`, `Exception?`, `Timestamp`.
- `PermittedTriggers<TState,TTrigger>` — `State`, `Triggers`, `Timestamp`.
- `StateMachineError` — `State`, `Trigger`, `Exception`, `Policy`, `ErrorKind`
  (`UnhandledTrigger`/`GuardException`/`ActionException`/`ConfigurationError`).
- `TriggerWithParameters<TState,TTrigger>` — `Trigger`, `Parameters`.

**Options (S-03/S-09):** `StateMachineOptions` — `Scheduler` (default `Scheduler.Default`),
`FiringMode` (`Immediate`/`Queued`), `UnhandledTriggerPolicy` (`Throw`/`Ignore`/`Observe`),
`ExceptionPolicy` (same set). **Default policy: `Throw`** (PRD §14.4).

**Fluent configuration (S-02…S-06, S-11):**
`Permit`, `PermitIf(…, description)`, `PermitIfAsync`, `PermitReentry`, `InternalTransition`,
`PermitDynamic` (D-07), `Otherwise` (D-06), `OnEntry`/`OnEntryAsync`, `OnExit`/`OnExitAsync`,
`OnEntryFrom(trigger, …)`, `OnActivated`/`OnDeactivated` (D-10), hierarchy members in S-11
(`SubstateOf`, `InitialTransitionTarget`, history).

---

## 4.3 Environment defaults (proposed)

| Item                | Proposed default                                                                                                                       |
| ------------------- | -------------------------------------------------------------------------------------------------------------------------------------- |
| .NET SDK            | Latest stable **.NET 10 SDK** (10.0.x) — required for `net10.0`; builds `netstandard2.0` too                                           |
| Library targets     | `netstandard2.0;net10.0` (NFR-8)                                                                                                       |
| Test project target | `net10.0` (library stays multi-targeted)                                                                                               |
| Test tooling        | xUnit (latest stable), FsCheck + FsCheck.Xunit, BenchmarkDotNet (latest), System.Reactive **6.x** pinned                               |
| CI                  | GitHub Actions; matrix `ubuntu-latest` + `windows-latest`; build + test + pack                                                         |
| Package             | `PackageId = RxStateMachine`, version `0.1.0` during dev, SemVer, XML docs embedded, deterministic build; publish to NuGet.org (later) |
| Docs                | Start minimal (`docs/` + placeholder index); DocFX considered in S-14                                                                  |
| Repo layout         | See README §8                                                                                                                          |

---

## 4.4 API readability check (UC-2 & UC-5) — findings

> Result of T4. Filled in when the check is performed. Expected focus: does the fluent API read
> naturally? Are there missing members (e.g., guard-description overloads, timer helpers)? Are the
> stream names intuitive?

- **UC-2 (payment):** <findings>
- **UC-5 (connection FSM):** <findings>

---

## 5. Example scenarios

### Scenario: Payment lifecycle (UC-2) — used to validate D-02/D-03 and the async API

- **Context:** a payment goes `Initiated` → `Authorized` → `Captured` (or `Failed`/`Refunded`), with
  an async authorization call on entry and a 30s timeout that cancels.
- **Configuration (against proposed API):**

```csharp
var machine = new StateMachine<PaymentState, PaymentTrigger>(PaymentState.Initiated);

machine.Configure(PaymentState.Initiated)
    .OnEntry(_ => _ = AuthorizeAsync())
    .Permit(PaymentTrigger.Authorized, PaymentState.Authorized)
    .Permit(PaymentTrigger.Failed, PaymentState.Failed);

machine.Configure(PaymentState.Authorized)
    .Permit(PaymentTrigger.Capture, PaymentState.Captured)
    .Permit(PaymentTrigger.Refund, PaymentState.Refunded);

Observable.Timer(TimeSpan.FromSeconds(30))
    .Select(_ => PaymentTrigger.Failed)
    .Subscribe(machine);

machine.Subscribe(state => _repo.Save(transactionId, state));
```

- **Walkthrough:** firing `Authorized` from `Initiated` runs no guards, exits `Initiated`, emits
  `Transitions`, runs `Authorized`'s entry action, then emits `TransitionsCompleted` (D-02). A thrown
  entry action leaves state at `Initiated` (D-03).
- **State table:**

| #   | Trigger                        | Guard | Actions                          | New state   | Streams emitted                    |
| --- | ------------------------------ | ----- | -------------------------------- | ----------- | ---------------------------------- |
| 1   | Authorized                     | —     | exit Initiated; entry Authorized | Authorized  | Transitions → TransitionsCompleted |
| 2   | (timer) Failed                 | —     | exit Authorized                  | Failed      | Transitions → TransitionsCompleted |
| 3   | Capture (invalid in Initiated) | —     | none                             | (unchanged) | Errors (UnhandledTrigger)          |

### Scenario: Connection FSM (UC-5) — used to validate reentry + internal transitions

- **Context:** `Connecting` → `Open` ↔ `Retrying`, with heartbeat as an internal transition and
  `Retry` as a reentry.
- **Configuration (against proposed API):**

```csharp
machine.Configure(ConnectionState.Connecting)
    .Permit(ConnectionTrigger.Connected, ConnectionState.Open)
    .Permit(ConnectionTrigger.Failed, ConnectionState.Retrying);

machine.Configure(ConnectionState.Open)
    .InternalTransition(ConnectionTrigger.Heartbeat, _ => _lastSeen = DateTimeOffset.UtcNow)
    .Permit(ConnectionTrigger.Lost, ConnectionState.Retrying);

machine.Configure(ConnectionState.Retrying)
    .PermitReentry(ConnectionTrigger.Retry)
    .Permit(ConnectionTrigger.Connected, ConnectionState.Open);
```

- **Walkthrough:** `Heartbeat` in `Open` runs the side effect, **no** exit/entry, **no** state change,
  no `Transitions` (only internal). `Retry` in `Retrying` exits + re-enters `Retrying`, re-running its
  entry action (backoff reset), and emits `Transitions` with `Kind=Reentry`.
- **State table:**

| #   | Trigger                   | Guard | Actions                     | New state        | Streams emitted                    |
| --- | ------------------------- | ----- | --------------------------- | ---------------- | ---------------------------------- |
| 1   | Connected (in Connecting) | —     | exit Connecting; entry Open | Open             | Transitions → TransitionsCompleted |
| 2   | Heartbeat (in Open)       | —     | update \_lastSeen           | Open (unchanged) | (internal; no Transitions)         |
| 3   | Retry (in Retrying)       | —     | exit+entry Retrying         | Retrying         | Transitions (Kind=Reentry)         |

---

## 6. Tests & samples checklist

- [ ] Decision register D-01..D-10 complete (T1)
- [ ] Proposed API surface documented + marked provisional (T2)
- [ ] Environment defaults proposed (T3)
- [ ] UC-2 and UC-5 readability check recorded (T4)
- [ ] README register/status in sync (T5)
- [ ] Owner review/acceptance (T6)

_(No code is written in this sprint.)_

---

## 7. Risks & tricky bits

- **Over-designing before evidence** → every suggestion is provisional with a validate-via pointer;
  change is cheap early.
- **Contradicting the PRD** → PRD wins; follow change control (README §7).
- **Decision paralysis** → pick the suggested default, validate later; don't block S-01.
- **netstandard2.0 surprises (C# features need shims)** → verified in S-01 (T3) before engine work.

---

## 8. Progress log / resume

| Task | Status      | Notes                                                     |
| ---- | ----------- | --------------------------------------------------------- |
| T1   | Not started | Decisions D-01..D-10 drafted above; pending review        |
| T2   | Not started | Proposed API surface drafted above (§4.2); pending review |
| T3   | Not started | Env defaults drafted above (§4.3); pending review         |
| T4   | Not started | Findings placeholders in §4.4                             |
| T5   | Not started | README register currently has the philosophy entry only   |
| T6   | Not started | Awaiting owner                                            |

**If work is interrupted, resume here:**

- Nothing is in flight; the doc contains drafts ready for review.

**Next actions on resume:**

1. Owner reviews §4.1 (decisions), §4.2 (API), §4.3 (env defaults).
2. Perform T4 (readability check) and record findings in §4.4.
3. On acceptance: update README §3.1 statuses + change-control register, set S-00 status → Ready.

**Checkpoint notes:**

- Adopted "no hard contracts / iterate via tests & samples" philosophy (README §2).
- All API shapes in §4.2 are provisional.

---

## 9. Change control

> **Change control.** This document is derived from `Docs/RxStateMachine PRD.md` (the source of
> truth). If during development you are instructed — or conclude — that a change **deviates from the
> PRD** (new behavior, changed signature, removed feature, altered semantics), you MUST:
>
> 1. **Stop** and record the proposal in `Docs/Sprint Planning/README.md` → Change Control Register
>    (what, why, which `FR`/`NFR`/`UC` affected).
> 2. **Cross-reference the PRD**: confirm whether it should be updated and draft the edit (section,
>    old/new text).
> 3. **Check this sprint and all future sprint documents for conflict**; update affected tasks,
>    prerequisites, and the dependency map.
> 4. **Do not implement** the change until the PRD update is confirmed (or explicitly waived and
>    recorded).

**Nuance:** because v1 has no hard contracts, an API change that merely refines a _suggestion_ and is
validated by tests/samples — without contradicting the PRD — is a normal part of development. Record
it in this sprint's Progress Log (Checkpoint notes), not the Change Control Register.
