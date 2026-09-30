# RxStateMachine Documentation

## For anyone

| Document | What it covers |
| -------- | -------------- |
| [vision.md](vision.md) | Vision, problem, goals, objectives, non-goals |
| [observable-model.md](observable-model.md) | The producer/consumer model, streams, worked examples |
| [use-cases.md](use-cases.md) | The ten example scenarios the library must support |
| [design-rationale.md](design-rationale.md) | Why the hybrid design; ideas borrowed from other libraries |
| [glossary.md](glossary.md) | Terms |
| [references.md](references.md) | Related libraries and tools |
| [diagrams.md](diagrams.md) | Example diagram exports |
| [saga-exploration.md](saga-exploration.md) | An open question kept out of the core |

## For building the library

| Document | What it covers |
| -------- | -------------- |
| [roadmap.md](roadmap.md) | Feature delivery order, risks, definition of a successful final deliverable |
| [design-decisions.md](design-decisions.md) | Cross-cutting behaviours (`DD-nn`) that features rely on |
| [engineering-standards.md](engineering-standards.md) | Engineering rules; the input to the project constitution |
| [api-sketch.md](api-sketch.md) | Illustrative, non-normative API and package layout |
| [features/](features/) | One brief per feature, in delivery order; each is the input to one specification run |

## Using these documents with Spec Kit

1. Run the constitution step using `engineering-standards.md` as its input.
2. For each brief in `features/`, in numeric order, run the specify step using the brief's
   "Feature input" section, then clarify using its "Open questions", then plan and tasks.
3. Requirement IDs in briefs use a feature prefix (for example `CORE-04`); decisions are `DD-nn`.
   Neither changes when a document is edited, so specs can cite them.
