# PRD Traceability Matrix (temporary — deleted together with the PRD)

Source: `RxStateMachine PRD DRAFT.md`. Purpose: prove nothing is lost before the PRD is deleted.
Status: `done` = content present in the destination (verified by the coverage check); `changed` =
present but deliberately altered (see §6); `dropped` = intentionally removed (see §6).

Feature briefs live in `Docs/features/` (`01` foundations … `10` concurrency-and-hardening). New
requirement IDs use the brief's prefix (`FND`, `CORE`, `OBS`, `INS`, `ASY`, `TES`, `HIE`, `PER`,
`EXP`, `HRD`). `STD` = `Docs/engineering-standards.md` (section number in parentheses). `DD` =
`Docs/design-decisions.md`.

## 1. Sections

| PRD heading | Destination | Status |
| --- | --- | --- |
| Title / status block | `Docs/README.md`, `README.md` badges | done |
| 1 Executive Summary | `vision.md` | done |
| 2 Vision | `vision.md`, `README.md` | done |
| 3 Background & Problem (3.1, 3.2) | `vision.md` | done |
| 3.3 Why the hybrid architecture (benefits, caveats, stream-reduction sidebar) | `design-rationale.md` | done |
| 4.1 Goals | `vision.md` | done |
| 4.2 Objectives | `vision.md` (O1 reworded, no milestone) | changed |
| 4.3 Non-goals | `vision.md`, `engineering-standards.md` (10), briefs "Out of scope" | done |
| 5.1 Core mental model | `observable-model.md` | done |
| 5.2 Architecture baseline | `observable-model.md`, `engineering-standards.md` (1) | done |
| 5.3 Streams table | `observable-model.md`, OBS-01..06 | done |
| 5.4 Worked examples + Scan design note | `observable-model.md` (Example 2 corrected), `design-rationale.md` | changed |
| 6 / 6.1 Design influences | `design-rationale.md` | done |
| 6.2 Saga | `saga-exploration.md` | done |
| 7.1 / 7.2 Requirements | §2 below | done |
| 8 Use cases + 8.1 examples | `use-cases.md` (UC-2 corrected) | changed |
| 9 Complexity, milestones, why this order | `roadmap.md` (milestones/sprints dropped) | changed |
| 10.1 – 10.3 Architecture sketch | `api-sketch.md` | done |
| 10.4 Design principles | `design-rationale.md`, `engineering-standards.md` (1) | done |
| 11 Testing & Quality | `engineering-standards.md` (5) | done |
| 12 Risks | `roadmap.md` | done |
| 13 KPIs | `roadmap.md` (definition of successful deliverable), `engineering-standards.md` | changed |
| 14.1 – 14.4 Baseline | duplicates of §7; mapped in §2 | done |
| 15.1 Glossary | `glossary.md` | done |
| 15.2 Example exports | `diagrams.md`, EXP briefs | done |
| 15.3 References | `references.md` | done |

## 2. Requirement mapping

| PRD ID | New IDs / location |
| --- | --- |
| FR-1 | CORE-01 |
| FR-2 | CORE-03, CORE-04, CORE-05, CORE-07, CORE-08, CORE-09 |
| FR-3 | CORE-10, CORE-12 |
| FR-4 | CORE-11 |
| FR-5 | CORE-05, CORE-06; DD-06 |
| FR-6 | CORE-14 |
| FR-7 | CORE-08 |
| FR-8 | CORE-09 |
| FR-9 | ASY-01 |
| FR-10 | CORE-16, CORE-17, HIE-10 |
| FR-11 | OBS-01 |
| FR-12 | OBS-02, OBS-03, OBS-04, OBS-05, OBS-06 |
| FR-13 | OBS-08, OBS-09 |
| FR-14 | OBS-11, OBS-12, OBS-13, OBS-14, OBS-15, CORE-15 |
| FR-15 | OBS-16, OBS-17 |
| FR-16 | INS-01, INS-02 |
| FR-17 | INS-03 |
| FR-18 | INS-04 |
| FR-19 | ASY-07, ASY-08, ASY-09, ASY-10, ASY-11, ASY-12; DD-13 |
| FR-20 | ASY-15 |
| FR-21 | TES-08, TES-09, TES-10; DD-14 |
| FR-22 | PER-01, PER-02, PER-03, PER-04, PER-05 |
| FR-23 | PER-06 |
| FR-24 | TES-02, TES-03, TES-04, TES-05; DD-17 |
| FR-25 | HIE-01, HIE-02, HIE-03 |
| FR-26 | HIE-04, HIE-07 |
| FR-27 | EXP-01, EXP-02 |
| FR-28 | EXP-03 |
| FR-29 | FND-01 |
| FR-30 | CORE-15; DD-11 |
| FR-31 | OBS-06, ASY-09 |
| NFR-1 | FND-03; STD (1, 8) |
| NFR-2 | CORE-01, FND-07; STD (2, 3) |
| NFR-3 | CORE-21, HRD-01, HRD-02; DD-04 |
| NFR-4 | ASY-02; STD (4) |
| NFR-5 | HRD-10; STD (6) |
| NFR-6 | FND-05, OBS-16, OBS-17, TES-04, HRD-09; STD (4) |
| NFR-7 | FND-11, FND-19, HRD-11, HRD-12; STD (10) |
| NFR-8 | FND-02, FND-04; STD (8) |
| NFR-9 | FND-10; STD (2) |
| NFR-10 | ASY-03 |
| NFR-11 | ASY-02; STD (4) |
| G1–G6, O1–O6 | `vision.md` |
| UC-1 … UC-10 | `use-cases.md`; coverage cited in each brief's "Use-case coverage" |

## 3. Enumerated content

| Content | Destination | Status |
| --- | --- | --- |
| Non-goals (5) | `vision.md` | done |
| Benefits table (6 rows) + caveats (2) | `design-rationale.md` | done |
| Streams table (7 rows) | `observable-model.md` | done |
| Ecosystem feature groups (4) | `design-rationale.md` | done |
| Complexity tiers (3) + why this order (4) | `roadmap.md` | done |
| Risks (6) | `roadmap.md` | done |
| KPIs (4) | `roadmap.md` | changed |
| Testing strategy (7 bullets) | `engineering-standards.md` (5) | done |
| Design principles (4) | `design-rationale.md` | done |
| Glossary (8 terms) | `glossary.md` | done |
| Saga open question + exploration plan | `saga-exploration.md` | done |

## 4. Fenced blocks (12)

| PRD lines | Content | Destination | Status |
| --- | --- | --- | --- |
| 186–214 | Mermaid flowchart | `observable-model.md` | done |
| 259–277 | Example 1 | `observable-model.md` | done |
| 282–291 | Example 2 | `observable-model.md` (corrected) | changed |
| 296–304 | Scan design note | `design-rationale.md` | done |
| 500–520 | UC-2 sample | `use-cases.md` (corrected) | changed |
| 524–536 | UC-5 sample | `use-cases.md` | done |
| 577–594 | Package tree | `api-sketch.md` | changed |
| 598–630 | Core type sketch | `api-sketch.md` | changed |
| 634–645 | Configuration surface | `api-sketch.md` | done |
| 770–780 | Order lifecycle config | `diagrams.md` | done |
| 784–795 | Mermaid stateDiagram | `diagrams.md` | done |
| 799–816 | D2 sample | `diagrams.md` | done |

Also the stream-reduction sidebar (fold trace + `Scan` sample) → `design-rationale.md` (done).

## 5. External links (7)

d2lang.com, Stateless, Appccelerate, Automatonymous/MassTransit, WorkflowCore, System.Reactive, FsCheck →
all in `references.md` (done).

## 6. Deliberate changes and losses (approved during review)

- **Milestones and sprints dropped:** M0–M4 table, "Target" column, "Sprint 1…12" estimates, and the
  "(M2)/(M3)" tags on requirement headings. Replaced by the ordered feature list in `roadmap.md`.
- **Section-number cross-references dropped** (for example "see §14.4"); links point to the new docs.
- **§14 Baseline** not kept as a section (it repeated §7); every bullet is mapped in §2.
- **PRD wording on "v1"** replaced by "the final deliverable"; O1 reworded accordingly.
- **Example 2 and UC-2 samples corrected** (fire-and-forget entry action, uncancelled free-standing timer,
  non-existent state); rules in DD-01 and DD-17.
- **Observer-input type ambiguity** (`IObserver<TTrigger>` vs `IObserver<TriggerWithParameters<…>>`)
  recorded as DD-02 and left open in `api-sketch.md`.
- **"Active" worker-thread mode** replaced by a single opt-in queued mode (DD-04).
- **Errors stream** no longer includes configuration errors (DD-13).
- **Feature-coverage KPI** ("completed for M1") reworded to "for the most-used constructs".

## 7. Deletion gate
The PRD may be deleted only when: the coverage check passes with zero unmapped items; the user has
approved the re-organised documents; the final state is committed and tagged `prd-final`. This file and
`Docs/decisions.md` are deleted in the same commit as the PRD.
