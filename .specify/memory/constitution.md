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
- Schedulers MUST be injected (`IScheduler`, default `Scheduler.Default` for the machine's own timed
  work), never ambient. There MUST be no static mutable state and no hidden dependence on
  wall-clock time, culture, or thread state.
- Every timestamp the machine produces MUST come from the injected scheduler's clock, so timestamps
  are deterministic under virtual time. `TimeProvider`, `DateTime.Now`, and `DateTimeOffset.UtcNow`
  MUST NOT be used to produce them, and `TimeProvider` MUST NOT appear in the public API.
- Immediate mode MUST be single-threaded and reentrancy-protected. Concurrent `Fire` is supported
  only in the explicit queued mode, which MUST NEVER be enabled implicitly.
- Locks MUST NOT be held across an `await` or while invoking consumer-supplied code (guards,
  actions, observer callbacks). Lock objects MUST be private (never `lock(this)` or a public
  type). Shared state MUST be immutable, confined to one thread, or guarded by a documented
  mechanism.
- Every timed behaviour MUST be reproducible under Rx `TestScheduler` virtual time.

Rationale: blocking causes deadlocks and starvation in UI, server, and IoT hosts, and injectable
schedulers are what make deterministic virtual-time testing possible.

### V. Test-First
- Tests SHOULD be written before implementation (red → green → refactor). A bug fix MUST begin with
  a failing regression test, and core engine behaviour (transition semantics, reentrancy,
  concurrency) MUST have its tests written before the implementation.
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
  behaviour tests. The same suite MUST run on every target runtime. A target that is not a runtime
  (`netstandard2.0`) MUST be exercised by running that same suite on a runtime that resolves its
  asset, so the compatibility code path is executed rather than merely compiled.
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
- A BenchmarkDotNet project MUST exist and be runnable locally. Absolute timings are not
  comparable between machines (a developer machine and a CI runner differ), so latency is judged
  by a before/after comparison run on the same machine at implementation time, not against a
  stored baseline or a CI threshold.
- A feature that touches the hot path (transition engine, guard evaluation, observable surface)
  MUST record a before/after benchmark comparison in its plan or tasks; any other feature
  states "no hot-path impact". An unexplained regression in that comparison MUST be justified or
  reworked before merge.
- The zero-allocation guarantee is machine-independent and MUST be enforced on every change by a
  deterministic test, on every target runtime whose BCL exposes a per-thread allocation measurement;
  where a runtime does not, the limitation MUST be stated rather than hidden. Optimisations that
  reduce clarity MUST be justified by benchmark results.
- How each performance target is measured is defined in `Docs/design-decisions.md` (DD-22).

Rationale: the machine is embedded in UI handlers, message pumps, and processing loops, where
avoidable allocation and per-transition setup show up as throughput loss and GC pressure.

### VII. Code Quality & Maintainability
- Code style MUST be defined in a committed `.editorconfig` and enforced in CI
  (`dotnet format --verify-no-changes` or equivalent); style is not debated in review.
- Methods and types MUST have a single, clear responsibility. A suppression MUST carry a written
  justification.
- Composition over inheritance, small focused abstractions, and immutability by default MUST be
  preferred. An abstraction, option, or extension point MUST NOT be added for a hypothetical need.
- Exceptions MUST be specific (no throwing or catching bare `Exception` except at a documented
  policy boundary), MUST NOT be swallowed silently, and messages MUST say what happened, which
  state and trigger were involved, and how to fix it.
- Compiler and analyzer warnings MUST be errors. Suppression MUST be as narrow as possible
  (scoped `#pragma` or attribute) with a justification, never global.
- Comments MUST explain why, not what. Dead code and commented-out code MUST NOT be merged.
  Deferred work MUST be tracked within the feature structure (a task in the current feature's
  tasks, an addition to the brief of the later feature it belongs to, or a new feature) and not in
  an external issue tracker. A `TODO` comment MUST point at such an item; an untracked `TODO` MUST
  NOT be merged.
- Library code MUST NOT use reflection, `dynamic`, or static mutable state.
- Analyzers MUST be enabled at the latest recommended level or stricter, and centralised in
  repository-wide build properties.

Rationale: a library stays trustworthy only if quality rules are mechanical and consistent rather
than dependent on who reviews a change.

### VIII. Documentation, Samples & Simplicity
- 100% of public members MUST have XML documentation (a missing-doc warning is an error). Where
  relevant, the documentation SHOULD also state exceptions thrown and the threading and hot/cold
  behaviour of streams. The XML
  documentation MUST be updated in the same change that alters the public surface. A generated
  documentation site or user guides are optional and are not a gate.
- Each use case in `Docs/use-cases.md` MUST end up with a runnable sample. The set of use cases and
  samples is open-ended and grows as use cases are discovered; there is no fixed count or domain
  list.
- The simplest design that satisfies the requirement MUST be preferred. Later features MUST NOT
  leak partial designs into earlier features' public API.
- The non-goals in `Docs/vision.md` MUST NOT be built into the core without a constitutional
  amendment. Saga-style usage is supported, but compensation, distributed coordination, and
  long-running workflow scheduling belong in a separate companion library.
- The repository `README.md` MUST describe the product only. It MAY link to the consumer-facing
  documents in `Docs/` (vision, observable model, use cases, glossary, references, diagrams) and
  MUST NOT link to the internal planning documents: `Docs/roadmap.md`, `Docs/features/`,
  `Docs/design-decisions.md`, `Docs/engineering-standards.md`, `Docs/api-sketch.md`, and
  `Docs/saga-exploration.md`. Its code examples SHOULD be taken from the samples (which build in CI)
  so they stay correct.

Rationale: low-friction adoption depends on excellent documentation and a focused scope; scope
creep is a named project risk.

## Platform, Packaging & Security Constraints

- **Target frameworks:** `netstandard2.0`, `net10.0`, `net11.0`. Earlier `net*` targets are not
  carried: .NET 8 and .NET 9 both reach end of support in November 2026. A declared target served by
  a preview SDK MUST build and be tested like any other; there is no preview carve-out.
  Multi-targeting differences MUST be confined to language-feature shims (`IsExternalInit`,
  `RequiredMemberAttribute`, both declared `internal` so they cannot collide with a consumer's own)
  and narrow `#if` gates; public behaviour MUST be identical on every target.
- **Dependencies:** the only runtime dependency MUST be `System.Reactive` (6.x), plus the BCL and
  unavoidable compatibility backports. No DI, UI, messaging, ORM, serializer, or
  logging-framework dependencies; integrations are documentation examples only.
- **Package:** a single NuGet package named `RxStateMachine`, MIT licensed, with README, license
  expression, repository URL, and icon in the package metadata.
- **Reproducibility:** builds MUST be deterministic, with SourceLink and symbol packages. Package
  versions come from one source. Package validation MUST run on every pack: framework-compatibility
  validation, so the public surface of each target asset is checked against the others, and baseline
  validation against the previous release for stable releases. Packages MUST be published only from
  CI, never from a developer machine.
  Central package version management and repository-wide build properties MUST be used.
- **Security:** no dynamic code loading and no deserialization of untrusted input in the core.
  Dependencies MUST be scanned for known vulnerabilities in CI on every change and on a schedule
  (a scheduled workflow whose failure is visible to the maintainer satisfies this). A `SECURITY.md`
  MUST describe how to report vulnerabilities. Dependencies MUST be pinned centrally. Secrets MUST
  never be stored in the repository.

## Continuous Integration & Development Workflow

- **Platform:** CI runs on GitHub Actions. Each workflow file MUST begin with a plain-language
  comment saying what it does, when it runs, and what it needs, and each step MUST have a
  human-readable name.
- **Gates:** on every push to `main` and every pull request, CI MUST restore, build every target
  framework with zero warnings, run the full test suite on every target runtime, check formatting,
  run analyzers, build and run the samples, pack, and validate the package. A red build MUST block a release, and MUST
  block merge when a pull request is used.
- **Publishing:** MUST be a separate, explicit, CI-only step that is safe to re-run without
  creating a duplicate release. CI MUST NOT contain secrets in plain text; secrets use the
  platform's secret store.
- **Sequencing:** features are built in the order of `Docs/roadmap.md`. A later feature MUST NOT
  start before its prerequisites are complete.
- **Changes:** a change SHOULD be small and focused on one concern. Pull requests are optional.
  Every change MUST meet the Definition of Done: tests passing, analyzers clean, public-API
  baseline updated, XML docs updated, deferred work tracked within the features, and no
  unjustified suppressions. Release notes are generated by the platform; no changelog file is
  maintained.
- **Review:** the maintainer (optionally with an agent-assisted review) MUST check each change
  against this constitution; a violation requires a fix or a justified, recorded exception.
- **Commits and releases:** commits MUST be focused and messages MUST describe the why; the
  release process MUST tag the commit that produced the published package.

## Governance

This constitution supersedes other project practices. Where any planning document conflicts with
it, the constitution wins until amended.

- **Amendments:** made with a written rationale and recorded as a versioned change to this
  document. The version line below MUST be updated on every amendment.
- **Versioning policy:** MAJOR for backward-incompatible governance changes or principle
  removals/redefinitions; MINOR for a new principle or materially expanded guidance; PATCH for
  clarifications, wording fixes, and other non-semantic refinements.
- **Compliance review:** each feature's specification, plan, and tasks MUST be checked against
  this constitution before implementation, and unresolved conflicts MUST be resolved before work
  begins. Each change is checked for compliance by the maintainer; added complexity that violates a
  principle MUST be explicitly justified or removed.
- **Runtime guidance:** feature behaviour lives in the briefs under `Docs/features/` and in
  `Docs/design-decisions.md`; the documentation index is `Docs/README.md`. This constitution
  carries the non-negotiable engineering rules.

**Version**: 1.0.0 | **Ratified**: 2026-09-30 | **Last Amended**: 2026-09-30
