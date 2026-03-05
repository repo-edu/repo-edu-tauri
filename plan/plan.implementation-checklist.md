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

- [x] Create the `../repo-edu` repository root with `package.json`,
      `pnpm-workspace.yaml`, `tsconfig.base.json`, and the base workspace
      directory layout from [plan.architecture.md](./plan.architecture.md).
- [x] Create empty app shells for `apps/desktop`, `apps/cli`, and `apps/docs`
      with build entrypoints that compile successfully.
- [x] Create empty package shells for `packages/ui`, `packages/app`,
      `packages/domain`, `packages/application-contract`,
      `packages/application`, `packages/renderer-host-contract`,
      `packages/host-runtime-contract`, `packages/integrations-lms-contract`,
      `packages/integrations-git-contract`, `packages/host-node`,
      `packages/host-browser-mock`, `packages/integrations-lms`, and
      `packages/integrations-git`.

### 1.2 Tooling Baseline

- [x] Configure pnpm workspace wiring and TypeScript project references so all
      app and package shells resolve consistently.
- [x] Add shared lint, format, and test tooling for the new repo, with one
      canonical command each for lint/check, typecheck, and tests.
- [x] Define explicit `exports` maps and package entrypoints for every package
      that exists in the initial workspace.
- [x] Add CI-friendly root scripts for build, typecheck, lint/check, and test.

### 1.3 Early Validation Spikes

- [x] Build a minimal Electron shell spike in `apps/desktop` and record package
      size plus cold-start measurements in a checked-in note or plan artifact.
- [x] Build a minimal docs smoke harness in `apps/docs` that mounts
      `packages/app` against browser-safe mocks and proves one shared workflow
      path runs in a browser-only environment.
- [x] Prove one end-to-end typed `electron-trpc` workflow path with subscription
      progress or output streaming.
- [x] Prove one real CORS-constrained provider flow executes through a Node-side
      HTTP adapter rather than the renderer.

### 1.4 Phase-1 Planning Artifacts

- [x] Reconfirm that the retained-contract inventory is still sufficient for the
      implementation scope, and update it only if phase-1 discoveries make a
      material difference.
- [x] Reconfirm that the test triage inventory and legacy test migration map
      still match the intended target architecture, and update them only if
      phase-1 discoveries make a material difference.
- [x] Add a short checked-in phase-1 status note in this file if any bootstrap
      decisions materially deviate from the current plan documents.

### Phase-1 Status Note

Phase-1 bootstrap is complete. Two material deviations from the plan documents:

1. **`electron-trpc` replaced with `trpc-electron`.** `electron-trpc` v0.7.1 is
   incompatible with tRPC v11 — its `ipcLink` accesses
   `runtime.transformer.serialize()` which no longer exists in v11's link
   runtime. The `trpc-electron` fork (mat-sz) provides tRPC v11 support with an
   identical API surface. Import paths changed from
   `electron-trpc/{main,renderer}` to `trpc-electron/{main,renderer}`. The
   plan's architecture references to `electron-trpc` should be read as
   `trpc-electron` going forward.

2. **Preload scripts must be CommonJS, not ESM.** Electron sandbox mode rejects
   ESM preload scripts. The electron-vite preload build is configured with
   `rollupOptions.output.format: "cjs"` and outputs `preload.cjs`.

Neither deviation changes the retained-contract inventory, the test triage
inventory, or the legacy test migration map. All three documents remain current.
The `WorkflowEvent` subscription protocol, `HttpPort` port injection pattern,
and shared use-case architecture all validated as designed.

## Phase 2: Domain and Port Foundations

### 2.1 Core Package Contracts

- [x] Define the canonical hand-authored domain types in `packages/domain`.
- [x] Define the workflow definition map, workflow payload types, and
      `WorkflowClient` interface in `packages/application-contract`.
- [x] Define the cross-shell user-file boundary: `UserFileRef` /
      `UserSaveTargetRef` in `packages/application-contract`, renderer-safe
      file-picker return types in `packages/renderer-host-contract`, and
      `UserFilePort` resolution in `packages/host-runtime-contract`.
- [x] Define renderer-safe direct host capability interfaces in
      `packages/renderer-host-contract`.
- [x] Define application-side runtime ports in `packages/host-runtime-contract`.
- [x] Define app-owned LMS integration contracts in
      `packages/integrations-lms-contract`.
- [x] Define app-owned Git/provider integration contracts in
      `packages/integrations-git-contract`.

### 2.2 Shared Error and Workflow Semantics

- [x] Define the initial shared `AppError` discriminated union and the ownership
      rules for which layer may create or normalize each variant.
- [x] Define shared workflow call options for progress, output, and
      cancellation.
- [x] Classify each long-running workflow by progress granularity and
      cancellation guarantee.
- [x] Define the shared subscription event protocol that desktop transport must
      project into `WorkflowClient`.

### 2.3 Persistence and Boundary Validation

- [x] Define the new settings and profile file formats for the rewritten app.
- [x] Add runtime validation at untrusted boundaries only, using the selected
      validation approach from the architecture plan.
- [x] Add browser-bundle guardrails for packages that must stay safe in docs and
      browser-side tests.

### 2.4 Phase-2 Mapping Completion

- [x] Ensure every retained workflow-backed contract is mapped to a concrete
      shared workflow id and intended delivery surface.
- [x] Perform a code-derived scope review for `repo.create`, `repo.clone`, and
      `repo.delete`, then record which current behaviors are retained contracts
      versus intentional redesigns before implementing those workflows.
- [x] Ensure every major legacy test area has an explicit target destination in
      the new architecture before feature migration starts.
- [x] Record any newly discovered contract ambiguity as an explicit note in the
      relevant plan inventory instead of allowing it to become an accidental
      requirement.

Phase-2 completion note (2026-03-04):

- The implementation workspace now contains the hand-authored contract packages
  and browser-safe proof points required by this phase.
- The planning gates for workflow mapping, repository-workflow scope review, and
  test-layer destinations are recorded in the companion phase-2 plan
  inventories.

## Phase 3: Application Layer Rewrite

### 3.1 Infrastructure Replacements

- [x] Implement boundary validation with `zod` at the agreed untrusted
      boundaries.
- [x] Implement CSV parsing and serialization adapters with `papaparse`.
- [x] Implement Excel import and export adapters with `xlsx`.
- [x] Implement GitHub provider adapters behind app-owned ports using
      `@octokit/rest`.
- [x] Implement GitLab provider adapters behind app-owned ports using
      `@gitbeaker/rest`.
- [x] Implement the CLI command tree foundation with `commander`.

### 3.2 Thin Host and Remote Adapters

- [x] Implement the Node-side Gitea adapter with built-in `fetch`.
- [x] Implement the Node-side Canvas adapter with built-in `fetch`.
- [x] Implement the Node-side Moodle adapter with built-in `fetch`.
- [x] Implement `ProcessPort` over `child_process.spawn` for system Git CLI
      execution.
- [x] Implement the filesystem and batch execution host seams required by
      repository workflows without embedding business rules in host adapters.
- [x] Ensure the Git execution layer already supports the broader command shape
      needed for future inspection flows such as follow-aware history and blame.

### 3.3 Shared Domain Modules

- [x] Port roster normalization into `packages/domain` with invariant tests.
- [x] Port system group-set generation and repair logic into `packages/domain`
      with invariant tests.
- [x] Port group naming, slugging, and pattern/glob behavior into
      `packages/domain` with invariant tests.
- [x] Port group-set import/export pure semantics into `packages/domain` with
      invariant tests.
- [x] Port assignment validation and selection rules into `packages/domain` with
      invariant tests.
- [x] Port repository planning and collision rules into shared TypeScript
      modules, keeping pure planning logic separate from host execution.

### 3.4 Application Use-Cases

- [x] Implement profile lifecycle workflows in `packages/application`.
- [x] Implement app settings and connection verification workflows in
      `packages/application`.
- [x] Implement roster import/export and system group-set workflows in
      `packages/application`. Note: Current implementation is CSV-only over the
  existing text-based `UserFilePort`; XLSX import/export currently returns a
  validation error until binary file-port support is introduced.
- [x] Implement LMS group-set discovery and sync workflows in
      `packages/application`. Note: `groupSet.fetchAvailableFromLms` and
  `groupSet.syncFromLms` are implemented in `packages/application` and exposed
  in `@repo-edu/application-contract`.
- [x] Implement group-set file preview/export workflows in
      `packages/application`. Note: `groupSet.previewImportFromFile`,
  `groupSet.previewReimportFromFile`, and `groupSet.export` are implemented for
  CSV (plus YAML export); XLSX preview/export remains blocked by text-only
  `UserFilePort`.
- [x] Implement Git username import/verification and validation workflows in
      `packages/application`. Note: `gitUsernames.import` now applies CSV
  updates and performs provider verification when a Git connection is
  configured. A separate `gitUsernames.verify` workflow id is still absent from
  `@repo-edu/application-contract`.
- [x] Implement repository create/clone/delete workflows in
      `packages/application`. Note: `repo.create`, `repo.clone`, and
  `repo.delete` are implemented with shared assignment-scoped planning in
  `packages/domain`; clone/delete now use app-owned Git provider operations plus
  host `GitCommandPort`/`FileSystemPort` orchestration.

### 3.5 Phase-3 Tests

- [x] Add or migrate invariant tests alongside each domain module as it lands.
- [x] Add `packages/application` workflow tests alongside each workflow as it
      lands.
- [x] Add adapter tests for LMS, Git/provider, and host-node boundaries as they
      land.
- [x] Keep docs-required packages browser-safe as each migrated concern lands,
      and add guardrail tests where breakage would be easy to miss. Note: Added
  explicit docs guardrail test in
  `apps/docs/src/__tests__/browser-guardrail.test.ts` to block Node-only import
  leakage into docs-required shared packages.

## Phase 4: React App Refactor

### 4.1 Shared App Structure

- [x] Port the current React app into `packages/app` on top of the new package
      boundaries.
- [x] Replace legacy backend command invocation patterns with injected
      `WorkflowClient` calls.
- [x] Replace old generated-binding assumptions with direct use of shared domain
      types and workflow contracts.

### 4.2 State and UX Behavior

- [x] Rebuild profile, settings, and connection flows on the new shared
      contracts.
- [x] Rebuild roster editing, import/export, and validation flows on the new
      shared contracts.
- [x] Rebuild group-set and assignment flows on the new shared contracts.
- [x] Rebuild repository operation flows on the new shared contracts.
- [x] Validate undo/redo, async checkpoints, and optimistic-update behavior in
      the new app architecture.

### 4.3 App Tests

- [x] Rewrite retained high-value UI and state tests into `packages/app`.
- [x] Add missing workflow-focused app tests for repository operations and other
      current low-confidence areas.

## Phase 5: Electron Shell

### 5.1 Desktop Host Wiring

- [x] Implement the Electron main tRPC router for all shared application
      workflows.
- [x] Implement the main-side exhaustive workflow registry derived from the
      shared workflow definition map.
- [x] Implement preload wiring for the `electron-trpc` IPC link.
- [x] Implement the desktop `WorkflowClient` adapter over the inferred tRPC
      client, with exhaustive renderer-side bindings derived from the same
      shared workflow definition map.
- [x] Inject the desktop `WorkflowClient` into `packages/app` without allowing
      ad hoc IPC paths outside the router.

### 5.2 Desktop Validation

- [x] Add minimal but meaningful desktop integration coverage for the Electron
      bridge.
- [x] Add end-to-end coverage for at least one core workflow path and at least
      one repository-operation flow.
- [x] Reconfirm that packaging and distribution concerns remain isolated to the
      shell and have not leaked into shared packages.

## Phase 6: CLI

### 6.1 Command Surface

- [x] Implement the new CLI command families on top of shared application
      workflows.
- [x] Ensure the intended major command families exist: `profile`, `roster`,
      `lms`, `lms cache`, `git`, `repo`, and `validate`.
- [x] Keep shell-local output formatting concerns isolated inside `apps/cli`.

### 6.2 CLI Tests

- [x] Add CLI help/output/golden tests for the command tree.
- [x] Add CLI tests for the retained workflow-backed command behaviors.

## Phase 7: Docs Demo

### 7.1 Browser-Safe Delivery

- [x] Port the docs site into `apps/docs`.
- [x] Implement browser-safe mocks in `packages/host-browser-mock` for the docs
      demo and browser-side tests.
- [x] Implement a local browser-safe `WorkflowClient` adapter for docs.
- [x] Ensure the standalone demo exercises the same major UI flows without
      requiring Electron or Node-only APIs in browser bundles.

### 7.2 Docs Validation

- [x] Add docs smoke tests that mount the app against browser-safe mocks.
- [x] Add docs checks for retained shared workflow alignment where the docs demo
      depends on those workflows.

## Final Migration Closure

- [x] Verify that all hard requirements from [plan.md](./plan.md) are satisfied
      in the new repo.
- [x] Verify that no generated backend bindings, Rust code, Tauri runtime code,
      or legacy-settings migration logic remain in the target implementation.
- [x] Verify that all retained high-value contracts have explicit automated
      coverage in the appropriate target layers.
- [x] Add a final implementation status note here summarizing any intentional
      deviations from the original plan documents.

### Final Migration Closure Status Note (2026-03-05)

Hard-requirement verification snapshot against `/Users/dvbeek/1-repos/repo-edu`:

- Full workspace validation is green:
  - `pnpm typecheck`
  - `pnpm test`
  - `pnpm build`
- Desktop shell validation is green for retained bridge/e2e checks:
  - `pnpm --filter @repo-edu/desktop run validate:shell-boundary`
  - `/bin/zsh -lc 'REPO_EDU_DESKTOP_VALIDATE_TRPC=1 pnpm exec electron
    ./out/main/main.js'`
- CLI/docs retained-surface validation is green:
  - `apps/cli/src/__tests__/cli.test.ts` (command tree + workflow-backed
    behavior)
  - `apps/docs/src/__tests__/docs-smoke.test.ts`
  - `apps/docs/src/__tests__/workflow-alignment.test.ts`

Legacy-artifact elimination verification:

- Rust/Tauri artifacts absent in target repo:
  - `find /Users/dvbeek/1-repos/repo-edu -type f \( -name "*.rs" -o -name
  "Cargo.toml" -o -name "Cargo.lock" \) | wc -l` -> `0` - `rg -n
  "@tauri-apps|tauri::|src-tauri|Tauri" ... | wc -l` -> `0`
- Generated-binding and legacy-migration footprints absent in app/package code:
  - `rg -n
    "legacy.*(settings|profile).*migrat|migrat(e|ion).*legacy|schema-to-bindings|generated
    bindings|backend-interface|command manifest" ... | wc -l` -> `0`

Retained high-value contract coverage verification:

- Domain invariants and pure product rules:
  - `packages/domain/src/__tests__/roster.test.ts`
  - `packages/domain/src/__tests__/groups.test.ts`
  - `packages/domain/src/__tests__/group-set-import-export.test.ts`
  - `packages/domain/src/__tests__/validation.test.ts`
  - `packages/domain/src/__tests__/repository-planning.test.ts`
- Workflow orchestration and host-crossing semantics:
  - `packages/application/src/__tests__/validation.test.ts`
- App/store workflow wiring and UI-level retained semantics:
  - `packages/app/src/__tests__/profile-store.test.ts`
  - `packages/app/src/__tests__/app-settings-store.test.ts`
  - `packages/app/src/__tests__/issues-utils.test.ts`
  - `packages/app/src/__tests__/repository-workflow.test.ts`
- CLI retained contract layer:
  - `apps/cli/src/__tests__/cli.test.ts`
- Docs retained contract layer:
  - `apps/docs/src/__tests__/browser-guardrail.test.ts`
  - `apps/docs/src/__tests__/docs-smoke.test.ts`
  - `apps/docs/src/__tests__/workflow-alignment.test.ts`

Intentional deviations from original plan documents:

1. Desktop IPC transport package is `trpc-electron` instead of `electron-trpc`
   due tRPC v11 compatibility (recorded in Phase-1 status note).
2. Desktop preload output is CommonJS (`preload.cjs`) instead of ESM to satisfy
   Electron sandbox/runtime constraints (recorded in Phase-1 status note).
3. `UserFilePort` remains text-based in current implementation; XLSX paths that
   require binary file-port support intentionally return validation errors for
   now (recorded in Phase-3 notes).
4. `plan.workflow-mapping.md` remains a draft planning map; several draft
   workflow ids were intentionally consolidated into existing workflows or
   shell-local behavior (for example `groupSet.cache.*`,
   `roster.ensureSystemGroupSets`, and extended profile lifecycle ids), while
   implemented workflow authority is the shared `workflowCatalog` in
   `packages/application-contract` plus the validated desktop/cli/docs
   invocation surfaces.
