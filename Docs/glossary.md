# Glossary

- **State machine (FSM):** a model with a finite set of states, one current state, and guarded
  transitions between them triggered by triggers.
- **Trigger:** the input that may cause a transition (sometimes called an _event_ in other state
  machine libraries).
- **Guard:** a predicate that must return true for a transition to be permitted.
- **Observable (`IObservable<T>`):** a push-based, lazily-composed stream of values.
- **Hot vs cold:** a hot observable pushes regardless of subscribers and typically replays/loses
  values; a cold observable produces a fresh sequence per subscriber.
- **Hybrid:** a machine that is both a classic imperative FSM core and a native observable producer +
  observable-input consumer.
- **Reentry:** a transition that exits and re-enters the same state, re-running entry/exit actions.
- **Internal transition:** a side-effect transition that does not change state and runs no exit/entry
  actions.

## Terms used by the feature briefs

- **Payload:** typed data carried by a trigger and available to guards and actions.
- **Blocked vs fault:** *blocked* means no guard permitted the trigger (an expected outcome); a *fault*
  is an exception thrown by consumer code such as a guard or action.
- **Unhandled:** the current state has no registration at all for the trigger — distinct from
  *blocked*, where a registration exists but no guard permitted it (DD-11).
- **Otherwise:** an optional destination used when none of a trigger's guards pass (DD-06).
- **Dynamic destination:** a destination chosen by a function at firing time rather than fixed in the
  configuration.
- **Activation / deactivation:** consumer-invoked hooks that run for the state the machine already
  occupies, without a transition; introduced with snapshot restore (DD-10).
- **Error policy:** how failures are handled — `Throw` or `Observe`
  ([design-decisions.md](design-decisions.md), DD-13).
- **Firing mode:** *immediate* (single-threaded, reentrancy-protected) or *queued* (safe for
  concurrent callers) (DD-04).
- **Scheduler:** the Rx `IScheduler` that decides where and when notifications and timers run.
- **Virtual time:** a test scheduler's simulated clock, which makes timed behaviour deterministic.
- **Superstate / substate:** in a hierarchy, a state that contains other states / a state contained
  in another.
- **History (none / shallow / deep):** what a superstate remembers when it is re-entered.
- **Snapshot:** a saved description of a machine's position (and, in queued mode, pending triggers)
  that can be restored later.
- **Sample:** a runnable example project that also acts as an integration test.
