# RxStateMachine — Sprint Plan & Roadmap

> **Purpose.** This folder breaks the build-out of the `RxStateMachine` library into small,
> **outcome-based sprints** (no time estimates). Each sprint is a self-contained document a
> developer can pick up, execute, and resume if interrupted. This file is the **index**, the
> **dependency map**, and the **Change Control Register** for the whole effort.

---

## 1. Source of truth

- **PRD:** [`../RxStateMachine PRD.md`](../RxStateMachine%20PRD.md) — the authoritative product
  specification. Every sprint document references PRD items by ID (`FR-x`, `NFR-x`, `UC-x`, `§x`).
  If a sprint doc and the PRD ever disagree, **the PRD wins** until the PRD is updated through the
  change-control process in §7.
- **Repo:** `c:\Data\Dev\GitHub\mjfreelancing\RxStateMachine` — currently only `LICENSE` + this
  `Docs` folder (greenfield; no solution or code yet).

---

## 2. Design philosophy (read this first)

> **No hard contracts are established up front.**

The PRD sketches the architecture and API, but its §10 code is explicitly "illustrative." We
deliberately do **not** pin a frozen API contract before writing code. Instead:

1. Each sprint starts from **suggested** API shapes and semantics (see `S-00`).
2. We **iterate through tests and sample applications**: writing a test or sample against a
   proposed API is the primary way we validate the design and uncover gaps early.
3. The API surface **evolves** as we build. A change that refines a _suggestion_ and is validated
   by tests/samples — **without contradicting the PRD** — is a normal part of development. Record it
   in the relevant sprint's Progress Log (see §7 for when the full change-control process applies).
4. The PRD remains the source of truth for **what** we build; sprint docs and tests clarify
   **how** it behaves.

---

## 3. PRD completeness assessment (basis for S-00)

The PRD is **largely complete** (`FR-1..31`, `NFR-1..11`, `UC-1..10`, milestones `M0–M4`, §10
architecture, §11 testing strategy). It is not "task-ready" until a set of open questions is
resolved. We treat these as **working assumptions to validate** — not frozen contracts.

### 3.1 Blocking decisions (owned by S-00)

| #    | Topic                                                                                        | Status                         |
| ---- | -------------------------------------------------------------------------------------------- | ------------------------------ |
| D-01 | Pinned v1 public API contract (PRD §10 is "illustrative")                                    | Provisional suggestion in S-00 |
| D-02 | Async sequencing — `FireAsync` await order; `Transitions` vs `TransitionsCompleted` ordering | Open                           |
| D-03 | Mid-transition exception / rollback semantics                                                | Open                           |
| D-04 | `FiringMode.Immediate` vs `Queued` vs worker-thread "active" mode                            | Open                           |
| D-05 | Mixed sync/async guard ordering                                                              | Open                           |
| D-06 | `otherwise` / default transition (FR-5)                                                      | Open                           |
| D-07 | Dynamic destination selectors (FR-2)                                                         | Open                           |
| D-08 | `PermittedTriggers` stream re-emission semantics                                             | Open                           |
| D-09 | Dispose / lifetime semantics of machine + streams                                            | Open                           |
| D-10 | Activation / deactivation actions (FR-3): M1 or M3                                           | Open                           |

### 3.2 Deferrable decisions (designed inside their owning sprint)

- Hierarchy ordering rules (exit/entry ordering, history restore order, nested `IsInState`) → S-11.
- `IStateMachinePersistence` shape / snapshot format & versioning → S-12.
- SemVer / versioning policy detail (NFR-9) → S-01 / S-14.
- Environment specifics (SDK version, CI, package target, tooling) → proposed in S-00, built in S-01.

---

## 4. Approach to move forward

1. **S-00** proposes the working assumptions, a suggested (provisional) API surface, and
   environment defaults — each with a **"validate via"** pointer to the tests/samples that will
   exercise it.
2. Sprint documents are generated **just-in-time**: a sprint is fully detailed when we start it, so
   a decision made in one sprint is reflected in the next without rework.
3. Every sprint is **iterative**: it builds only on completed prerequisite sprints (see the
   dependency map in §6).
4. At the end of each milestone (M0–M4) we review; if a PRD change is needed we follow the
   change-control process in §7.

---

## 5. Sprint document format

Every sprint follows [`templates/sprint-template.md`](templates/sprint-template.md) and contains:

- **Metadata** — ID, title, status, PRD refs, prerequisites, depends-on.
- **Outcome + Definition of Done** — what "done" looks like, objectively.
- **Concepts & Vocabulary** — plain-language explanations of every state-machine / Rx term used in
  the sprint, written for a developer who is **not fluent in state machine terminology**.
- **Tasks** — each with goal, work to perform, files/locations, acceptance criteria, tests, samples.
- **Example scenarios** — fleshed-out, step-by-step worked examples (state tables, mermaid
  diagrams) tied to the PRD use cases.
- **Tests & samples checklist** — tickable.
- **Risks & tricky bits** — sprint-specific + PRD §12 pointers.
- **Progress log / resume** — per-task status table + "if interrupted, resume here" instructions.
- **Change control note** — standard wording (see §7).

---

## 6. Sprint breakdown & dependency map

Finer-grained than the PRD's indicative "Sprint 1–12" (treated as guidance, not binding). Each
sprint depends on the one(s) in **Depends on**.

| ID   | Sprint                                     | Outcome (Definition of Done in a phrase)                                                                                                           | Key PRD refs                             | Depends on |
| ---- | ------------------------------------------ | -------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------- | ---------- |
| S-00 | **Decisions & API Suggestions**            | Working assumptions for D-01..D-10, suggested v1 API surface, env defaults — all provisional, each with a validate-via pointer                     | FR-29, NFR-2/8/9, §5, §10, §14           | PRD        |
| S-01 | **Scaffolding & CI**                       | Green multi-targeted solution (netstandard2.0 + net10.0 + net11.0), test projects, CI, docs stub, benchmark harness stub, sample stub                        | M0, FR-29, NFR-7/8                       | S-00       |
| S-02 | **State model, config & validation**       | Machine that configures, validates fail-fast, owns state, exposes `GetInfo` skeleton                                                               | FR-1, FR-2 (part), FR-10, FR-17 (part)   | S-01       |
| S-03 | **Basic transitions & observability**      | `Permit` + `Fire` returning `Transition`; machine is `IObservable<TState>` (replays); `Transitions`/`StateChanges` streams; `OnTransitioned`       | FR-4, FR-11, FR-12/13, FR-20, FR-30      | S-02       |
| S-04 | **Actions + internal & reentrant**         | `OnEntry`/`OnExit`/`OnEntryFrom`, `InternalTransition`, `PermitReentry`, `TransitionsCompleted`                                                    | FR-3, FR-7, FR-8, FR-12, FR-20           | S-03       |
| S-05 | **Guards, guard streams & default errors** | `PermitIf` (ordered), guard descriptions, `GuardResults` stream, `GetPermittedTriggers`, `Errors` stream + **Throw** policy, `StateMachineError`   | FR-5, FR-16, FR-18, FR-19 (throw), FR-31 | S-04       |
| S-06 | **Parameterized triggers & payloads**      | `TriggerWithParameters`, `Fire<TArg>`, payload flows through actions/guards/streams                                                                | FR-6, FR-30                              | S-05       |
| S-07 | **Async transitions**                      | `FireAsync`, async actions/guards, async permitted triggers, `CancellationToken`, async sequencing per D-02                                        | FR-9, FR-16, NFR-4/10/11                 | S-06       |
| S-08 | **Observable input, schedulers & timers**  | Machine is `IObserver<…>` (`.Subscribe(machine)`), scheduler integration, timer helpers                                                            | FR-14, FR-15, FR-24                      | S-07       |
| S-09 | **Full error policies & queued firing**    | `Ignore`/`Observe` policies, action-exception policies, `FiringMode.Queued` semantics                                                              | FR-19 (all), FR-31, NFR-3 (part)         | S-08       |
| S-10 | **External state storage & introspection** | `Func/Action` accessor/mutator, full `StateMachineInfo`                                                                                            | FR-17, FR-21                             | S-09       |
| S-11 | **Hierarchical states & history**          | `SubstateOf`, `InitialTransitionTarget`, history `None/Shallow/Deep`, `IsInState(superstate)`, activation/deactivation actions, hierarchy ordering | FR-25, FR-26, FR-3 (activation)          | S-10       |
| S-12 | **Snapshot persistence**                   | `IStateMachinePersistence`; serialize state + active substate history + queued triggers; persistence-as-observer                                   | FR-22, FR-23                             | S-11       |
| S-13 | **Visualization & reporting**              | `MermaidFormatter`, `D2Formatter`, `TextReport`                                                                                                    | FR-27, FR-28                             | S-12       |
| S-14 | **Hardening**                              | Thread-safe active mode, concurrency stress tests (O6), perf benchmarks (O4/NFR-5), full docs + all samples (NFR-7/O5), semver 1.0                 | NFR-3/5/6/7/9, O4/5/6                    | S-13       |

Milestone mapping: **S-00/S-01 → M0 · S-02…S-06 → M1 · S-07…S-10 → M2 · S-11…S-13 → M3 · S-14 → M4**

### 6.1 Status of sprints

| Sprint    | Document                                                                         | Status                  |
| --------- | -------------------------------------------------------------------------------- | ----------------------- |
| S-00      | [`S-00-Decisions-and-API-Suggestions.md`](S-00-Decisions-and-API-Suggestions.md) | Draft (awaiting review) |
| S-01      | [`S-01-Scaffolding-and-CI.md`](S-01-Scaffolding-and-CI.md)                       | Draft (awaiting review) |
| S-02…S-14 | _(generated just-in-time)_                                                       | Not started             |

---

## 7. Change control

> **Change control.** This planning folder and all sprint documents are derived from
> `Docs/RxStateMachine PRD.md` (the source of truth). If during development you are instructed — or
> conclude — that a change **deviates from the PRD** (new behavior, changed signature, removed
> feature, altered semantics), you MUST:
>
> 1. **Stop** and record the proposal in the Change Control Register below (what, why, which
>    `FR`/`NFR`/`UC` affected).
> 2. **Cross-reference the PRD**: confirm whether the PRD should be updated and draft the edit
>    (section, old/new text).
> 3. **Check this sprint and all future sprint documents for conflict**; update affected tasks,
>    prerequisites, and the dependency map.
> 4. **Do not implement** the change until the PRD update is confirmed (or explicitly waived and
>    recorded).

**Important nuance for this project:** because v1 deliberately has **no hard contracts** (§2), an
API change that merely _refines a suggestion_ — and is validated by tests/samples **without
contradicting the PRD** — does **not** require the full process above. Record it in the owning
sprint's Progress Log instead. The full process applies only when the change alters PRD-stated
behavior (an `FR`/`NFR`/`UC`/§ decision).

### Change Control Register

| Date       | Sprint | Change                                                                    | PRD impact                                | PRD update? | Conflicts checked?        | Status |
| ---------- | ------ | ------------------------------------------------------------------------- | ----------------------------------------- | ----------- | ------------------------- | ------ |
| 2026-08-24 | —      | Adopted "no hard contracts / iterate via tests & samples" philosophy (§2) | None — aligns with PRD draft status (§14) | No          | n/a (adopted before S-00) | Agreed |
| 2026-08-31 | —      | Added `net11.0` as an additional library target (`netstandard2.0;net10.0;net11.0`); tests/samples/SDK/CI use `net11.0` | NFR-8, §14.2, header | Yes         | README, S-00, S-01 updated            | Agreed |

---

## 8. Proposed repo layout (from S-00 T3)

```
RxStateMachine/
├── .github/workflows/ci.yml
├── Directory.Build.props
├── RxStateMachine.sln
├── src/RxStateMachine/                  # netstandard2.0 + net10.0 + net11.0
│   ├── RxStateMachine.csproj
│   ├── StateMachine{TState,TTrigger}.cs
│   ├── StateMachineConfiguration.cs
│   ├── Transition.cs · StateChange.cs · GuardResult.cs · PermittedTriggers.cs
│   ├── StateMachineError.cs · TriggerWithParameters.cs · StateMachineOptions.cs
│   ├── Reactive/                        # observable helpers, timers (S-08)
│   ├── Persistence/                     # (S-12)
│   ├── Diagnostics/                     # (S-13)
│   └── Validation/
├── tests/
│   ├── RxStateMachine.Tests/            # xUnit, net11.0
│   ├── RxStateMachine.Tests.Reactive/   # TestScheduler virtual time (S-07+)
│   └── RxStateMachine.Tests.Stress/     # concurrency stress (S-14)
├── samples/
│   ├── HelloRxStateMachine/             # smoke sample (S-01)
│   └── ...                              # per-UC samples added as built
├── benchmarks/RxStateMachine.Benchmarks/
├── docs/                                # docs site skeleton (S-01)
└── Docs/Sprint Planning/                # this folder
```
