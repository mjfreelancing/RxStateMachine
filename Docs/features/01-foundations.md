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
  `#if NET8_0_OR_GREATER` (true on `net10.0` and `net11.0`; `net8.0` is still not a target) with a
  `DateTimeOffset.UtcNow` fallback for `netstandard2.0`.
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
- FND-13 Package validation against the previous release MUST run for stable releases. When no
  previous release exists, validation is skipped and the skip and its reason are logged (FND-E9).
- FND-14 Dependencies MUST be scanned for known vulnerabilities on every change and on a schedule (at
  least weekly). A failed scheduled scan MUST notify the maintainers (for example by opening an issue
  or failing a visible workflow).
- FND-15 A `SECURITY.md`, a changelog (Keep a Changelog format), and contribution guidance MUST exist.

**Tests, benchmarks, docs, samples**
- FND-16 A unit-test project MUST exist and run against every target framework; property-based
  testing and virtual-time (Rx `TestScheduler`) support MUST be available to tests.
- FND-17 Line coverage of the core engine MUST be collected in CI, with an 80% floor that fails the
  build when not met. The floor is enforced only once the core assembly contains executable code
  (FND-E10).
- FND-18 A benchmark project MUST be runnable locally and in CI, with a stored baseline and a
  documented relative regression threshold. Allocation regressions (DD-22: zero bytes per
  steady-state transition) MUST fail the build; latency regressions beyond the threshold MUST fail
  only when reproduced on a re-run, because shared CI runners are noisy (FND-E3).
- FND-19 An API reference site MUST be generated from XML documentation and built in CI.
- FND-20 A samples area MUST exist; every sample MUST be built and run in CI. An empty samples area,
  or a single placeholder sample, MUST pass (FND-E10).

**Continuous integration (GitHub Actions)**
- FND-21 On every proposed change CI MUST restore, build every target with zero warnings, run all
  tests on every target, check formatting, run analyzers, build and run samples, pack, validate the
  package, and scan dependencies. A red build MUST block merging.
- FND-22 Publishing MUST be a separate, explicit, CI-only step that is safe to re-run without
  creating a duplicate release, and MUST NOT be possible from a developer machine. It MUST be
  triggered only by a version tag and gated by a protected environment requiring maintainer approval.
  In this feature the publish job is built and exercised against a dry run or local feed only;
  registering on a public feed is out of scope.
- FND-23 Every workflow file MUST begin with a plain-language comment explaining what it does, when it
  runs, and what it needs; every step MUST have a readable name and a one-line comment giving its
  reason.
- FND-24 A beginner guide MUST explain, assuming no prior knowledge, how to find a workflow run, read
  pass/fail, open a failing step's log, and re-run a job, plus a glossary (workflow, job, step,
  runner, trigger, secret, artefact).
- FND-25 Secrets MUST live only in the platform secret store, never in files.
- FND-26 Third-party actions MUST be pinned to a full commit SHA or an exact version, and every
  workflow MUST declare least-privilege `permissions`. The beginner guide (FND-24) MUST explain why.
- FND-27 Workflows MUST cancel superseded runs on the same pull request and MUST cache package
  restores, without the cache ever hiding a failure.
- FND-28 Dependency updates MUST be proposed automatically (for both NuGet packages and GitHub
  Actions) so FND-06 justifications and FND-14 scans have a steady input.
- FND-29 The repository MUST document, as a manual checklist, the settings that cannot live in files
  and that FND-21 relies on: required status checks and branch protection on `main`, and the
  protected publishing environment from FND-22.
- FND-30 Mutation testing (engineering-standards.md) is deferred: the foundations MUST leave room for a
  scheduled mutation-testing job but MUST NOT require one yet.

**Branching**
- FND-31 The repository uses trunk-based development: short-lived branches merge to `main` by pull
  request. CI runs on pull requests and on pushes to `main`; publishing runs only on version tags.

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
- FND-E8 `net11.0` may be a preview SDK. CI MUST install the required SDKs explicitly; if a required
  SDK cannot be installed the build fails with a clear message (FND-E5). Whether a preview-only
  failure blocks merging MUST be decided and documented (proposed: it blocks, since all targets are
  required).
- FND-E9 The first release has no previous version to validate against; package validation is skipped
  with a logged reason rather than failing.
- FND-E10 With no executable core code and no samples yet, the coverage gate (FND-17) and the sample
  run (FND-20) pass vacuously with a logged note, not a divide-by-zero or a failure.

## Use-case coverage
Enables all samples ([use-cases.md](../use-cases.md)); delivers none.

## Design decisions relied on
DD-22 (how targets are measured).

## Success criteria
- SC-1 On a fresh clone, one documented command builds and tests every target locally.
- SC-2 A newcomer using only the beginner guide (FND-24) can locate a failing step's log and re-run it.
- SC-3 Every edge case FND-E1 to FND-E10 is covered by a CI job, a check, or a documented manual step.
- SC-4 The first CI run on the empty library is green on every target.
- SC-5 Introducing an undocumented public member, an unrecorded API change, a formatting violation, or
  a vulnerable dependency each independently turns CI red.

## Proposed defaults (tool choices to confirm in `/speckit-clarify` or `/speckit-plan`)
Tool names are kept out of the requirements above; these are the intended picks.
- **Documentation site:** DocFX, published to GitHub Pages. **Versioning:** MinVer, deriving the
  version from git tags (alternative: Nerdbank.GitVersioning if a committed version file is wanted).
- **Coverage:** Coverlet (Cobertura output) with ReportGenerator for the summary, scoped to the core
  assembly, 80% line floor.
- **Benchmarks:** BenchmarkDotNet. Allocation is the hard gate. Latency threshold: 10% relative
  regression on mean time, failing only if reproduced on re-run; compare against a baseline produced
  on the same runner class; run latency comparison on a schedule or label rather than every pull
  request.
- **Property-based testing:** FsCheck. **Dependency updates:** Dependabot.
- **Branching:** trunk-based (FND-31). Revisit only if old major versions must be maintained.

## Open questions (for `/speckit-clarify`)
- Confirm the proposed defaults above.
- Cadence and notification channel for the scheduled vulnerability scan (proposed: weekly, issue opened
  on failure).
- Whether a failure on a preview SDK target blocks merging (FND-E8; proposed: yes).
- Which feed receives pre-release versus stable packages when publishing is eventually enabled.

## Out of scope
Any state machine behaviour; registering the package on a public feed before the final deliverable
(decided at release time).
