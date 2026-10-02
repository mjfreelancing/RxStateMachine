# 07 — Hierarchical States

**Builds on:** 02–05. **Prefix:** `HIE`.

## Feature input (paste into `/speckit-specify`)

Let states nest. A state can be declared a substate of a superstate, inherits the superstate's
transitions, and can be asked whether the machine is "in" a superstate whenever it is in any of its
substates. Entering and leaving nested states runs exit and entry actions in a well-defined order, and
a superstate can remember which substate was active (none, shallow, or deep history) so re-entering it
returns to the right place.

## Requirements
- HIE-01 A state MUST be declarable as a substate of a superstate.
- HIE-02 A superstate MUST be able to declare an initial substate used when it is the destination of a
  transition.
- HIE-03 History (none, shallow, deep) MUST determine which substate is re-entered (DD-18).
- HIE-04 The "is in state" query MUST be true for the current state and every ancestor.
- HIE-05 The current state MUST always be a leaf (DD-18).
- HIE-06 A transition defined on a substate MUST take precedence over the same trigger on its
  superstate; a trigger unhandled by the substate MUST fall back to the superstate.
- HIE-07 Exits MUST run innermost-first and entries outermost-first; the lowest common ancestor MUST
  neither be exited nor entered.
- HIE-08 Reentry and internal transitions MUST behave consistently with nesting (rules stated in the
  specification).
- HIE-09 Each state MUST have at most one parent; cycles are forbidden.
- HIE-10 Hierarchy configuration MUST be validated at configuration time (unknown superstate, cycles,
  a superstate used as a destination without an initial substate) and closed after the first trigger.
- HIE-12 Introspection (permitted triggers, machine description) MUST include inherited transitions
  and the hierarchy.
- HIE-13 The remembered-history value MUST be expressible in a storage-neutral form for snapshots
  (feature 08).

## Edge cases and failure modes
- HIE-E1 A transition between sibling substates.
- HIE-E2 A transition from a substate to its own superstate and from a superstate to one of its
  substates.
- HIE-E3 History requested when the superstate has never been visited (falls back to the initial
  substate).
- HIE-E4 Deep history remembering a path through several levels.
- HIE-E5 A guard on a superstate's transition when the trigger arrives in a substate.
- HIE-E6 An internal transition defined on a superstate and triggered from a substate.
- HIE-E7 A superstate with no initial substate that is never a destination (allowed) versus used as a
  destination (error).
- HIE-E8 Deeply nested hierarchies (several levels) still evaluate in order.
- HIE-E9 A state name reused in two branches is rejected or disambiguated (state identity is global).

## Use-case coverage
UC-3 (draft-of-published) — see [use-cases.md](../use-cases.md).

## Design decisions relied on
DD-06, DD-07, DD-08, DD-10, DD-18.

## Open questions (for `/speckit-clarify`)
- Exit/entry rule for a transition from a substate to an ancestor, and for reentry of a superstate.
- Whether the wording "a superstate used as destination without an initial substate is an error" also
  covers unresolvable composite destinations.
- Whether orthogonal regions are permanently out of scope (currently: yes).

## Out of scope
Orthogonal (parallel) regions, snapshot persistence (08), export rendering (09), and
activation/deactivation hooks — those are introduced with snapshot restore in feature 08, where
their hierarchy semantics are defined (DD-10). (`HIE-11` held that requirement; the identifier is
retired, not reused.)
