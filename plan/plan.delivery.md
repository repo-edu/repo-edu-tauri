# Repo Edu Tauri -> Electron Migration Plan: Delivery

This document contains the phased execution plan and validation strategy referenced by
[plan.md](./plan.md).

## Delivery Phases

### Phase 1: New Monorepo Bootstrap in `../repo-edu`

- create the new workspace skeleton
- set up TypeScript, pnpm, linting, testing, and build tooling
- choose the package compilation and resolution strategy (`tsc` project references, package
  `exports`, and any conditional entrypoints)
- create `apps/desktop`, `apps/cli`, `apps/docs`
- create the shared packages listed above
- build a minimal Electron shell spike and measure package size plus cold-start budget before
  feature work begins
- prove that the `electron-trpc` router can expose a use-case as a subscription with typed progress
  streaming and that Node-routed HTTP solves at least one real CORS-constrained provider flow

Exit criteria:

- the new repo builds as an empty but coherent workspace
- package resolution behaves consistently across desktop, CLI, tests, and docs
- package size and startup budgets are measured with a minimal desktop shell
- the tRPC router and IPC link are validated with at least one end-to-end use-case spike
- the Node-routed HTTP bridge is validated early instead of being deferred to feature phases

### Phase 2: Domain and Port Foundations

- define the canonical domain types
- define the workflow invocation contract in `packages/application-contract`
- define all host contracts
- define progress/event/error types in use-case signatures (tRPC infers the IPC contract from
  these)
- define persistence file formats for the new app
- define runtime validation only for boundary inputs (settings/profile files, imports, bridge
  payloads)

Exit criteria:

- all current schema-defined domain shapes needed by app and CLI have hand-authored TS equivalents
- every defined host port has at least one concrete consumer
- shared types and ports are stable enough that app/CLI can build on them without regeneration

### Phase 3: Application Layer Rewrite

#### Phase 3A: Domain Logic

- port pure roster, group-set, validation, naming, glob, and diff logic into `domain`
- add parity tests with each migrated domain module

#### Phase 3B: LMS Integrations

- port Canvas/Moodle integrations into `integrations-lms`
- port LMS orchestration into `application`
- add parity and adapter tests with each LMS workflow

#### Phase 3C: Git Integrations

- port Git provider integrations into `integrations-git`
- port repository planning logic into `application`
- add parity and adapter tests with each Git workflow

#### Phase 3D: Persistence and File Workflows

- rebuild settings/profile persistence services
- rebuild CSV and Excel import/export logic
- add parity tests with each persistence and file workflow

#### Phase 3E: Host-Executed Repository Tasks

- define infrastructure-level batch specs used by clone/delete and other filesystem-heavy
  repository operations
- implement `TaskRunnerPort` over generic filesystem/process primitives without embedding business
  rules in the host
- add parity tests with each host-executed repository workflow

Exit criteria:

- all former Rust behaviors exist in TS packages with tests
- parity coverage is added alongside each migrated concern, not deferred

### Phase 4: React App Refactor

- port current UI into `packages/app`
- replace backend command invocations with injected `WorkflowClient` calls
- wire the renderer against host contracts and `packages/application-contract`
- validate state-management behavior for undo/redo, async checkpoints, and optimistic updates

Exit criteria:

- the app runs in a browser-safe environment against mocks
- state transitions remain coherent under async use-cases

### Phase 5: Electron Shell

- implement the tRPC router in Electron main with procedures for all `packages/application`
  use-cases
- wire the `electron-trpc` IPC link in preload
- implement the desktop `WorkflowClient` adapter over the inferred tRPC client
- inject that adapter into `packages/app`
- add packaging and desktop build scripts
- implement desktop updater integration

Exit criteria:

- the desktop app runs with the real Node host adapter via the tRPC router
- all use-cases are callable from the renderer through `WorkflowClient` backed by tRPC with typed
  progress streaming
- desktop packaging and update paths are wired

### Phase 6: CLI

- implement the CLI command tree
- map each command to shared application use-cases
- add parity tests for command behavior

Exit criteria:

- the new CLI covers the full current CLI surface

### Phase 7: Docs Demo

- port the Astro docs site
- wire the standalone demo against `host-browser-mock`
- provide a local browser-safe `WorkflowClient` adapter for docs
- ensure the mock simulation remains behaviorally aligned with the app
- keep docs parity checks aligned with the same shared workflows used by app and CLI

Exit criteria:

- docs build and the standalone demo works without Electron

## Testing Strategy

The new project should be validated at the architecture boundaries, not only through UI smoke
tests.

Required test layers:

- unit tests for pure domain logic
- contract tests for host ports and adapter implementations
- integration tests for LMS/Git adapter behavior
- React tests for major workflows
- end-to-end Electron tests for core desktop flows
- CLI golden/output tests for command parity
- docs demo smoke tests to ensure the browser-safe simulation still mounts
- boundary validation tests for persisted files and other untrusted inputs

Parity tests should be written as each module or workflow is rebuilt. Do not defer parity coverage
until the end of the migration.

Add a parity checklist that maps each existing user-visible feature to:

- desktop workflow
- CLI workflow if applicable
- docs mock coverage if applicable

The migration is complete only when every current feature has a corresponding verified path.

Use the parity checklist continuously during phases 3 through 7, with a final audit only as a
completion gate rather than as a separate implementation phase.
