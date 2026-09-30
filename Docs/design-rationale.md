# Design Rationale

Why the machine is built the way it is, and which existing ideas it borrows. Rules that follow from
this reasoning are written up in [engineering-standards.md](engineering-standards.md).

## Why the hybrid architecture

The transition engine is a **classic imperative FSM core** (guards, entry/exit/transition actions,
async support, validation, introspection), and **everything observable flows through native
`IObservable<T>` streams**. The engine is deliberately not a pure-Rx reduction of an input stream.
This is the most reusable structure: it does not force consumers into pure-Rx patterns they may not
want, while fully enabling them when they do.

### Benefits

| Benefit                       | What it means                                                                                                                                                                  |
| ----------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| Natural async side effects    | Complex async work (I/O, retries, database calls) is written with plain `async`/`await` in entry/exit/transition actions — not `SelectMany` spaghetti or race-prone pipelines. |
| Readable debugging            | Standard C# stack traces in the core, not deep Rx pipelines and scheduler forensics.                                                                                           |
| First-class guards & errors   | Guard logic and error handling are explicit, with clear policies and rich error objects.                                                                                       |
| Explicit state ownership      | The current state is owned by the machine in one place, not derived from event-stream history; no parallel copies to drift.                                                    |
| Seamless reactive consumption | UI / telemetry / persistence binding is plain LINQ over the exposed streams.                                                                                                   |
| Consumer-side time operators  | Debounce, throttle, and timer operators compose on the consumer side of the streams, with helper APIs for timers.                                                              |

### Caveats

- The machine is not a pure function of an input stream; consumers who want a fully derived model
  must build it themselves from the exposed streams.
- The sweet spot is domain services, workflows, payments, and most line-of-business applications.
  High-frequency input streams (games, IoT telemetry) are fully supported through immediate firing
  and observable inputs, but the library is not built as a pure stream-reduction engine.

### What "stream-reduction" means here

In functional programming, _reduce_ / _fold_ collapses a sequence into a single accumulated value by
repeatedly applying a combining function — Rx's `Scan` is the incremental version, emitting the
running accumulator after each input:

```
seed = Draft
step 1: fold(Draft,     Submit)  → Submitted
step 2: fold(Submitted, Pay)     → Paid
step 3: fold(Paid,      Ship)    → Shipped
```

A "stream-reduction engine" is therefore a state machine whose state is **never stored** — it is
derived on the fly as a pure fold of the incoming trigger stream:

```csharp
// A "stream-reduction" state machine — state is a running fold of the trigger stream:
currentState = triggers.Scan(OrderState.Draft,
    (state, trigger) => (state, trigger) switch
    {
        (OrderState.Draft, OrderTrigger.Submit) => OrderState.Submitted,
        (OrderState.Submitted, OrderTrigger.Pay) => OrderState.Paid,
        (OrderState.Paid, OrderTrigger.Ship) => OrderState.Shipped,
        _ => state
    });
```

RxStateMachine is not built this way (see the design note below). The current state is **owned
explicitly** by the transition engine, which runs guards, entry/exit actions, async work, and
validation; the observable streams are _outputs_ of that engine — not a fold recomputed from trigger
history on every emission.

### Design note — why the core is not a pure `Scan`

A fully reactive engine could derive state with `Scan`:

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
inside guards/actions, and it has no place for entry/exit side effects, guard descriptions, or "why
is this blocked?" introspection. The hybrid keeps all of that while still exposing the same
`IObservable<TState>` to consumers.

## Design principles

- **Observable-pure core:** the engine pushes to internal subjects; the public API exposes them
  `AsObservable()`-style (read-only, no external `OnNext`).
- **Validation at configuration time:** a validator runs after the configuration is built to catch
  most errors before runtime.
- **Single source of truth:** the current state lives in one place; the observable replays it; there
  are no parallel copies to drift.
- **No blocking:** all async flows return `Task`; schedulers are injected, never ambient.

## Design influences

The design draws on the familiar vocabulary and feature set of the .NET state machine ecosystem (see
[references.md](references.md)), so that users of the established libraries feel at home, while
taking a different approach with an observable-first API that simplifies reactive usage and supports
complex async scenarios.

### Feature ideas adopted from the ecosystem

1. Firing modes (immediate vs queued); transition and transition-completed notifications; external
   state storage via a getter/setter pair; a machine-description object for introspection; graph
   export in **Mermaid + D2**; guard descriptions for rich error messages; parameterized triggers with
   a typed argument.
2. Hierarchical states with history types (none / shallow / deep); a thread-safe, worker-thread
   ("active") style of machine versus a passive one; extension callbacks around the full lifecycle
   (entering / entered state, firing / fired event, guard and action exceptions); persistence of
   current state plus queued events plus history; reporting.
3. Saga-style usage; scheduling and timeouts; state-machine-as-persistence-subject; correlation.
   (Saga _orchestration_ is a future consideration — see [saga-exploration.md](saga-exploration.md).)
4. Persistence-first design for long-running workflows; snapshot persistence is supported without full
   long-running workflow orchestration.

Not every idea is adopted as-is. For example, the separate worker-thread "active" style is replaced by
a single opt-in queued mode; see [design-decisions.md](design-decisions.md) (DD-04).
