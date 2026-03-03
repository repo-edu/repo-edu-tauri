# Repo Edu Tauri -> Electron Migration Plan: Delivery

This document contains the phased execution plan and validation strategy
referenced by [plan.md](./plan.md).

## Delivery Phases

### Phase 1: New Monorepo Bootstrap in `../repo-edu`

- create the new workspace skeleton
- set up TypeScript, pnpm, linting, testing, and build tooling
- choose the package compilation and resolution strategy (`tsc` project
  references, package `exports`, and any conditional entrypoints)
- create `apps/desktop`, `apps/cli`, `apps/docs`
- create the shared packages listed above
- inventory every current user-visible desktop, CLI, and docs-demo feature and
  create the canonical behavior-preservation checklist before feature work
  starts
- inventory the existing Tauri TS and Rust tests, and define the test triage
  policy for what will be kept, rewritten, removed, or superseded in the new
  project
- build a minimal Electron shell spike and measure package size plus cold-start
  budget before feature work begins
- build a minimal browser-safe docs smoke harness early that mounts
  `packages/app` against mocks and exercises at least one shared workflow path,
  so browser-bundle safety is validated before feature migration
- prove that the `electron-trpc` router can expose a use-case as a subscription
  with typed progress streaming and that Node-routed HTTP solves at least one
  real CORS-constrained provider flow

Exit criteria:

- the new repo builds as an empty but coherent workspace
- package resolution behaves consistently across desktop, CLI, tests, and docs
- the canonical behavior-preservation checklist exists before phase 2 completes
  and is required input for later phases
- the test migration policy and initial test triage inventory exist before phase
  2 completes
- package size and startup budgets are measured with a minimal desktop shell
- the early browser-safe docs smoke harness proves the shared package
  boundaries can build and run without Node or Electron leakage
- the tRPC router and IPC link are validated with at least one end-to-end
  use-case spike
- the Node-routed HTTP bridge is validated early instead of being deferred to
  feature phases

### Phase 2: Domain and Port Foundations

- define the canonical domain types
- define the workflow invocation contract in `packages/application-contract`,
  including the canonical workflow definition map
- define `packages/renderer-host-contract` and
  `packages/host-runtime-contract`
- define progress/event/error types in use-case signatures (tRPC infers the IPC
  contract from these)
- define the shared workflow call options shape for progress plus cancellation,
  and require all long-running ports/use-cases to accept it, and classify each
  long-running workflow by declared progress granularity (`none`, `milestone`,
  `granular`) plus cancellation guarantee (`non-cancellable`, `best-effort`,
  `cooperative`)
- split renderer-safe direct host capabilities from application-side runtime
  ports so browser code cannot type-reach filesystem, subprocess, or network
  ports
- define persistence file formats for the new app
- define runtime validation only for boundary inputs (settings/profile files,
  imports, bridge payloads)
- add continuous browser-bundle guardrails for packages that must stay usable in
  docs (`packages/domain`, `packages/application-contract`,
  browser-safe `packages/application` entrypoints, and `packages/app`)
- map every behavior-preservation checklist item to planned shared workflow ids
  and delivery targets before phase 3
- map each major legacy test area to a migration destination in the new
  architecture (domain, application, host adapter, Electron E2E, CLI, or docs)

Exit criteria:

- all current schema-defined domain shapes needed by app and CLI have
  hand-authored TS equivalents
- every defined runtime port and renderer host capability has at least one
  concrete consumer
- every behavior-preservation checklist item is mapped to a planned shared
  workflow, plus the invoking delivery surfaces (desktop, CLI, docs) where
  applicable
- every major legacy test area has an explicit migration classification and
  target layer
- shared types and ports are stable enough that app/CLI can build on them
  without regeneration
- browser-safe package guardrails are active before shared feature migration

### Phase 3: Application Layer Rewrite

#### Phase 3A: Domain Logic

- port pure roster, group-set, validation, naming, glob, and diff logic into
  `domain`
- migrate and improve invariant tests with each migrated domain module

#### Phase 3B: LMS Integrations

- port Canvas/Moodle integrations into `integrations-lms`
- port LMS orchestration into `application`
- add behavior-preservation checks where external semantics must stay stable, and
  add adapter tests with each LMS workflow

#### Phase 3C: Git Integrations

- port Git provider integrations into `integrations-git`
- port repository planning logic into `application`
- add behavior-preservation checks where external semantics must stay stable, and
  add adapter tests with each Git workflow

#### Phase 3D: Persistence and File Workflows

- rebuild settings/profile persistence services
- rebuild CSV and Excel import/export logic
- add behavior-preservation and boundary validation tests with each persistence
  and file workflow

#### Phase 3E: Host-Executed Repository Tasks

- define infrastructure-level batch specs used by clone/delete and other
  filesystem-heavy repository operations
- implement `TaskRunnerPort` over generic filesystem/process primitives without
  embedding business rules in the host
- add workflow and host-adapter tests with each host-executed repository
  workflow

Exit criteria:

- all former Rust behaviors exist in TS packages with tests
- retained behavior-preservation coverage and migrated invariant tests are added
  alongside each migrated concern, not deferred
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
- add packaging and desktop build scripts
- implement desktop updater integration

Exit criteria:

- the desktop app runs with the real Node host adapter via the tRPC router
- all use-cases are callable from the renderer through `WorkflowClient` backed
  by tRPC with typed progress streaming
- registration completeness is compile-time enforced on both sides: missing or
  extra main-side or renderer-side desktop workflow bindings do not compile
- desktop packaging and update paths are wired

### Phase 6: CLI

- implement the CLI command tree
- map each command to shared application use-cases
- add behavior-preservation and golden/output tests for command behavior

Exit criteria:

- the new CLI covers the full current CLI surface

### Phase 7: Docs Demo

- port the Astro docs site
- wire the standalone demo against `host-browser-mock`
- provide a local browser-safe `WorkflowClient` adapter for docs
- ensure the mock simulation remains behaviorally aligned with the app
- keep docs behavior-preservation checks aligned with the same shared workflows
  used by app and CLI

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
- docs demo smoke tests to ensure the browser-safe simulation still mounts
- boundary validation tests for persisted files and other untrusted inputs
- targeted behavior-preservation tests for retained external semantics only

Test triage and migrated tests should be handled as each module or workflow is
rebuilt. Do not defer test migration or behavior-preservation coverage until the
end of the migration.

Maintain the behavior-preservation checklist created in phase 1 so it maps each
retained user-visible contract to:

- shared workflow id
- desktop invocation path if applicable
- CLI workflow if applicable
- docs mock coverage if applicable

The migration is complete only when every retained user-visible contract has a
corresponding verified path, and every major legacy test area has been
explicitly migrated, rewritten, removed, or superseded.

Use the behavior-preservation checklist and test triage inventory continuously
from phase 2 through phase 7, with a final audit only as a completion gate
rather than as a separate implementation phase.
