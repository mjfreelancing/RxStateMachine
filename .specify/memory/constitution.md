<!--
SYNC IMPACT REPORT (temporary — remove before committing)
Version change: (blank template) → 1.0.0
Modified principles: none (initial ratification)
Added principles:
  I. Observable-First, Framework-Agnostic Core
  II. Public API Discipline & Versioning
  III. Type Safety & Fail-Fast Configuration
  IV. Async, Concurrency & Determinism
  V. Test-First (NON-NEGOTIABLE)
  VI. Performance
  VII. Code Quality & Maintainability
  VIII. Documentation, Samples & Simplicity
Added sections:
  Platform, Packaging & Security Constraints
  Continuous Integration & Development Workflow
  Governance
Removed sections: none
Source: Docs/engineering-standards.md (sections 1-12 mapped: 1→I, 2→II, 3→III, 4→IV, 5→V,
  6→VI, 7→VII, 10→VIII, 8+9→Platform section, 11+12→Workflow section and Governance)
Templates reviewed, not modified (this command writes only the constitution):
  .specify/templates/plan-template.md, spec-template.md, tasks-template.md, checklist-template.md
Follow-up TODOs: none
-->

# RxStateMachine Constitution

## Core Principles

### I. Observable-First, Framework-Agnostic Core
- Every trigger, however it arrives (`Fire`/`FireAsync` or an observable), MUST flow through the
  same single transition engine; there MUST NOT be a second code path with different behaviour.
- Current state MUST be owned in exactly one place. Observable streams are outputs of the engine,
  never a fold recomputed from trigger history. With external state storage, the consumer's store
  is that single owner and the machine MUST NOT keep a copy.
- The public API MUST NOT expose `Subject<T>`, writable observers, or any mutable stream handle
  other than the documented trigger-input surface. Streams are exposed read-only
  (`AsObservable()`-style).
- State and transition streams MUST be hot and replay the latest value; all subscriptions MUST be
  disposable and MUST NOT leak. Emissions MUST honour the Rx grammar (serialized `OnNext`, no calls
  after `OnCompleted`/`OnError`).
- Callers who only use `Fire()` MUST get full functionality without needing to know Rx.
- The library MUST NOT depend on any application framework (UI, DI, messaging, storage).

Rationale: the hybrid of a classic transition engine and native observable streams is the
product's defining differentiator. Two behaviours for one trigger, or leaked internals, would
destroy the guarantees consumers rely on.

### II. Public API Discipline & Versioning
- Everything not intentionally public MUST be `internal`. New public types MUST default to
  `sealed` unless extension is a documented design goal.
- The package MUST follow Semantic Versioning 2.0.0. Public API changes MUST be tracked with a
  public-API analyzer (or an equivalent API-diff gate) so every addition or removal is visible in
  review.
- Before the first stable release the API is unfrozen and `[Obsolete]` MUST NOT be used. From the
  first stable release, a removal MUST go through `[Obsolete]` with a migration message for at
  least one MINOR release before removal in the next MAJOR.
- Public members MUST follow the .NET Framework Design Guidelines (naming, `Async` suffix,
  `CancellationToken` as the last optional parameter, no `async void`, no public fields).
- Public models (transitions, errors, events) MUST be immutable. Public APIs MUST NOT expose
  mutable collections or internal state, and MUST validate arguments at the boundary, naming the
  parameter.
- Types that own subscriptions or other disposable resources MUST implement `IDisposable` (and
  `IAsyncDisposable` where async cleanup is needed). Disposal MUST be idempotent; use after
  disposal MUST throw `ObjectDisposedException`.
- Library code MUST NOT let an exception thrown from an observer callback tear down the machine.

Rationale: the public surface is a contract with other developers; consumers can upgrade safely
only if it is small, explicit, and changes are detected mechanically rather than by reviewer
memory.

### III. Type Safety & Fail-Fast Configuration
- Nullable reference types MUST be enabled in all projects, public generics MUST use `notnull`
  where appropriate, and nullable warnings are errors.
- Invalid configuration MUST fail fast, at configuration time, with messages that name the state
  and trigger involved and say how to fix the problem.
- Every blocked transition MUST be explainable ("why was this blocked?").

Rationale: mistakes must surface at compile time or startup, never as production incidents.

### IV. Async, Concurrency & Determinism
- Library code MUST NOT block on async work (`.Result`, `.Wait()`, `.GetAwaiter().GetResult()`).
- Async APIs MUST be `Task`-based, accept an optional `CancellationToken`, and use
  `ConfigureAwait(false)` internally. `IAsyncEnumerable` MUST NOT be part of the public API.
- Schedulers MUST be injected (`IScheduler`, default `Scheduler.Default`), never ambient. There
  MUST be no static mutable state and no hidden dependence on wall-clock time, culture, or thread
  state.
- `TimeProvider` MUST NOT appear in the public API; it MAY be used for cosmetic timestamps behind
  `#if NET8_0_OR_GREATER`, with `DateTimeOffset.UtcNow` as the `netstandard2.0` fallback.
- Immediate mode MUST be single-threaded and reentrancy-protected. Concurrent `Fire` is supported
  only in the explicit queued mode, which MUST NEVER be enabled implicitly.
- Locks MUST NOT be held across an `await` or while invoking consumer-supplied code (guards,
  actions, observer callbacks). Lock objects MUST be private (never `lock(this)` or a public
  type). Shared state MUST be immutable, confined to one thread, or guarded by a documented
  mechanism.
- Every timed behaviour MUST be reproducible under Rx `TestScheduler` virtual time.

Rationale: blocking causes deadlocks and starvation in UI, server, and IoT hosts, and injectable
schedulers are what make deterministic virtual-time testing possible.

### V. Test-First (NON-NEGOTIABLE)
- Tests MUST be written before implementation: write the test, see it fail for the right reason,
  then implement (red → green → refactor). A bug fix MUST begin with a failing regression test.
- Unit tests MUST use xUnit.v3 (the `xunit.v3` packages) and MUST cover transition correctness,
  guard ordering, reentry, internal transitions, async actions and guards, error policies, and
  configuration validation.
- Time-based and stream-ordering behaviour MUST be tested with `TestScheduler`; tests MUST NOT
  rely on real sleeps or wall-clock timing.
- State-transition invariants MUST be covered by property-based tests (FsCheck), for example
  "never transition to an unconfigured state" and "permitted set matches configured transitions
  given guards".
- Concurrency features MUST ship with a stress test: many workers firing random valid and invalid
  triggers, started together with a barrier, asserting no lost updates, no unexpected exceptions,
  and a consistent final state. Failing runs MUST be replayable from a logged seed.
- Core-engine line coverage MUST be at least 80%. Coverage is a floor, not a substitute for
  behaviour tests. Tests MUST run against every target framework.
- Tests MUST be independent and order-insensitive, MUST assert observable behaviour rather than
  implementation details, and MUST have names that state scenario and expected outcome. Flaky
  tests MUST be fixed or removed, never retried or ignored.
- Mutation testing (for example Stryker.NET) SHOULD be run periodically on the core engine.
- Every sample project MUST build and run in CI; samples double as integration tests.

Rationale: a state machine engine is a correctness product; subtle bugs in reentrancy,
concurrency, and hierarchy are cheap to prevent and expensive to diagnose downstream.

### VI. Performance
- A simple synchronous transition with no actions MUST complete in microseconds and MUST NOT
  allocate in the steady state; any unavoidable allocation MUST be justified by benchmark
  evidence.
- Configuration is one-time work and MUST NOT be repeated per transition. Exposing the observable
  surface MUST NOT allocate per subscription beyond what the subscription needs.
- Hot-path performance MUST be tracked with BenchmarkDotNet against a stored baseline; a
  regression beyond the agreed relative threshold MUST block release. Optimisations that reduce
  clarity MUST be justified by benchmark results.
- How each performance target is measured is defined in `Docs/design-decisions.md` (DD-22).

Rationale: the machine is embedded in UI handlers, message pumps, and processing loops, where
avoidable allocation and per-transition setup show up as throughput loss and GC pressure.

### VII. Code Quality & Maintainability
- Code style MUST be defined in a committed `.editorconfig` and enforced in CI
  (`dotnet format --verify-no-changes` or equivalent); style is not debated in review.
- Methods and types MUST have a single, clear responsibility. Cyclomatic complexity and method
  length MUST be bounded by analyzer thresholds configured in the repository; a suppression MUST
  carry a written justification.
- Composition over inheritance, small focused abstractions, and immutability by default MUST be
  preferred. An abstraction, option, or extension point MUST NOT be added for a hypothetical need.
- Exceptions MUST be specific (no throwing or catching bare `Exception` except at a documented
  policy boundary), MUST NOT be swallowed silently, and messages MUST say what happened, which
  state and trigger were involved, and how to fix it.
- Compiler and analyzer warnings MUST be errors. Suppression MUST be as narrow as possible
  (scoped `#pragma` or attribute) with a justification, never global.
- Comments MUST explain why, not what. Dead code, commented-out code, and `TODO`s without a
  tracked issue MUST NOT be merged.
- Library code MUST NOT use reflection, `dynamic`, or static mutable state.
- Analyzers MUST be enabled at the latest recommended level or stricter, and centralised in
  repository-wide build properties.

Rationale: a library stays trustworthy only if quality rules are mechanical and consistent rather
than dependent on who reviews a change.

### VIII. Documentation, Samples & Simplicity
- 100% of public members MUST have XML documentation (a missing-doc warning is an error),
  including exceptions thrown and the threading and hot/cold behaviour of streams. The API
  reference MUST be updated in the same change that alters the public surface.
- Every use case in `Docs/use-cases.md` MUST have a runnable, documented sample; at least 8
  samples spanning web, IoT, payment, workflow, and UI domains MUST ship.
- The simplest design that satisfies the requirement MUST be preferred. Later features MUST NOT
  leak partial designs into earlier features' public API.
- The non-goals in `Docs/vision.md` MUST NOT be built into the core without a constitutional
  amendment. Saga-style usage is supported, but compensation, distributed coordination, and
  long-running workflow scheduling belong in a separate companion library.
- The repository `README.md` MUST describe the product only, every code example in it MUST
  compile and run in CI, and it MUST NOT link to internal planning documents.

Rationale: low-friction adoption depends on excellent documentation and a focused scope; scope
creep is a named project risk.

## Platform, Packaging & Security Constraints

- **Target frameworks:** `netstandard2.0`, `net10.0`, `net11.0`. `net8.0` is not targeted.
  Multi-targeting differences MUST be confined to language-feature shims (`IsExternalInit`,
  `RequiredMemberAttribute`) and narrow `#if` gates; public behaviour MUST be identical on every
  target.
- **Dependencies:** the only runtime dependency MUST be `System.Reactive` (6.x), plus the BCL and
  unavoidable compatibility backports. No DI, UI, messaging, ORM, serializer, or
  logging-framework dependencies; integrations are documentation examples only. Adding a runtime
  dependency or a target framework MUST be justified in the plan.
- **Package:** a single NuGet package named `RxStateMachine`, MIT licensed, with README, license
  expression, repository URL, and icon in the package metadata.
- **Reproducibility:** builds MUST be deterministic, with SourceLink and symbol packages. Package
  versions come from one source. Package validation against the previous release MUST run for
  stable releases. Packages MUST be published only from CI, never from a developer machine.
  Central package version management and repository-wide build properties MUST be used.
- **Security:** no dynamic code loading and no deserialization of untrusted input in the core.
  Dependencies MUST be scanned for known vulnerabilities in CI on every change and on a schedule.
  A `SECURITY.md` MUST describe how to report vulnerabilities. New dependencies MUST be justified
  and pinned centrally. Secrets MUST never be stored in the repository.

## Continuous Integration & Development Workflow

- **Platform:** CI runs on GitHub Actions. The repository owner has no prior GitHub Actions
  experience, so every CI artefact MUST be reviewable by a non-expert: each workflow file MUST
  have a comment block saying, in plain language, what it does, when it runs, and what it needs;
  each step MUST have a human-readable name and a one-line comment explaining why it exists; a
  guide MUST explain how to open a workflow run, read a pass/fail result, find a failing step's
  log, and re-run a job, with no assumed prior knowledge; and a glossary of the terms used
  (workflow, job, step, runner, trigger, secret, artefact) MUST be provided.
- **Gates:** on every proposed change CI MUST restore, build every target framework with zero
  warnings, run the full test suite on every target, check formatting, run analyzers, build and
  run the samples, pack, and validate the package. A red build MUST block merge.
- **Publishing:** MUST be a separate, explicit, CI-only step that is safe to re-run without
  creating a duplicate release. CI MUST NOT contain secrets in plain text; secrets use the
  platform's secret store.
- **Sequencing:** features are built in the order of `Docs/roadmap.md`. A later feature MUST NOT
  start before its prerequisites are complete.
- **Pull requests:** every change MUST arrive as a small pull request focused on one concern and
  meeting the Definition of Done: tests written first and passing, analyzers clean, public-API
  baseline updated, XML docs, samples, and changelog (Keep a Changelog format) updated, and no
  unjustified suppressions.
- **Review:** reviewers MUST check each change against this constitution; a violation requires a
  fix or a justified, recorded exception.
- **Commits and releases:** commits MUST be focused and messages MUST describe the why; the
  release process MUST tag the commit that produced the published package.

## Governance

This constitution supersedes other project practices. Where any planning document conflicts with
it, the constitution wins until amended.

- **Amendments:** proposed with a written rationale, reviewed, and recorded as a versioned change
  to this document. The version line below MUST be updated on every amendment.
- **Versioning policy:** MAJOR for backward-incompatible governance changes or principle
  removals/redefinitions; MINOR for a new principle or materially expanded guidance; PATCH for
  clarifications, wording fixes, and other non-semantic refinements.
- **Compliance review:** each feature's specification, plan, and tasks MUST be checked against
  this constitution before implementation, and unresolved conflicts MUST be resolved before work
  begins. Every pull request review verifies compliance; added complexity that violates a
  principle MUST be explicitly justified or removed.
- **Runtime guidance:** feature behaviour lives in the briefs under `Docs/features/` and in
  `Docs/design-decisions.md`; the documentation index is `Docs/README.md`. This constitution
  carries the non-negotiable engineering rules.

**Version**: 1.0.0 | **Ratified**: 2026-09-30 | **Last Amended**: 2026-09-30
