# Decision Log (review record — temporary, deleted with the PRD)

## Review outcome
- **Accepted as proposed:** D-01, D-02, D-04 to D-18, D-20, D-21, D-22.
- **D-03 revised:** no milestones. Phasing = order of the feature slices only.
- **D-19 accepted** with an added note: suggested serialization options for future consideration.
- **D-23 revised:** ignore the PRD's structure. No milestone concept at all; features are sequenced and delivered in order; the only milestone is the final deliverable; no time constraints.
- **D-24 revised:** new ID convention chosen by me (feature-area prefixes, e.g. `CORE-01`); the PRD is not referenced from any retained document.
- **D-25 accepted:** GitHub Actions is the CI provider; specs/docs must assume no prior GitHub Actions knowledge so a human can review with confidence.
- The permanent record of the accepted semantics is `Docs/design-decisions.md` (no PRD/archive references).

## Original log (kept for traceability until deletion)

Every item below is a PRD defect, gap, or undefined behaviour that must be settled **before** the feature
briefs are written, so each brief states one agreed answer instead of repeating an inconsistency.

**Status legend:** `proposed` = my recommendation, needs your answer · `decided` = you confirmed.
**Source tags:** `[PRD]` current PRD · `[ARCH-C]` Claude archive specs/constitution · `[ARCH-P]` Copilot archive decision log/specs · `[NEW]` best practice.
Archive items are candidates only; none is adopted until you mark it `decided`.

Reply with, for example, `D-01 to D-05 accept, D-06 change to …, D-11 discuss`.

---

## A. PRD defects to correct

### D-01 Worked examples contradict the requirements — `proposed`
- **Issue:** UC-2 (§8.1) uses `OnEntry(_ => _ = AuthorizeAsync())` (fire-and-forget, violates NFR-4), a 30 s timer that is never cancelled on leaving the state (violates FR-24), and a comment about an "Authorizing" state that does not exist. Example 2 (§5.4) has a 10-minute `Observable.Timer` not tied to any state, so it fires even after the order is paid.
- **Proposal:** rewrite both with `OnEntryAsync`, a real `Authorizing` state, and the FR-24 timeout helper; keep the original text in `Docs/prd-traceability.md` notes only. Samples in `Docs/use-cases.md` become the corrected versions.

### D-02 Observer input type and payload type disagree — `proposed`
- **Issue:** §10.1 says `IObserver<TTrigger>`; §5.2, §10.2, §14.1 say `IObserver<TriggerWithParameters<TState,TTrigger>>`. FR-6 defines `TriggerWithParameters<TTrigger,TPayload>` (different shape). Examples pipe a bare `OrderTrigger` into the machine, which matches neither.
- **Options:** (a) one observer type accepting a plain trigger, with payload triggers via a separate wrapper; (b) single wrapper type for everything; (c) the machine exposes `AsObserver()` adapters instead of implementing `IObserver` itself.
- **Proposal (a):** `IObserver<TTrigger>` for payload-free triggers plus a documented wrapper for payload-carrying triggers; type names remain illustrative until planning. [PRD] [ARCH-P 008]

### D-03 Milestone phasing contradicts itself — `proposed`
- **Issue:** FR-22/23 headed "(M2)" but delivered in M3 (§9.1); FR-25/26 headed "(M2)" and delivered in M3; FR-14 (observer input) is "fully" M2 but basic use is needed for M1 samples; FR-18 (guard descriptions/GuardResult) is M2 while FR-5/FR-12 imply it earlier; FR-20 lifecycle events phase unclear; `IsInState` is M3 in §10.2 but hierarchy is M2 in §7.
- **Proposal:** phasing is defined only by the feature slice order (F01…F10). Requirement text carries no milestone tag; `Docs/delivery-plan.md` keeps the original M0–M4 table as history. Basic observer input and GuardResult ship with F03; the rest of the policies with F05. [ARCH-C 001 made the same correction.]

### D-04 "Queued" vs "active" vs "immediate" — `proposed`
- **Issue:** the PRD uses "queued/active mode" as one concept in NFR-3/O6 but §6.1 lists active (worker-thread) vs passive machines separately from `FiringMode.Queued`. [ARCH-C 003 vs 008] disagree on whether queued mode is a thread-safety guarantee.
- **Proposal:** two modes only. **Immediate** (default): single-threaded, reentrancy-protected. **Queued**: explicit opt-in, safe for concurrent `Fire`, triggers processed one at a time in arrival order. No separate worker-thread "active" mode in v1. [ARCH-P 015]

### D-05 §14 and NFR duplication — `proposed`
- **Issue:** §14.1–14.4 repeat FR-29/30/31 and NFR-2/6/8/11, with small wording drift.
- **Proposal:** treat §7 as authoritative; §14 is verified line-by-line against it (no unique content found so far except the explicit "source generation remains a non-goal, may be revisited") and then dropped. The traceability matrix marks each §14 bullet `done` only after the equivalent exists.

---

## B. Behaviours the PRD leaves undefined

### D-06 `otherwise`/default transition and guard evaluation — `proposed`
- **Issue:** FR-5 mentions an `otherwise` concept that appears nowhere else. [ARCH-P 004, S-00 D-05/D-06]
- **Proposal:** guards on the same trigger are evaluated sequentially in registration order; first passing guard wins; guard evaluation is never parallel; an optional `otherwise` destination applies when none pass; if none pass and no `otherwise` exists the trigger is "blocked" (an outcome, not a fault; see D-11). A guard that throws is a fault, distinct from "blocked".

### D-07 Atomicity when a guard or action throws — `proposed`
- **Issue:** not stated. [ARCH-C 003, ARCH-P S-00 D-03]
- **Proposal:** if a guard throws, no state change; if an exit/transition/entry action throws, the position is unchanged only if the failure occurs before the position is committed (see D-08 order); failures after commit leave the new position in place. Either way the failure is handled per the error policy (D-13) and streams stay consistent.

### D-08 Ordering of operations and emission timing — `proposed`
- **Issue:** PRD does not say whether `Transitions` emits before or after entry actions, when the state observable updates, or when `TransitionsCompleted` fires (only "after the last entry action").
- **Proposal:** exit actions → transition action → position committed and state/Transition emitted → general entry actions → trigger-specific entry actions → `TransitionsCompleted` emitted. [ARCH-P B2, S-00 D-02]

### D-09 `Fire` from inside an action, and re-entrancy — `proposed`
- **Issue:** PRD only says immediate mode is "reentrancy-protected".
- **Proposal:** in immediate mode a `Fire` issued from within an action is queued behind the current firing and returns a transition whose kind is `Queued`; it runs after the current firing completes. [ARCH-P B11]

### D-10 Dispose/stop semantics and activation hooks — `proposed`
- **Issue:** no dispose behaviour; FR-3 lists activation/deactivation without saying when they run before hierarchy exists.
- **Proposal:** disposing completes all streams, releases subscriptions and timers, and later `Fire` throws `ObjectDisposedException`; dispose is idempotent. Activation/deactivation hooks are accepted in F02 and are only meaningful once hierarchy exists (F07); before then they run on machine start/stop. Stopping a machine is not disposing it. [ARCH-P B4, S-00 D-09/D-10] [NEW]

### D-11 What `Fire` returns and how failures surface — `proposed`
- **Issue:** FR-30 says `Fire` returns the resulting `Transition`, but for unhandled/blocked/ignored triggers the PRD does not say whether it returns or throws. Copilot archive uses a closed `Kind` set: Occurred, Reentrant, Internal, Unhandled, Refused, Queued.
- **Proposal:** `Throw` policy (default) → an unhandled trigger throws; `Ignore`/`Observe` → `Fire` returns a `Transition` describing the outcome. Blocked-by-guard is an outcome exposed on the `GuardResult` stream and the returned transition, not an exception unless the policy says so. Final kind set decided in F02/F05 specify. [ARCH-P B1, B7]

### D-12 Shape of `Transition` and `StateChange` — `proposed`
- **Issue:** §5.3 (Source, Destination, Trigger, payload, kind, duration) vs §10.1 (adds Timestamp; StateChange has Previous, Current, Timestamp, Kind vs IsReentry/IsInternal). [ARCH-P 001 fixed exactly six members and dropped duration.]
- **Proposal:** describe by information carried (source, destination, trigger, payload, kind, when it occurred) not by member count; whether duration is included is an open question for F03 clarify. Timestamps come from the machine's injected clock/scheduler (see D-22).

### D-13 Error policy semantics — `proposed`
- **Issue:** FR-19 names Throw/Ignore/Observe but not what each retains, or what happens when input arrived via an observable (no caller to throw to). Errors stream is also said to carry "configuration errors" (contradicts fail-fast at configuration time).
- **Proposal:** three policies. `Throw` (default) reaches the caller. `Observe` publishes `StateMachineError` (trigger, state, exception, policy). `Ignore` absorbs the failure but keeps a bounded record and a count. Failures from observable input have no caller and are always published on the errors stream. Configuration errors throw at configuration time and never appear on the errors stream. The errors stream never terminates because of a handled failure. [ARCH-P A2/B6, ARCH-C 003]

### D-14 State authority when external storage is configured — `proposed`
- **Issue:** "state is owned in one place" (§10.4) vs FR-21 external accessor/mutator.
- **Proposal:** with external storage the consumer's store is the single owner and the machine keeps no separate copy; the state is read before each decision and written as part of a transition; a write failure means the transition is not published. [ARCH-P A1, ARCH-C 004]

### D-15 When `PermittedTriggers` re-emits — `proposed`
- **Issue:** §5.3 says "re-emitted when the state changes" but permitted triggers can depend on guards.
- **Proposal:** re-emit on every state change and whenever guard-dependent availability is re-evaluated after an internal or reentrant transition; `GetPermittedTriggers()` always evaluates current guards. [ARCH-P S-00 D-08]

### D-16 Cancellation vs error policy — `proposed`
- **Issue:** NFR-10 gives `CancellationToken` without semantics.
- **Proposal:** cancellation takes precedence over error policy; a cancelled awaited firing surfaces `OperationCanceledException` to its caller regardless of policy and does not count as a failure on the errors stream; sync `Fire` against an async-only configuration is rejected rather than blocking (NFR-4). [ARCH-C 003]

### D-17 Timer semantics and the timeout/trigger race — `proposed`
- **Issue:** FR-24 says only "auto-cancel when the state is left". [ARCH-C 004]
- **Proposal:** a state timeout is armed on entry and cancelled on exit; reentry restarts it; an internal transition leaves it untouched; pending timers are not persisted; if a timeout and an explicit trigger race, whichever the machine processes first wins and the other is treated as an ordinary trigger in the new state (no special handling). Race rule is an F06 clarify item.

### D-18 Hierarchy semantics — `proposed`
- **Issue:** FR-25/26 are two lines.
- **Proposal:** current state is always a leaf; substate definitions take precedence over the superstate's for the same trigger; exits run innermost-first and entries outermost-first, the lowest common ancestor is neither exited nor entered; history None/Shallow/Deep defined by which substate is re-entered; single parent per state (tree, no orthogonal regions); hierarchy configuration is validated at configuration time and closed after the first trigger. [ARCH-C 005, ARCH-P 013]

### D-19 Persistence restore semantics — `proposed`
- **Issue:** FR-22 lists what a snapshot holds, not restore rules.
- **Proposal:** a snapshot is versioned; restore is atomic (all or nothing), validates structure (unknown state ⇒ refusal), runs no entry/exit actions and emits no historic transitions; the library ships no serializer or storage; timers are not captured; "persistence as an observer" (FR-23) needs no dedicated API. [ARCH-C 006, ARCH-P 011]

### D-20 Diagram export determinism — `proposed`
- **Issue:** FR-27/28 say nothing about stability or escaping.
- **Proposal:** export is deterministic (same configuration ⇒ identical text, stable ordering), read-only (never changes machine state), escapes special characters in state/trigger names, supports a minimal and a detailed level, and Mermaid and D2 outputs carry equivalent content; the text report derives from the same introspection data (`GetInfo`). [ARCH-C 007, ARCH-P 014]

### D-21 Concurrency test reproducibility and history invariants — `proposed`
- **Issue:** O6 gives "8 workers × 10k triggers" but no invariants or reproducibility.
- **Proposal:** stress runs log their random seed so a failure is replayable; invariants: no lost or duplicated triggers, the observed transition sequence has no gaps (each transition's source equals the previous destination), no unexpected exceptions; queue depth/idle and clean shutdown observable; cancellation of queued triggers supported. [ARCH-C 008]

---

## C. Targets and conventions that need a definition

### D-22 Measurable targets need a measurement definition — `proposed`
- **Issue:** "microseconds", "no allocation", "80% coverage", "≥ 8 samples", "regression threshold" have no measurement method. [ARCH-P flagged the same problem.]
- **Proposal:** allocation = zero bytes allocated per steady-state simple transition as reported by BenchmarkDotNet memory diagnoser; latency = tracked against a stored baseline with a relative regression threshold rather than an absolute number on a named machine; coverage = line coverage of the core engine assembly from the CI test run; samples = count of sample projects in the repo that build and run in CI. Timestamps and timeouts use the injected scheduler/clock. Adopt "automatable where practical" wording rather than a blanket "no human judgement" principle. [NEW] [ARCH-P Principle VII, scaled back]

### D-23 v1 scope — `proposed`
- **Issue:** G4/O2 promise "most-used constructs" while hierarchy, persistence, export and timers are also listed; risk register flags over-engineering.
- **Proposal:** treat F01–F05 as the v1 core; F06–F10 stay in scope but sequenced after, each independently shippable. The saga layer stays out (Docs/saga-exploration.md). Sprint numbers are dropped; the M0–M4 order is retained as sequencing guidance. Confirm whether F09 (export) or F07/F08 may slip past v1.

### D-24 Requirement identifiers and brief conventions — `proposed`
- **Issue:** archives restarted `FR-001` in every spec, making bare IDs ambiguous; specs ballooned to 18–90 KB.
- **Proposal:** keep PRD IDs (`FR-n`, `NFR-n`, `UC-n`) as globally unique stable identifiers; new discrete requirements in a brief get sub-IDs (`FR-5.1`, `FR-5.2`); briefs are ≤ ~2 pages, cross-cutting semantics live here in the decision log once, and archive-derived items are tagged. [NEW]

### D-25 Constitution scope — `proposed`
- **Issue:** Copilot's Principle VII ("Verifiable by Machine") produced a large audit with self-inflicted defects; Claude's constitution left `TODO(CI_PROVIDER)` unresolved.
- **Proposal:** constitution input (`Docs/engineering-standards.md`) contains principles and gates only; feature-level numbers live in specs. Pick the CI provider now (recommend GitHub Actions since the repo is on GitHub) or leave as an explicit plan-time decision. Confirm.

---

## D. Not carried over (for the record)
- Sprint numbering (Sprint 1–12) and sprint-planning scaffolding.
- Per-spec "Inherited Constraints" blocks and cross-spec dependency numbering from the archives.
- Numeric success-criteria guesses from the archives (e.g. "detects 20% slowdown in 95% of trials").
- Verification-machinery specs (forbidden-token scans, byte allowlists, per-change-class duration budgets).
- Stock speckit templates/scripts in the archive folders.
