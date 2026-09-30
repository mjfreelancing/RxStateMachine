# 01 — Project Foundations

**Builds on:** nothing. **Prefix:** `FND`.

## Feature input (paste into `/speckit-specify`)

Establish the RxStateMachine project so every later feature is built and verified the same way: a
single multi-targeted NuGet package, an automated build-and-test pipeline on GitHub Actions that a
person with no prior GitHub Actions knowledge can read and trust, quality gates (formatting,
analyzers, coverage, documentation completeness, public-API tracking, vulnerability scanning), a
benchmark harness, a documentation site, and a place for runnable samples. No state machine behaviour
is delivered here; the goal is a green, empty-but-complete pipeline ("walking skeleton").

## Requirements

**Package and targets**
- FND-01 The repository MUST produce one NuGet package named `RxStateMachine`, MIT licensed, with
  README, license expression, repository URL, and icon in its metadata.
- FND-02 The package MUST target `netstandard2.0`, `net10.0`, and `net11.0`, and MUST NOT target
  `net8.0`. Public behaviour MUST be identical on every target.
- FND-03 The only runtime dependency MUST be `System.Reactive` 6.x (plus the BCL and unavoidable
  compatibility backports).
- FND-04 Multi-targeting differences MUST be confined to the `IsExternalInit` and
  `RequiredMemberAttribute` shims and narrow, commented `#if` gates.
- FND-05 `TimeProvider` MUST NOT appear in the public API; cosmetic use is gated behind
  `#if NET8_0_OR_GREATER` with a `DateTimeOffset.UtcNow` fallback for `netstandard2.0`.
- FND-06 Adding a runtime dependency or a target framework MUST require a recorded justification.

**Build and quality gates**
- FND-07 Nullable reference types MUST be enabled and warnings treated as errors in all projects.
- FND-08 A committed `.editorconfig`, repository-wide build properties, and central package version
  management MUST define style, analyzer level, and versions once.
- FND-09 Formatting MUST be verified in CI; a violation fails the build.
- FND-10 The public/`internal` boundary MUST be explicit and the public API surface MUST be tracked in
  a committed baseline so any addition or removal is visible in review.
- FND-11 Every public member MUST have XML documentation; a missing comment is a build error.
- FND-12 The package version MUST come from a single source; builds MUST be deterministic and include
  source-link information and symbol packages.
- FND-13 Package validation against the previous release MUST run for stable releases.
- FND-14 Dependencies MUST be scanned for known vulnerabilities on every change and on a schedule.
- FND-15 A `SECURITY.md`, a changelog (Keep a Changelog format), and contribution guidance MUST exist.

**Tests, benchmarks, docs, samples**
- FND-16 A unit-test project MUST exist and run against every target framework; property-based
  (FsCheck) and virtual-time (`TestScheduler`) support MUST be available to tests.
- FND-17 Line coverage of the core engine MUST be collected in CI, with an 80% floor that fails the
  build when not met.
- FND-18 A benchmark project MUST be runnable locally and in CI, with a stored baseline and a
  documented relative regression threshold.
- FND-19 An API reference site MUST be generated from XML documentation and built in CI.
- FND-20 A samples area MUST exist; every sample MUST be built and run in CI.

**Continuous integration (GitHub Actions)**
- FND-21 On every proposed change CI MUST restore, build every target with zero warnings, run all
  tests on every target, check formatting, run analyzers, build and run samples, pack, validate the
  package, and scan dependencies. A red build MUST block merging.
- FND-22 Publishing MUST be a separate, explicit, CI-only step that is safe to re-run without
  creating a duplicate release, and MUST NOT be possible from a developer machine.
- FND-23 Every workflow file MUST begin with a plain-language comment explaining what it does, when it
  runs, and what it needs; every step MUST have a readable name and a one-line comment giving its
  reason.
- FND-24 A beginner guide MUST explain, assuming no prior knowledge, how to find a workflow run, read
  pass/fail, open a failing step's log, and re-run a job, plus a glossary (workflow, job, step,
  runner, trigger, secret, artefact).
- FND-25 Secrets MUST live only in the platform secret store, never in files.

## Edge cases and failure modes
- FND-E1 A first run on an empty library (no features yet) MUST be green.
- FND-E2 A pre-release version tag and a stable tag MUST both be handled; re-publishing the same
  version MUST not fail destructively or create duplicates.
- FND-E3 Benchmark results vary run to run; the regression threshold MUST allow for noise and say how.
- FND-E4 A change from a contributor's fork MUST not require secrets to build and test.
- FND-E5 A target framework that is unavailable on the CI machine MUST fail with a clear message.
- FND-E6 A dependency with a newly disclosed vulnerability fails the scheduled scan even when no code
  changed.
- FND-E7 A new public member without documentation, or with an unrecorded API change, fails the build.

## Use-case coverage
Enables all samples ([use-cases.md](../use-cases.md)); delivers none.

## Design decisions relied on
DD-22 (how targets are measured).

## Open questions (for `/speckit-clarify`)
- Which documentation generator and versioning tool to use (planning decision).
- Exact relative regression threshold and coverage tool.
- Whether to use trunk-based pull requests or long-lived branches (affects CI triggers).

## Out of scope
Any state machine behaviour; registering the package on a public feed before the final deliverable
(decided at release time).
