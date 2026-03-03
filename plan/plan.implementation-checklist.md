# Repo Edu Tauri -> Electron Migration Plan: Implementation Checklist

This document is the canonical execution ledger for implementing the migration
defined in [plan.md](./plan.md).

Its purpose is operational, not architectural:

- break the migration into resumable, implementation-sized tasks
- make task ordering and dependencies explicit
- let a fresh AI continue work safely after any interruption
- record what must be updated before a task can be considered complete

If this document and another plan file disagree about execution order or task
granularity, this file controls implementation sequencing while the other plan
files still control architecture and scope.

## Usage Rules

1. Treat each checkbox item as an atomic execution unit. Do not mark it done
   until the code, tests, and documentation required by that item are complete.
2. When work is interrupted, the next AI should resume from the first unchecked
   item whose prerequisites are satisfied.
3. If a task is partially complete, do not check it off. Instead, add a short
   note under the item describing what remains.
4. When completing an item, update this file in the same change set as the
   implementation whenever possible.
5. Do not skip ahead across unmet dependencies just because a later task looks
   easier. Reorder only if the architecture changes, and then update this file
   first.
6. If implementation reveals that a task is too large for one uninterrupted AI
   session, split it into smaller checkbox items before continuing.

## Resume Protocol

Any AI resuming this migration in a fresh context should read these files in
this order before making changes:

1. [plan.md](./plan.md)
2. [plan.delivery.md](./plan.delivery.md)
3. This file
4. Any phase-specific supporting inventory directly relevant to the next
   unchecked task

Before starting implementation, the AI should:

1. Find the first unchecked item below whose prerequisites are complete.
2. Verify that any files listed in that item already exist or still need to be
   created.
3. Check whether earlier completed items have obvious drift that must be fixed
   before proceeding.
4. Continue only after updating this checklist if the planned execution order
   has changed.

## Completion Discipline

Every completed task should leave behind:

- the required code or document artifacts for that task
- tests or explicit test placeholders if the task is planning-only
- an updated checkbox in this file
- any concise progress note needed for the next AI to resume safely

Recommended progress note format for partially completed tasks:

```md
Note: <what was finished>. Remaining: <what is still blocked or incomplete>.
```

## Phase 1: New Monorepo Bootstrap in `../repo-edu`

### 1.1 Workspace Skeleton

- [ ] Create the `../repo-edu` repository root with `package.json`,
      `pnpm-workspace.yaml`, `tsconfig.base.json`, and the base workspace
      directory layout from [plan.architecture.md](./plan.architecture.md).
- [ ] Create empty app shells for `apps/desktop`, `apps/cli`, and `apps/docs`
      with build entrypoints that compile successfully.
- [ ] Create empty package shells for `packages/ui`, `packages/app`,
      `packages/domain`, `packages/application-contract`,
      `packages/application`, `packages/renderer-host-contract`,
      `packages/host-runtime-contract`, `packages/integrations-lms-contract`,
      `packages/integrations-git-contract`, `packages/host-node`,
      `packages/host-browser-mock`, `packages/integrations-lms`, and
      `packages/integrations-git`.

### 1.2 Tooling Baseline

- [ ] Configure pnpm workspace wiring and TypeScript project references so all
      app and package shells resolve consistently.
- [ ] Add shared lint, format, and test tooling for the new repo, with one
      canonical command each for lint/check, typecheck, and tests.
- [ ] Define explicit `exports` maps and package entrypoints for every package
      that exists in the initial workspace.
- [ ] Add CI-friendly root scripts for build, typecheck, lint/check, and test.

### 1.3 Early Validation Spikes

- [ ] Build a minimal Electron shell spike in `apps/desktop` and record package
      size plus cold-start measurements in a checked-in note or plan artifact.
- [ ] Build a minimal docs smoke harness in `apps/docs` that mounts
      `packages/app` against browser-safe mocks and proves one shared workflow
      path runs in a browser-only environment.
- [ ] Prove one end-to-end typed `electron-trpc` workflow path with subscription
      progress or output streaming.
- [ ] Prove one real CORS-constrained provider flow executes through a Node-side
      HTTP adapter rather than the renderer.

### 1.4 Phase-1 Planning Artifacts

- [ ] Reconfirm that the retained-contract inventory is still sufficient for the
      implementation scope, and update it only if phase-1 discoveries make a
      material difference.
- [ ] Reconfirm that the test triage inventory and legacy test migration map
      still match the intended target architecture, and update them only if
      phase-1 discoveries make a material difference.
- [ ] Add a short checked-in phase-1 status note in this file if any bootstrap
      decisions materially deviate from the current plan documents.

## Phase 2: Domain and Port Foundations

### 2.1 Core Package Contracts

- [ ] Define the canonical hand-authored domain types in `packages/domain`.
- [ ] Define the workflow definition map, workflow payload types, and
      `WorkflowClient` interface in `packages/application-contract`.
- [ ] Define the cross-shell user-file boundary: `UserFileRef` /
      `UserSaveTargetRef` in `packages/application-contract`, renderer-safe
      file-picker return types in `packages/renderer-host-contract`, and
      `UserFilePort` resolution in `packages/host-runtime-contract`.
- [ ] Define renderer-safe direct host capability interfaces in
      `packages/renderer-host-contract`.
- [ ] Define application-side runtime ports in `packages/host-runtime-contract`.
- [ ] Define app-owned LMS integration contracts in
      `packages/integrations-lms-contract`.
- [ ] Define app-owned Git/provider integration contracts in
      `packages/integrations-git-contract`.

### 2.2 Shared Error and Workflow Semantics

- [ ] Define the initial shared `AppError` discriminated union and the ownership
      rules for which layer may create or normalize each variant.
- [ ] Define shared workflow call options for progress, output, and
      cancellation.
- [ ] Classify each long-running workflow by progress granularity and
      cancellation guarantee.
- [ ] Define the shared subscription event protocol that desktop transport must
      project into `WorkflowClient`.

### 2.3 Persistence and Boundary Validation

- [ ] Define the new settings and profile file formats for the rewritten app.
- [ ] Add runtime validation at untrusted boundaries only, using the selected
      validation approach from the architecture plan.
- [ ] Add browser-bundle guardrails for packages that must stay safe in docs and
      browser-side tests.

### 2.4 Phase-2 Mapping Completion

- [ ] Ensure every retained workflow-backed contract is mapped to a concrete
      shared workflow id and intended delivery surface.
- [ ] Perform a code-derived scope review for `repo.create`, `repo.clone`, and
      `repo.delete`, then record which current behaviors are retained contracts
      versus intentional redesigns before implementing those workflows.
- [ ] Ensure every major legacy test area has an explicit target destination in
      the new architecture before feature migration starts.
- [ ] Record any newly discovered contract ambiguity as an explicit note in the
      relevant plan inventory instead of allowing it to become an accidental
      requirement.

## Phase 3: Application Layer Rewrite

### 3.1 Infrastructure Replacements

- [ ] Implement boundary validation with `zod` at the agreed untrusted
      boundaries.
- [ ] Implement CSV parsing and serialization adapters with `papaparse`.
- [ ] Implement Excel import and export adapters with `xlsx`.
- [ ] Implement GitHub provider adapters behind app-owned ports using
      `@octokit/rest`.
- [ ] Implement GitLab provider adapters behind app-owned ports using
      `@gitbeaker/rest`.
- [ ] Implement the CLI command tree foundation with `commander`.

### 3.2 Thin Host and Remote Adapters

- [ ] Implement the Node-side Gitea adapter with built-in `fetch`.
- [ ] Implement the Node-side Canvas adapter with built-in `fetch`.
- [ ] Implement the Node-side Moodle adapter with built-in `fetch`.
- [ ] Implement `ProcessPort` over `child_process.spawn` for system Git CLI
      execution.
- [ ] Implement the filesystem and batch execution host seams required by
      repository workflows without embedding business rules in host adapters.
- [ ] Ensure the Git execution layer already supports the broader command shape
      needed for future inspection flows such as follow-aware history and blame.

### 3.3 Shared Domain Modules

- [ ] Port roster normalization into `packages/domain` with invariant tests.
- [ ] Port system group-set generation and repair logic into `packages/domain`
      with invariant tests.
- [ ] Port group naming, slugging, and pattern/glob behavior into
      `packages/domain` with invariant tests.
- [ ] Port group-set import/export pure semantics into `packages/domain` with
      invariant tests.
- [ ] Port assignment validation and selection rules into `packages/domain` with
      invariant tests.
- [ ] Port repository planning and collision rules into shared TypeScript
      modules, keeping pure planning logic separate from host execution.

### 3.4 Application Use-Cases

- [ ] Implement profile lifecycle workflows in `packages/application`.
- [ ] Implement app settings and connection verification workflows in
      `packages/application`.
- [ ] Implement roster import/export and system group-set workflows in
      `packages/application`.
- [ ] Implement LMS group-set discovery and sync workflows in
      `packages/application`.
- [ ] Implement group-set file preview/export workflows in
      `packages/application`.
- [ ] Implement Git username import/verification and validation workflows in
      `packages/application`.
- [ ] Implement repository create/clone/delete workflows in
      `packages/application`.

### 3.5 Phase-3 Tests

- [ ] Add or migrate invariant tests alongside each domain module as it lands.
- [ ] Add `packages/application` workflow tests alongside each workflow as it
      lands.
- [ ] Add adapter tests for LMS, Git/provider, and host-node boundaries as they
      land.
- [ ] Keep docs-required packages browser-safe as each migrated concern lands,
      and add guardrail tests where breakage would be easy to miss.

## Phase 4: React App Refactor

### 4.1 Shared App Structure

- [ ] Port the current React app into `packages/app` on top of the new package
      boundaries.
- [ ] Replace legacy backend command invocation patterns with injected
      `WorkflowClient` calls.
- [ ] Replace old generated-binding assumptions with direct use of shared domain
      types and workflow contracts.

### 4.2 State and UX Behavior

- [ ] Rebuild profile, settings, and connection flows on the new shared
      contracts.
- [ ] Rebuild roster editing, import/export, and validation flows on the new
      shared contracts.
- [ ] Rebuild group-set and assignment flows on the new shared contracts.
- [ ] Rebuild repository operation flows on the new shared contracts.
- [ ] Validate undo/redo, async checkpoints, and optimistic-update behavior in
      the new app architecture.

### 4.3 App Tests

- [ ] Rewrite retained high-value UI and state tests into `packages/app`.
- [ ] Add missing workflow-focused app tests for repository operations and other
      current low-confidence areas.

## Phase 5: Electron Shell

### 5.1 Desktop Host Wiring

- [ ] Implement the Electron main tRPC router for all shared application
      workflows.
- [ ] Implement the main-side exhaustive workflow registry derived from the
      shared workflow definition map.
- [ ] Implement preload wiring for the `electron-trpc` IPC link.
- [ ] Implement the desktop `WorkflowClient` adapter over the inferred tRPC
      client, with exhaustive renderer-side bindings derived from the same
      shared workflow definition map.
- [ ] Inject the desktop `WorkflowClient` into `packages/app` without allowing
      ad hoc IPC paths outside the router.

### 5.2 Desktop Validation

- [ ] Add minimal but meaningful desktop integration coverage for the Electron
      bridge.
- [ ] Add end-to-end coverage for at least one core workflow path and at least
      one repository-operation flow.
- [ ] Reconfirm that packaging and distribution concerns remain isolated to the
      shell and have not leaked into shared packages.

## Phase 6: CLI

### 6.1 Command Surface

- [ ] Implement the new CLI command families on top of shared application
      workflows.
- [ ] Ensure the intended major command families exist: `profile`, `roster`,
      `lms`, `lms cache`, `git`, `repo`, and `validate`.
- [ ] Keep shell-local output formatting concerns isolated inside `apps/cli`.

### 6.2 CLI Tests

- [ ] Add CLI help/output/golden tests for the command tree.
- [ ] Add CLI tests for the retained workflow-backed command behaviors.

## Phase 7: Docs Demo

### 7.1 Browser-Safe Delivery

- [ ] Port the docs site into `apps/docs`.
- [ ] Implement browser-safe mocks in `packages/host-browser-mock` for the docs
      demo and browser-side tests.
- [ ] Implement a local browser-safe `WorkflowClient` adapter for docs.
- [ ] Ensure the standalone demo exercises the same major UI flows without
      requiring Electron or Node-only APIs in browser bundles.

### 7.2 Docs Validation

- [ ] Add docs smoke tests that mount the app against browser-safe mocks.
- [ ] Add docs checks for retained shared workflow alignment where the docs demo
      depends on those workflows.

## Final Migration Closure

- [ ] Verify that all hard requirements from [plan.md](./plan.md) are satisfied
      in the new repo.
- [ ] Verify that no generated backend bindings, Rust code, Tauri runtime code,
      or legacy-settings migration logic remain in the target implementation.
- [ ] Verify that all retained high-value contracts have explicit automated
      coverage in the appropriate target layers.
- [ ] Add a final implementation status note here summarizing any intentional
      deviations from the original plan documents.
