# Future Consideration — Saga Orchestration (out of scope for the core)

Saga-style usage is a **supported pattern** (UC-8 in [use-cases.md](use-cases.md)) — machines can be
composed with snapshot persistence, timeouts, and correlation to act as saga participants. What is
**not** in scope is a _built-in saga orchestration engine_: automatic compensation/undo, a
distributed coordinator, or a long-running workflow engine (see the non-goals in
[vision.md](vision.md)).

## Open question

If a saga layer is ever added, the working assumption is that it ships as a **separate companion
library** built _on top of_ RxStateMachine — providing the orchestration, compensation, and
coordination glue — rather than growing the core machine. No decision has been made; this stays off
the roadmap.

## Exploration plan (hand-rolled examples)

To inform that decision, hand-roll a few saga examples directly against RxStateMachine before
committing to a design — for example, an order-fulfilment saga that spans multiple machines and, on
failure, compensates by firing `Cancel` / `Refund` / `Rollback` transitions. Goal: get a feel for how
orchestration looks in practice, which ergonomics are missing from the core, and whether the friction
justifies a dedicated library.
