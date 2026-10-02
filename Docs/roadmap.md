# Roadmap: Features in Delivery Order

There are no milestones, sprints, or deadlines. Features are built **in the order below**, each one
completed and verified before the next starts. The only milestone is the **final deliverable**: the
complete library, documentation, and samples, released as a stable package.

Each feature has a brief in [features/](features/) that is the input to one specification run.

| # | Feature | Brief | Builds on | Relative complexity |
| - | ------- | ----- | --------- | ------------------- |
| 01 | Project foundations — package, build, CI, quality gates | [01](features/01-foundations.md) | — | Low |
| 02 | Core transitions — configuration, guards, actions, payloads, validation | [02](features/02-core-transitions.md) | 01 | Low–Medium |
| 03 | Observable surface and trigger input | [03](features/03-observable-surface-and-input.md) | 02 | Medium |
| 04 | Introspection and diagnostics | [04](features/04-introspection-and-diagnostics.md) | 02, 03 | Low–Medium |
| 05 | Async, cancellation and error policies | [05](features/05-async-cancellation-and-error-policies.md) | 02, 03, 04 | Medium |
| 06 | Timers, timeouts and external state | [06](features/06-timers-and-external-state.md) | 03, 05 | Medium |
| 07 | Hierarchical states | [07](features/07-hierarchical-states.md) | 02–05 | High |
| 08 | Snapshot persistence | [08](features/08-snapshot-persistence.md) | 06, 07 | Medium–High |
| 09 | Diagram export and reports | [09](features/09-diagram-export-and-reports.md) | 04, 07 | Medium |
| 10 | Concurrency, performance and release readiness | [10](features/10-concurrency-and-hardening.md) | all | High |

## Where the complexity lives

- **Low (ship first):** basic permits, entry/exit/transition actions, internal and reentrant
  transitions, parameterized payloads, observable outputs, `Fire`, configuration validation.
- **Medium:** guards and guard streams, async guards/actions, introspection, external state storage,
  scheduler integration, error policies, timer helpers, export.
- **High (do last):** hierarchical states with history, thread-safe queued mode, snapshot persistence
  of the full machine, performance hardening.

## Why this order

1. **Core first** (01–04) so the library is useful early and gives feedback for the riskier designs.
2. **Async and observer input** (03, 05) build directly on the core and are where most real-world
   integrations live (webhooks, gateways, buses).
3. **Hierarchy and history** (07) is one of the most error-prone parts of any state machine engine; it
   benefits from a battle-tested core and a solid test suite first.
4. **Hardening** (10) focuses on thread-safety and performance only after the semantics are stable.

## Risks and mitigations

| Risk                                                    | Impact               | Mitigation                                                                                             |
| ------------------------------------------------------- | -------------------- | ------------------------------------------------------------------------------------------------------ |
| Observable API is over-engineered / confusing           | Adoption friction    | Hybrid keeps `Fire()` simple; streams are additive. Docs and samples first.                            |
| Hierarchy/history bugs are subtle and easy to get wrong | Correctness          | Built late, on a proven core; modelled on well-documented hierarchy semantics; heavy test coverage.    |
| netstandard2.0 costs time (shims, `#if`, an extra test leg) | Effort           | Whether to keep it is an open decision in feature [01](features/01-foundations.md). If kept: shims stay `internal` and limited, and the same suite runs on a runtime that resolves that asset. |
| Thread-safety surprises in queued mode                  | Production incidents | Immediate (single-threaded) mode ships first and is documented; queued mode is a clear opt-in.         |
| Scope creep (full workflow engine)                      | Effort               | Non-goals ([vision.md](vision.md)) kept visible; long-running workflow orchestration is out.           |
| Rx dependency perceived as heavy                        | Adoption             | System.Reactive is already ubiquitous; the library depends on it directly, with no wrapper.            |

## Definition of a successful final deliverable

- Every feature brief is complete and its requirements verified.
- Feature-coverage checklist for the most-used constructs is complete.
- Zero race conditions found by the stress suite in queued mode.
- At least 80% line coverage of the core engine; every sample builds and passes in CI.
- XML documentation is complete and every use-case sample builds and runs in CI.
