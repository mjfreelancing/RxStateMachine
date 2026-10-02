# Design Decisions

Cross-cutting behaviours that more than one feature depends on. Each is stated once here; feature
briefs in `features/` refer to them by ID (`DD-nn`) instead of repeating them. Names such as
`Transition` or `Fire` describe concepts; exact type and member names are settled at planning time.

## DD-01 Examples must obey the rules they illustrate
Sample code follows the library's own rules: async side effects use async entry actions (never
fire-and-forget), and timeouts are tied to a real state and cancelled when that state is left.

## DD-02 Trigger input types
Payload-free triggers can be pushed from any observable of triggers. Payload-carrying triggers use a
documented wrapper that pairs a trigger with its payload. Both paths feed the same transition
engine and behave identically to `Fire`/`FireAsync`.

## DD-03 Phasing
There are no milestones. Delivery order is the order of the feature briefs (`01` … `10`). A feature
may rely only on earlier features. The only milestone is the final deliverable.

## DD-04 Firing modes
Two modes only.
- **Immediate** (default): single-threaded and reentrancy-protected.
- **Queued**: an explicit opt-in that is safe for concurrent `Fire` from many threads; triggers are
  processed one at a time in arrival order. It is never enabled implicitly.
There is no separate worker-thread "active" mode.

## DD-05 Single source of requirements
Where a behaviour appears in several places it is defined once, in the owning feature brief or in
this file.

## DD-06 Guards and `otherwise`
Guards on the same trigger are evaluated one at a time, in registration order. The first guard that
passes decides the destination. An optional `otherwise` destination applies when none pass. If none
pass and there is no `otherwise`, the trigger is *blocked* — an outcome, not a fault (see DD-11). A
guard that throws is a *fault*, which is different from blocked.

## DD-07 Atomicity
If a guard throws, nothing changes. If an exit or transition action throws before the new position
is committed (DD-08), the position is unchanged. If a later action throws, the new position stays.
In every case the failure is handled by the error policy (DD-13) and all streams remain consistent
with the machine's actual position.

## DD-08 Order of operations and emission timing
Guards are evaluated first, before any action runs (DD-06), and each evaluation is reported on the
guard-result stream. If the trigger is permitted:

1. exit actions
2. transition action
3. new position committed; state and transition notifications emitted
4. general entry actions
5. trigger-specific entry actions
6. transition-completed notification emitted

## DD-09 Firing from inside an action
In immediate mode, a trigger fired from inside an action is queued behind the current firing and
runs when it completes. Its call returns a transition whose kind is `Queued`.

## DD-10 Disposal and lifecycle
Disposing the machine completes all streams, releases subscriptions and timers, and makes later
`Fire` calls throw `ObjectDisposedException`. Disposal is idempotent.

There is no separate start/stop lifecycle: a machine is usable from construction, and the only
lifecycle operations are disposing an input subscription (which stops that input only) and disposing
the machine. Activation and deactivation hooks are introduced with snapshot persistence (feature
08) — restoring a position is the one moment the machine occupies a state without a transition
having run — and their hierarchy semantics are defined there. Disposal does not run deactivation
hooks.

## DD-11 What `Fire` returns, and how failures surface
`Fire`/`FireAsync` return the resulting transition. Under the default `Throw` policy an unhandled
trigger throws. Under `Observe`, `Fire` returns a transition whose kind describes what
happened. The set of kinds is closed and decided at specification time; it includes at least:
occurred, reentrant, internal, unhandled, blocked, queued. *Unhandled* means the current state has
no registration for the trigger; *blocked* means a registration exists but no guard permitted it and
there is no `otherwise` destination (DD-06). Blocked is reported both on the guard-result stream and
in the returned transition, and throws only if the policy says so. A payload of the wrong type is a
caller mistake, not a machine outcome: it is rejected with an exception naming the expected and the
supplied type, under either policy.

## DD-12 What a transition and a state change carry
A transition carries: source state, destination state, trigger, payload (if any), kind, and when it
occurred. A state change carries: previous state, current state, when it occurred, and whether it was
a reentry or internal. Times come from the machine's injected scheduler/clock (DD-22). Whether a
duration is included is an open question for specification.

## DD-13 Error policies
- `Throw` (default): the failure reaches the caller.
- `Observe`: the failure is published as a rich error (trigger, state, exception, policy).
Failures from triggers that arrived as observable input have no caller and are always published on
the errors stream. Configuration errors throw at configuration time and never appear on the errors
stream. A handled failure never terminates the errors stream.

## DD-14 State authority with external storage
When external storage is configured, the consumer's store is the single owner of the current state;
the machine keeps no second copy. State is read before each decision and written as part of a
transition; if the write fails, the transition is not published.

## DD-15 When permitted triggers re-emit
On every state change, and whenever guard-dependent availability is re-evaluated after an internal or
reentrant transition. The on-demand query always evaluates current guards.

The stream re-emits on machine events only; it does not watch the data a guard reads. A consumer
whose guards depend on data that changes without a transition MUST re-query (or drive a re-emission
itself) when that data changes — this is documented, not solved in the core.

## DD-16 Cancellation versus error policy
Cancellation takes precedence over the error policy. A cancelled awaited firing surfaces the standard
cancellation exception to its caller regardless of policy and is not counted as a failure on the
errors stream. Synchronous `Fire` against a configuration that requires async work is rejected
rather than blocking.

## DD-17 Timers
A state timeout is armed on entry and cancelled on exit. Reentry restarts it; an internal transition
leaves it untouched. Pending timers are not persisted. If a timeout races with an explicit trigger,
whichever the machine processes first wins; the other is treated as an ordinary trigger in the
resulting state. (Confirmed at clarification time.)

## DD-18 Hierarchy
The current state is always a leaf. A substate's definition takes precedence over its superstate's
for the same trigger. Exits run innermost-first and entries outermost-first; the lowest common
ancestor is neither exited nor entered. History (none / shallow / deep) determines which substate is
re-entered. Each state has one parent (a tree; no orthogonal regions). Hierarchy is validated at
configuration time and closed after the first trigger.

## DD-19 Persistence restore
A snapshot is versioned. Restore is atomic (all or nothing), validates structure (an unknown state is
refused), runs no entry/exit actions and emits no historic transitions. The library ships no
serializer or storage, and timers are not captured. Persisting on change needs no dedicated API — it
is an ordinary subscription.

Serialization options, for future consideration (none is adopted; the core takes no serializer
dependency):
- expose the snapshot as a plain, storage-neutral data shape so any serializer can handle it;
- `System.Text.Json` (in the platform on modern targets; a package on `netstandard2.0`);
- binary formats such as MessagePack or protobuf-net for compact or high-throughput storage;
- schema-first formats (e.g. Avro/Protobuf) where snapshots must be read by non-.NET systems;
- ship any adapters as separate optional packages or documentation examples, never in the core.

## DD-20 Diagram export
Export is deterministic (same configuration → identical text, stable ordering), read-only (never
changes machine state), escapes special characters in names, offers a minimal and a detailed level,
and the Mermaid and D2 outputs carry equivalent content. The formatters share a format-neutral
structure so a further format can be added without changing the machine description or the existing
formatters. The text report is produced from the same introspection data as the machine description.

## DD-21 Concurrency test guarantees
Stress runs log their random seed so a failure can be replayed. Invariants: no lost or duplicated
triggers; the observed transition sequence has no gaps (each transition's source equals the previous
destination); no unexpected exceptions. Queue depth and idleness are observable, shutdown is clean,
and queued triggers can be cancelled.

## DD-22 How targets are measured
- **Allocation:** zero bytes allocated per steady-state simple transition, as reported by the
  benchmark memory diagnoser.
- **Latency:** judged by a before/after benchmark comparison run on the same machine when a feature
  touches the hot path — never an absolute figure tied to a named machine, and not against a stored
  baseline or a CI threshold.
- **Coverage:** line coverage of the core engine assembly from the CI test run.
- **Samples:** every sample project in the repository builds and runs in CI.
- **Time:** timestamps and timeouts come from the injected scheduler/clock, so tests use virtual time.
- Acceptance criteria should be automatable where practical; where they cannot be, the limitation is
  stated rather than hidden.
