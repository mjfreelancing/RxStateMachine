# Sprint Template

> Copy this file to create a new sprint document, e.g. `Docs/Sprint Planning/S-XX-<short-slug>.md`.
> Fill every section. Keep `## Progress log / resume` updated as you work so interrupted work can be
> resumed from any machine.

---

## 1. Metadata

- **Sprint ID:** S-XX
- **Title:** <short title>
- **Status:** Draft / Ready / In Progress / Blocked / Done
- **PRD references:** <FR-x, NFR-x, UC-x, PRD §x — be specific>
- **Prerequisites:** <sprint IDs this depends on>
- **Depends-on (future sprints that need this):** <sprint IDs>
- **Owner:** TBD

---

## 2. Outcome

One or two sentences: what this sprint exists to achieve, and why it matters to the overall plan.

### 2.1 Definition of Done

Objective, verifiable criteria — a checklist that decides "this sprint is finished."

- [ ] <criterion>
- [ ] <criterion>

---

## 3. Concepts & Vocabulary

> Plain-language explanations of **every** state-machine / Rx term used in this sprint, each with a
> tiny concrete example. Assume the developer is **not fluent in state machine terminology**. Add new
> terms here; don't assume prior sprints' vocabulary was memorized (link back where useful).

### <Term>

- **What it is:** <plain-language definition>
- **Example:** <tiny example>

### <Term>

- ...

---

## 4. Tasks

> One block per task. Tasks should be small, verifiable, and build on each other within the sprint.

### T1 — <short title>

- **Goal:** <what we're trying to achieve>
- **Work to perform:** <step-by-step what to build/do>
- **Files / locations:** <paths>
- **Acceptance criteria:** <how to verify it's done>
- **Tests:** <which tests to write / extend>
- **Samples:** <which sample project(s) to create / extend>

### T2 — <short title>

- ...

---

## 5. Example scenarios

> Fleshed-out worked examples that explain the features this sprint delivers. Prefer step-by-step
> walkthroughs with state tables and `mermaid` diagrams. Tie each to a PRD use case (UC-x).

### Scenario: <name> (UC-x)

- **Context:** <what the scenario is>
- **Configuration:** <code>
- **Walkthrough:** <step-by-step: trigger → guard → action → state, what each stream emits>
- **State table:**

| #   | Trigger | Guard result | Action(s) run | New state | Streams emitted |
| --- | ------- | ------------ | ------------- | --------- | --------------- |
| 1   | ...     | ...          | ...           | ...       | ...             |

- **Diagram:** <mermaid>

---

## 6. Tests & samples checklist

- [ ] <unit test / property test / reactive test for ...>
- [ ] <sample run and output shown>
- [ ] <coverage / CI gate if applicable>

---

## 7. Risks & tricky bits

- <risk or subtlety> → <mitigation / how to spot it>
- Point to PRD §12 where relevant.

---

## 8. Progress log / resume

> Update this table as you work. This is the first place to look when resuming interrupted work.

| Task | Status      | Notes |
| ---- | ----------- | ----- |
| T1   | Not started |       |
| T2   | Not started |       |

**If work is interrupted, resume here:**

- <what was in flight / what remains>

**Next actions on resume:**

1. <first thing to do>
2. <second thing>

**Checkpoint notes:**

- <decisions made mid-sprint, API refinements, anything future sprints must know>

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
