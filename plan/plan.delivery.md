# Repo Edu Tauri -> Electron Migration Plan: Delivery

This document contains the phased execution plan and validation strategy
referenced by [plan.md](./plan.md).

Implementation must be tracked in
[plan.implementation-checklist.md](./plan.implementation-checklist.md). This
document defines the phase structure and exit criteria; the checklist breaks
those phases into resumable execution tasks and is the canonical place to mark
completed work during the rewrite.

## Delivery Phases

### Phase 1: New Monorepo Bootstrap in `../repo-edu`

- create the new workspace skeleton
- set up TypeScript, pnpm, linting, testing, and build tooling
- choose the package compilation and resolution strategy (`tsc` project
  references, package `exports`, and any conditional entrypoints)
- create `apps/desktop`, `apps/cli`, `apps/docs`
- create the shared packages listed above
- define `host-browser-mock` as the single reusable home for browser-safe
  runtime mocks plus LMS/Git integration mocks used by docs and browser-side
  tests
- derive a lean retained-contract inventory for major user-visible desktop, CLI,
  and docs-demo workflows from the current codebase, tests, and existing
  user-facing surfaces; this may be AI-generated, should be organized by major
  feature area rather than exhaustively, and must record confidence per area
- derive an area-level inventory of the existing Tauri TS and Rust tests, and
  define the test triage policy for what will be kept, rewritten, removed, or
  superseded in the new project; this is a major-feature-area decision log, not
  a file-by-file ledger
- treat `Roster` and `Group sets & Assignments` as higher-confidence inventory
  inputs because current behavior is meaningfully test-backed, and treat
  `Operations` as lower-confidence because its current behavior is largely
  untested and therefore only code-derived until explicitly reviewed
- build a minimal Electron shell spike and measure package size plus cold-start
  budget before feature work begins
- build a minimal browser-safe docs smoke harness early that mounts
  `packages/app` against mocks and exercises at least one shared workflow path,
  so browser-bundle safety is validated before feature migration
- prove that the `electron-trpc` router can expose a use-case as a subscription
  with typed progress/output streaming and that Node-routed HTTP solves at least
  one real CORS-constrained provider flow

Exit criteria:

- the new repo builds as an empty but coherent workspace
- package resolution behaves consistently across desktop, CLI, tests, and docs
- a lean retained-contract inventory exists for the major user-visible workflows
  that need automated coverage before phase 2 begins, with explicit confidence
  annotations by feature area; `Operations` entries are marked as code-derived
  and unverified where test backing is absent
- the test migration policy and initial area-level test triage inventory exist
  before phase 2 begins and are required phase-1 exit artifacts
- package size and startup budgets are measured with a minimal desktop shell
- the early browser-safe docs smoke harness proves the shared package boundaries
  can build and run without Node or Electron leakage
- the tRPC router and IPC link are validated with at least one end-to-end
  use-case spike
- the Node-routed HTTP bridge is validated early instead of being deferred to
  feature phases

### Phase 2: Domain and Port Foundations

- define the canonical domain types
- define the workflow invocation contract in `packages/application-contract`,
  including the canonical workflow definition map
- define the cross-shell user-file boundary: serializable `UserFileRef` /
  `UserSaveTargetRef` workflow DTOs, renderer-safe file-picker return types, and
  a `UserFilePort` that resolves those refs inside `packages/application`
- define `packages/renderer-host-contract` and `packages/host-runtime-contract`
- define `packages/integrations-lms-contract` and
  `packages/integrations-git-contract`
- define progress/output/event/error types in use-case signatures (tRPC infers
  the IPC contract from these)
- define the initial shared `AppError` discriminated-union taxonomy, including
  stable top-level variants, minimum per-variant fields, and ownership rules for
  which layer is allowed to create or normalize each variant
- define the shared workflow call options shape for progress, output, and
  cancellation, and require all long-running ports/use-cases to accept it, and
  classify each long-running workflow by declared progress granularity (`none`,
  `milestone`, `granular`) plus cancellation guarantee (`non-cancellable`,
  `best-effort`, `cooperative`)
- split renderer-safe direct host capabilities from application-side runtime
  ports so browser code cannot type-reach filesystem, subprocess, or network
  ports
- define persistence file formats for the new app
- define runtime validation only for boundary inputs (settings/profile files,
  imports, bridge payloads)
- use the replacement matrix to establish concrete TS boundaries where an
  up-front replacement decision already exists, and add or refine replacement
  classifications only when a missing decision would materially affect package
  boundaries or implementation strategy
- add continuous browser-bundle guardrails for packages that must stay usable in
  docs (`packages/domain`, `packages/application-contract`,
  `packages/integrations-lms-contract`, `packages/integrations-git-contract`,
  `packages/application`, and `packages/app`)
- map each retained user-visible contract in that lean inventory that needs
  stable automated coverage to planned shared workflow ids and delivery targets
  before phase 3
- perform a code-derived scope review for the low-confidence repository
  workflows (`repo.create`, `repo.clone`, `repo.delete`) and record which
  current behaviors are retained contracts versus intentional redesign before
  those workflows are implemented
- for lower-confidence areas such as `Operations`, promote only clearly
  intentional and user-visible behaviors into the retained-contract inventory;
  record ambiguous code-derived behavior as requiring explicit review instead of
  treating it as a preserved contract by default
- map each major legacy test area to a migration destination in the new
  architecture (domain, application, host adapter, Electron E2E, CLI, or docs)

Exit criteria:

- all current schema-defined domain shapes needed by app and CLI have
  hand-authored TS equivalents
- import/export workflows share one explicit cross-shell user-file boundary: raw
  filesystem paths and browser `File` objects do not leak into shared workflow
  signatures
- every defined runtime port and renderer host capability has at least one
  concrete consumer
- the shared `AppError` taxonomy and transport-normalization rules are defined
  before feature migration so UI, adapters, and use-cases share one failure
  contract
- every retained user-visible contract in scope that needs stable automated
  coverage is mapped to a planned shared workflow, plus the invoking delivery
  surfaces (desktop, CLI, docs) where applicable
- the low-confidence repository workflows have an explicit code-derived scope
  decision that freezes retained semantics versus redesign before phase 3
  implementation begins
- every major legacy test area has an explicit migration classification and
  target layer
- every major legacy area whose replacement strategy materially affects package
  boundaries or implementation approach has an explicit replacement decision
  before feature migration proceeds
- shared types and ports are stable enough that app/CLI can build on them
  without regeneration
- browser-safe package guardrails are active before shared feature migration

### Phase 3: Application Layer Rewrite

#### Phase 3A: Infrastructure Replacements

- implement boundary validation with `zod`
- implement CSV and Excel adapters with `papaparse` and `xlsx`
- implement GitHub and GitLab provider adapters with provider SDKs behind
  app-owned ports
- implement the CLI command parsing layer with `commander`

Exit criteria:

- low-value infrastructure concerns are removed from the custom code surface
  before domain migration expands

#### Phase 3B: Thin Host and Remote Adapters

- implement Gitea adapter over host-side `fetch`
- implement Canvas and Moodle adapters over host-side `fetch`
- implement Git subprocess execution over a process port using a thin explicit
  `child_process.spawn` wrapper over the system Git CLI, not `simple-git`
- define infrastructure-level batch specs used by clone/delete and other
  filesystem-heavy repository operations
- implement `ProcessPort` as an explicit subprocess execution port, while
  keeping validated filesystem plans in `FileSystemPort` without embedding
  business rules in the host
- shape the Git execution layer so it can support future in-app
  `gitinspectorgui` workflows with broader read/query coverage, including
  follow-aware commands such as `git log --follow` and `git blame --follow`

Exit criteria:

- host and remote integrations are available through explicit ports without
  leaking protocol details into use-cases
- the Git execution boundary is fixed and non-ambiguous: one explicit Git CLI
  adapter path exists, and it is extensible enough for future inspection
  workflows without introducing a second local Git stack

#### Phase 3C: Shared Domain Logic

- port pure roster, group-set, validation, naming, glob, diff, and repository
  planning logic into shared TypeScript modules
- keep these modules independent from provider SDKs and file-format libraries
  except through explicit boundary data shapes
- migrate and improve invariant tests with each migrated domain module

Exit criteria:

- product rules are isolated from infrastructure libraries and remain testable
  without host adapters

#### Phase 3D: LMS Orchestration

- port LMS orchestration into `application`
- add retained-contract tests where external semantics must stay stable, and add
  adapter tests with each LMS workflow

#### Phase 3E: Git Workflows and Persistence

- port repository planning orchestration into `application`
- rebuild settings/profile persistence services
- add retained-contract and boundary validation tests with each persistence, Git
  workflow, and host-executed repository workflow

Exit criteria:

- all former Rust behaviors exist in TS packages with tests
- retained-contract coverage and migrated invariant tests are added alongside
  each migrated concern, not deferred
- shared packages that must run in docs remain browser-bundle-safe as each
  migrated concern lands

### Phase 4: React App Refactor

- port current UI into `packages/app`
- replace backend command invocations with injected `WorkflowClient` calls
- wire the renderer against `packages/renderer-host-contract` and
  `packages/application-contract`
- validate state-management behavior for undo/redo, async checkpoints, and
  optimistic updates

Exit criteria:

- the app runs in a browser-safe environment against mocks
- state transitions remain coherent under async use-cases

### Phase 5: Electron Shell

- implement the tRPC router in Electron main with procedures for all
  `packages/application` use-cases
- implement one explicit main-side workflow registry typed as an exhaustive map
  from every workflow key to its router procedure
- wire the `electron-trpc` IPC link in preload
- implement the desktop `WorkflowClient` adapter over the inferred tRPC client,
  with its own exhaustive renderer-side binding derived from the same shared
  workflow definition map used by the main-side registry
- inject that adapter into `packages/app`
- keep packaging/build entrypoints isolated to the Electron shell so production
  distribution can be added later without refactoring shared packages
- do not require production release/distribution work in this phase: no
  committed artifact matrix, code signing/notarization path, updater provider,
  release channel model, or delta-update policy decision yet

Exit criteria:

- the desktop app runs with the real Node host adapter via the tRPC router
- all use-cases are callable from the renderer through `WorkflowClient` backed
  by tRPC with typed progress/output streaming
- registration completeness is compile-time enforced on both sides: missing or
  extra main-side or renderer-side desktop workflow bindings do not compile
- release/distribution concerns remain shell-local extension points: adding
  packaging, signing, notarization, or updater delivery later does not require
  changing shared application package boundaries

### Phase 6: CLI

- implement the CLI command tree
- map each command to shared application use-cases
- add retained-contract and golden/output tests for command behavior

Exit criteria:

- the new CLI covers the full current CLI surface

### Phase 7: Docs Demo

- port the Astro docs site
- wire the standalone demo against `host-browser-mock`
- provide a local browser-safe `WorkflowClient` adapter for docs
- ensure the mock simulation remains behaviorally aligned with the app
- keep docs retained-contract checks aligned with the same shared workflows used
  by app and CLI

Exit criteria:

- docs build and the standalone demo works without Electron

## Testing Strategy

The new project should be validated at the architecture boundaries, not only
through UI smoke tests.

Required test layers:

- unit tests for pure domain logic
- migrated invariant tests from the legacy codebase that still encode intended
  behavior
- contract tests for host ports and adapter implementations
- integration tests for LMS/Git adapter behavior
- React tests for major workflows
- end-to-end Electron tests for core desktop flows
- CLI golden/output tests for retained command semantics
- workflow output-channel tests where long-running workflows emit structured
  diagnostics distinct from progress
- docs demo smoke tests to ensure the browser-safe simulation still mounts
- boundary validation tests for persisted files and other untrusted inputs
- targeted retained-contract tests for retained external semantics only

Test triage and migrated tests should be handled as each module or workflow is
rebuilt. Do not defer automated test migration or retained-contract coverage
until the end of the migration.

Maintain explicit documentation of how each retained user-visible contract is
verified so it maps to:

- shared workflow id
- desktop invocation path if applicable
- CLI workflow if applicable
- docs mock coverage if applicable

The migration is complete only when every retained user-visible contract has a
corresponding automated verification path, every major legacy test area has been
explicitly migrated, rewritten, removed, or superseded, and the rewrite is ready
for user-based acceptance testing as a final end-to-end phase.

Automated testing should run continuously from phase 2 through phase 7, with a
final audit only as a completion gate rather than as a separate implementation
phase. User-based acceptance testing should be postponed until the rewrite is
completely finished; partial user-based testing during the migration is
intentionally out of scope.
