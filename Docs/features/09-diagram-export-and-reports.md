# 09 — Diagram Export and Reports

**Builds on:** 04, 07. **Prefix:** `EXP`.

## Feature input (paste into `/speckit-specify`)

Let developers turn a machine's configuration into documentation: diagrams in Mermaid and D2 text
formats they can paste into pull requests and docs, and a plain-text report listing states,
transitions, guards, and actions. Output is deterministic and read-only, and is produced from the
machine-description data provided by the introspection feature. The formats are built on a shared,
format-neutral structure so that another format can be added later without changing the machine
description or the existing formatters.

## Requirements
- EXP-01 A machine MUST be exportable as Mermaid state-diagram text.
- EXP-02 A machine MUST be exportable as D2 text.
- EXP-03 A plain-text report of states, transitions, guards, and actions MUST be available.
- EXP-04 Output MUST be deterministic: the same configuration produces byte-identical text with a
  stable ordering (DD-20).
- EXP-05 Export MUST be read-only and MUST NOT change the machine or run actions or guards.
- EXP-06 Special characters in state or trigger names MUST be escaped so the output remains valid.
- EXP-07 A minimal and a detailed level MUST be offered; the detailed level annotates guards,
  actions, and payloads.
- EXP-08 Mermaid and D2 outputs MUST carry equivalent content.
- EXP-09 Nested states MUST be rendered as nesting in both formats.
- EXP-10 Export MUST use only the machine description as its data source (feature 04).
- EXP-11 The formatters MUST share a format-neutral structure, so that adding a further format means
  adding one formatter and changing neither the machine description nor the existing formatters.
  Mermaid and D2 are both built this way from the start.
- EXP-12 Output for representative machines MUST be covered by snapshot (golden-file) tests, and a
  snapshot change MUST be reviewed deliberately. Validating output with the formats' own tools in CI
  is not required.

## Edge cases and failure modes
- EXP-E1 Names containing quotes, brackets, colons, arrows, newlines, or non-Latin characters.
- EXP-E2 Very long names and very large machines (hundreds of states).
- EXP-E3 A machine with one state and no transitions; a machine with unreachable states.
- EXP-E4 Self-transitions, reentry, and internal transitions render distinctly.
- EXP-E5 A dynamic destination cannot be resolved statically and is shown as such.
- EXP-E6 Multiple guarded permits for one trigger appear as separate labelled edges.
- EXP-E7 Two states whose names differ only in case or whitespace.
- EXP-E8 Culture settings never change output (invariant formatting).

## Use-case coverage
UC-1 — see [use-cases.md](../use-cases.md) and the sample outputs in [diagrams.md](../diagrams.md).

## Design decisions relied on
DD-20.

## Open questions (for `/speckit-clarify`)
- Final detail-level content and label wording.
- Is the format-neutral seam internal only, or a public extension point for consumers' own formats?

## Out of scope
A visual editor, image rendering, and delivering any format other than Mermaid, D2, and plain text
(the design must still allow further formats to be added).
