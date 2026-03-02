# Repo Edu Tauri -> Electron Migration Plan

## Objective

Rebuild the current `repo-edu-tauri` application as a greenfield Electron application in
`../repo-edu` with unchanged user-facing functionality, higher long-term code quality, and a
strictly smaller platform layer.

The migration is a redesign, not a port. Migration effort is intentionally ignored. The target
architecture should optimize for:

- clear ownership boundaries
- minimal Electron-specific code
- maximum TypeScript reuse across desktop, CLI, tests, and docs demo
- zero generated backend bindings
- zero Rust in the new project
- zero settings migration logic

## Hard Requirements

- Preserve all existing app behavior and feature scope.
- Replace the complete Rust backend with TypeScript.
- Replace the Rust CLI with a TypeScript CLI.
- Remove all generated backend bindings and the schema-to-bindings pipeline.
- Keep the desktop backend limited to host capabilities, primarily file I/O and process-native
  integration.
- Migrate the docs demo / mock simulation so it still runs outside the desktop shell.
- Do not add any migration path for legacy settings or profiles from Tauri.

## Current State Summary

The current codebase is already split in a way that makes a clean rewrite possible:

- `packages/app-core` is the main React application and already contains most UI and state logic.
- `packages/backend-interface` defines a very large generated `BackendAPI` contract.
- `packages/backend-mock` provides an in-memory implementation used by tests and the docs demo.
- `apps/repo-manage/src` is a thin Tauri entrypoint.
- `apps/repo-manage/src-tauri` exposes Tauri commands over a large Rust backend.
- `apps/repo-manage/core` contains the shared Rust business logic currently reused by GUI and CLI.
- `apps/repo-manage/cli` is a Rust CLI with domain-based subcommands.
- `docs` mounts the same React app against the mock backend for simulation.

This means the right target is not "Electron + translated Rust command handlers". The right target
is "shared TypeScript domain/app services + a very thin Electron host".

## Target Architecture

### Architectural Direction

Build the new `../repo-edu` as a pnpm TypeScript monorepo with a single canonical domain model and
no code generation layer between UI and native host capabilities.

Use these boundaries:

1. Pure domain logic
2. Application orchestration
3. Host adapters
4. Delivery shells (Electron desktop, CLI, docs demo)

The main process must not become a second application backend. It should expose only narrow host
capabilities and long-running task execution primitives. All business rules, validation, diffing,
LMS behavior, group-set behavior, and profile logic should live in shared TypeScript packages.

Network access to third-party APIs is the key exception to "renderer-first" execution. Canvas,
Moodle, GitHub, GitLab, and Gitea APIs cannot be called directly from a browser renderer because
their CORS policies are not designed for arbitrary browser origins. External HTTP calls must execute
in Node-owned host adapters behind an explicit port, while orchestration and decision-making remain
in shared TypeScript use-cases.

### Execution Model

Default execution path:

- the renderer owns user-triggered application use-cases
- `packages/application` use-cases execute in the renderer by default
- when a use-case needs host capabilities, it calls a typed port from `packages/host-contract`
- that port crosses preload/IPC into `apps/desktop` and delegates to `packages/host-node`
- the Node-side adapter performs the side effect and returns typed results or progress events

In concrete terms, LMS and remote Git workflows execute as:

- renderer event
- shared use-case
- integration adapter
- `HttpPort`
- preload/IPC bridge
- Node-owned `fetch`
- typed response back through the same boundary

This is intentionally not "Electron main owns the workflow." The renderer owns orchestration and
decision-making. The main process owns only the side-effectful capability behind the port.

For workflows that would otherwise require many fine-grained IPC round-trips, the renderer should
compute and validate a task plan in shared TS, then hand that plan to one long-running host task
through `TaskRunnerPort` with streamed progress events. The host may execute the task, but it must
not reintroduce app-specific command surfaces or hidden business rules.

### Proposed Workspace Layout

```text
../repo-edu/
├── package.json
├── pnpm-workspace.yaml
├── tsconfig.base.json
├── apps/
│   ├── desktop/              # Electron shell: main + preload + packaging
│   ├── cli/                  # TypeScript CLI (replaces Rust redu)
│   └── docs/                 # Astro docs site with standalone simulation
└── packages/
    ├── ui/                   # Shared visual components
    ├── app/                  # React application shell and feature modules
    ├── domain/               # Canonical types, invariants, pure transforms, boundary codecs
    ├── application/          # Use-cases orchestrating domain + ports
    ├── host-contract/        # Small typed host capability interfaces
    ├── host-node/            # Node/Electron/CLI host adapter implementation
    ├── host-browser-mock/    # Browser-safe mock host for docs/tests
    ├── integrations-lms/     # Canvas/Moodle TS clients
    └── integrations-git/     # Git and provider integrations in TS
```

This replaces the current split between `app-core`, generated bindings, Rust core, and Rust CLI
with one shared TypeScript architecture.

### Execution Defaults

Use these default implementation choices unless a concrete constraint forces a change:

- `electron-vite` for Electron development/build wiring
- `electron-builder` for desktop packaging
- built-in Node `fetch` inside `host-node` for HTTP clients
- `simple-git` for git CLI-backed repository operations
- `papaparse` for CSV parsing/serialization
- `xlsx` (SheetJS) for Excel import/export
- `commander` for the CLI command tree

These are implementation defaults, not architectural boundaries. They can change without changing
the package model described above.

Workspace defaults should also be fixed early:

- use TypeScript project references for package compilation boundaries
- define explicit `exports` maps for every package
- use conditional entrypoints only where Node vs browser delivery actually differs
- keep package resolution rules identical across Electron, CLI, tests, and docs builds

## Core Design Decisions

### 1. Remove Generated Bindings Entirely

The generated bindings layer should not be recreated.

Instead:

- define the domain model directly in TypeScript inside `packages/domain`
- keep hand-authored TypeScript types as the default source of truth for internal domain and
  application models
- use runtime schemas (for example Zod or Valibot) only at untrusted boundaries where validation is
  required
- share the same domain types everywhere: app, CLI, docs demo, tests, and host adapters

This removes:

- `packages/backend-interface`
- `packages/app-core/src/bindings/commands.ts`
- `apps/repo-manage/src/bindings/tauri.ts`
- all schema-driven Rust DTO generation
- the command manifest as an IPC contract definition

The new system should use hand-authored, explicit TypeScript module APIs, not generated command
bridges.

Runtime schemas should be applied selectively, not universally. Use them for:

- settings/profile file loading
- import file decoding
- preload-to-renderer payload validation where needed
- external API response normalization when defensive validation is justified

Do not force every internal domain type to be declared through a runtime schema if a plain
TypeScript type is sufficient.

### 2. Move Business Logic to Shared TS Packages

Everything currently implemented in Rust that is not inherently host-specific should move into
shared TS packages:

- roster normalization
- system group set generation
- group import/export parsing
- LMS sync merge logic
- assignment validation
- glob / pattern matching
- slug and naming rules
- repository operation planning
- profile read/write validation

This logic belongs in `packages/domain` and `packages/application`, not in Electron main.

The key rule is:

- decision-making in shared code
- side effects in adapters

### 3. Route External HTTP Through a Host Port

Do not let shared integrations call global `fetch` directly.

Instead:

- add an `HttpPort` (or `FetchPort`) to `packages/host-contract`
- require `packages/integrations-lms` and remote-provider portions of `packages/integrations-git`
  to depend on that port
- implement the real desktop/CLI version in `host-node` using Node `fetch`
- implement the docs/test version in `host-browser-mock` with canned or simulated responses

This keeps network code shared and testable while respecting renderer CORS constraints.

In production Node contexts (`apps/desktop` main-side adapters and `apps/cli`), `host-node` should
provide the default `HttpPort` implementation backed by native Node `fetch`. Consumers should
depend on the abstraction, but production wiring should not require repetitive custom plumbing.

The default architecture should be:

- use-cases run in shared TypeScript (`packages/application`)
- use-cases orchestrate domain logic and call ports
- ports own side effects such as HTTP, filesystem access, dialogs, secret storage, and process
  execution
- Electron preload exposes only typed port capabilities, not app-specific backend commands

### 4. Make Progress Events a First-Class Contract

Long-running workflows must use one explicit progress model across renderer, CLI, tests, and
desktop IPC.

Default rule:

- `packages/application` owns workflow orchestration
- when a port performs long-running work, it reports typed progress events back to the caller
- progress crossing preload/IPC boundaries must use serialized, versioned event payloads
- do not split one workflow across opaque main-process orchestration and renderer-local callbacks

Model progress as part of the host contract, not as an ad hoc callback shape hidden inside
individual features.

### 5. Use Promises With Typed Errors

Shared TypeScript APIs should use `Promise<T>` results and throw typed `AppError` values instead of
recreating Rust-style `Result<T, E>` wrappers.

Rules:

- `packages/application` use-cases return `Promise<T>`
- host ports return `Promise<T>` or progress-aware async abstractions that resolve/reject normally
- renderer, CLI, and tests handle failures through typed errors, not tagged union boilerplate
- errors that cross preload/IPC boundaries must be serialized and re-hydrated into typed `AppError`
  values

### 6. Keep the Electron Backend Very Small

The Electron host should expose a narrow capability surface through `contextBridge`:

- file open/save dialogs
- file read/write
- directory listing and path existence checks
- authenticated HTTP requests to external APIs through a generic `HttpPort`
- safe path utilities where needed
- shell open for external URLs
- OS/theme/window primitives
- secret storage primitives
- background task execution for filesystem-heavy or process-heavy operations

The Electron main process should not expose app-specific commands like `syncGroupSet()` or
`validateAssignment()`. Those become plain TypeScript functions in shared packages.

Where local filesystem access is required for repository operations, the renderer should assemble a
fully validated operation plan in shared TS code, then hand that plan to a minimal Node-side task
runner for execution.

### 7. Preserve a Browser-Safe Simulation Layer

The docs demo must remain a first-class delivery target.

To do that:

- the React app must depend on `host-contract`, not Electron APIs
- `host-browser-mock` must implement the same host contract in memory
- the docs demo should mount the same app package with the mock host
- no Electron import may leak into packages consumed by `apps/docs`

This keeps the current "real UI + mock backend" capability, but with a smaller and cleaner
contract.

### 8. Rebuild the CLI on the Same Application Layer

The CLI should become a Node-based TypeScript application in `apps/cli`.

It must reuse:

- the same domain model
- the same LMS and Git integration packages
- the same profile/settings persistence code
- the same validation and repository planning logic

It should not call the Electron app. It should use `host-node` directly.

The command surface should remain behaviorally equivalent to the current CLI:

- `profile`
- `roster`
- `lms`
- `git`
- `repo`
- `validate`

## Package Responsibilities

Decision rule:

- if a function accepts only domain data, applies deterministic rules, and returns domain data, it
  belongs in `packages/domain`
- if a function depends on a port directly or transitively, coordinates side effects, or represents
  an end-to-end workflow, it belongs in `packages/application`
- if `packages/application` becomes thin pass-through glue for a behavior, move the pure logic down
  into `packages/domain` and keep only orchestration in `packages/application`

### `packages/domain`

Pure, deterministic, side-effect-free logic.

Responsibilities:

- canonical domain types and invariants
- boundary codecs/schemas only for domain data that enters from untrusted sources
- invariants and parsing
- roster and group-set transforms
- assignment resolution
- validation rules
- serialization formats
- patch/diff models

Must not import:

- Electron
- Node filesystem APIs
- fetch implementations
- UI code

### `packages/application`

Use-cases that coordinate domain logic with ports.

Responsibilities:

- load/save profile workflows
- LMS import/sync orchestration
- CSV import/export orchestration
- repo create/clone/delete workflows
- settings loading and normalization
- operation progress event modeling
- shared error boundaries for typed `AppError` handling

Depends on abstract ports from `host-contract` and adapter interfaces from integration packages.

### `packages/host-contract`

A very small, explicit set of capabilities that can be implemented by:

- Electron main/preload
- CLI Node runtime
- docs/test mocks

Recommended contract families:

- `FileSystemPort`
- `DialogPort`
- `HttpPort`
- `ShellPort`
- `TaskRunnerPort`
- `CredentialPort`

The host contract also owns the canonical cross-boundary shapes for:

- progress event payloads
- serialized error payloads
- capability-specific request/response types

This replaces the huge generated `BackendAPI`.

Only define host ports that have concrete application consumers. Do not add speculative port
families before a real workflow needs them.

`TaskRunnerPort` is specifically for coarse-grained execution of prevalidated host tasks such as
filesystem-heavy repository operations or multi-step local workflows. It must accept typed task
plans, stream typed progress, and return typed results. It must not become a generic backdoor for
recreating the current backend-command layer under a different name.

### `packages/host-node`

Concrete Node implementations for desktop and CLI.

Responsibilities:

- disk access
- path operations
- Node-side HTTP execution
- child process or library-backed git execution
- OS-backed secret storage
- secure host execution of long-running tasks
- Electron IPC glue (desktop only entrypoint wrappers)

Keep application knowledge out of this package. It should implement ports, not business rules.

### `packages/app`

The React application. This is the evolution of `packages/app-core`, but now it should depend on
the application layer directly instead of calling a command facade.

Responsibilities:

- components
- stores
- feature controllers
- host wiring through React context
- user-triggered workflows that call shared use-cases

Refactor goal:

- remove "backend command" thinking
- replace it with direct use-case invocation over typed ports

### `packages/integrations-lms`

TypeScript LMS clients for Canvas and Moodle.

Responsibilities:

- API clients
- auth/token helpers
- response normalization into domain types
- dependency on `HttpPort`, not direct renderer `fetch`

These should expose thin external adapters. Merge and business interpretation stay in
`packages/application` / `packages/domain`.

### `packages/integrations-git`

TypeScript Git platform and repository adapters.

Responsibilities:

- platform verification
- repository provisioning integration
- clone/delete execution helpers
- remote provider API calls through `HttpPort`

The target design should split:

- repository planning in shared application logic
- repository execution in host-backed adapters

## Package Dependency Rules

Enforce a strict one-way dependency graph:

- `packages/domain` depends on no other workspace package
- `packages/host-contract` may depend only on `packages/domain`
- `packages/integrations-lms` and `packages/integrations-git` may depend only on
  `packages/domain` and `packages/host-contract`
- `packages/application` may depend only on `packages/domain`, `packages/host-contract`,
  `packages/integrations-lms`, and `packages/integrations-git`
- `packages/host-node` and `packages/host-browser-mock` may depend only on
  `packages/domain` and `packages/host-contract`
- `packages/app` may depend only on `packages/ui`, `packages/domain`, `packages/application`, and
  `packages/host-contract`
- `apps/desktop` composes `packages/app` with `packages/host-node`
- `apps/cli` composes `packages/application` with `packages/host-node`
- `apps/docs` composes `packages/app` with `packages/host-browser-mock`

Delivery shells are the composition roots. They construct port implementations, inject those ports
into integration clients, and inject the resulting clients into application use-cases at startup.
Shared packages must not reach for global port singletons.

Do not allow delivery shells to become backchannels that bypass the shared application layer.

## Desktop App Structure

### Electron Main

Keep `apps/desktop` intentionally small:

- window lifecycle
- preload registration
- updater/packaging integration
- host capability implementation wiring
- security defaults and CSP enforcement
- no domain logic

### Preload

Expose a minimal, versioned API to the renderer:

- do not expose raw `ipcRenderer`
- expose only typed host capabilities
- validate payloads at the bridge boundary using runtime schemas only where the payload is untrusted

Security invariants:

- `contextIsolation: true`
- `nodeIntegration: false`
- do not use the deprecated `remote` module
- apply an explicit Content Security Policy for renderer content

### Renderer

The renderer should run almost all application logic:

- profile mutations
- validation
- LMS orchestration
- import/export planning
- repository action planning

The renderer must not perform direct third-party HTTP requests. It invokes shared use-cases, which
call `host-contract` ports for external network access.

When a workflow requires multiple host interactions, prefer one validated task handoff plus
streamed progress over repeated ad hoc IPC calls. The renderer remains the composition point for
the workflow, but the host may execute the heavy side-effect segment through `TaskRunnerPort`.

The renderer should remain testable in a plain browser-like environment.

## Persistence Strategy

### Settings and Profiles

Do not implement any migration from the existing Tauri settings or profile storage.

Rules:

- define a new storage layout for the Electron project
- use explicit runtime validation on every load
- normalize invalid or partial data to defaults only within the new format
- if legacy files exist, do not auto-import, transform, or upgrade them

The new code may intentionally use different file locations, schemas, or internal representation.
Behavioral parity matters; backward compatibility with old persisted files does not.

### Credentials

Do not store LMS or Git tokens inside plain settings/profile files.

Rules:

- expose secret storage through `CredentialPort`
- desktop implementation uses Electron `safeStorage`
- CLI implementation uses OS-backed secure storage, with the concrete maintained mechanism chosen at
  implementation time
- docs/tests use in-memory mock secret storage only
- no migration of legacy secrets from the existing Tauri/keyring implementation

### Export Formats

User-visible exports that are part of app functionality should remain compatible unless there is a
clear quality reason to improve them without changing expected output semantics.

This includes:

- roster export
- assignment member export
- group set CSV import/export behavior

## Migration of Existing Functional Areas

### 1. UI and State

Start from the current `packages/app-core` feature set, but refactor heavily:

- keep component behavior
- keep tab structure
- keep state semantics where they are good
- remove command-wrapper indirection
- move business logic out of components and stores into shared use-cases

This should produce a cleaner UI layer than the current "store + backend command" split.

State management must be redesigned explicitly, not left as an implementation detail:

- stores may invoke shared use-cases directly or through thin feature controllers, but not through a
  regenerated command facade
- undo/redo must remain local to deterministic state transitions; remote or host-side effects must
  not be hidden inside undoable mutations
- async workflows that cross ports should commit state changes at explicit checkpoints so failures
  can be surfaced without corrupting undo history
- optimistic updates are allowed only where the rollback semantics are clearly defined and tested

### 2. Rust Core Rewrite

Rewrite `apps/repo-manage/core` and the crates into TypeScript by concern, not by file parity.

Do not mirror the Rust folder structure mechanically.

Rebuild the capabilities into domain-oriented TS modules:

- LMS clients
- roster domain logic
- group-set import/export logic
- validation engine
- repository workflow engine
- settings/profile persistence services

### 3. CLI Rewrite

Rebuild the CLI as a first-class TypeScript app, not a thin wrapper around desktop internals.

Requirements:

- preserve command names and command intent
- preserve output semantics where practical
- share all non-output logic with the app
- keep CLI-specific prompting/output isolated to the CLI layer

### 4. Docs Demo Rewrite

Rebuild the docs simulation around the new host contract:

- docs app imports the same `packages/app`
- docs uses `host-browser-mock`
- mock data and simulated workflows remain separate from production adapters
- the standalone demo page remains fully functional without Electron

## Delivery Phases

### Phase 1: New Monorepo Bootstrap in `../repo-edu`

- create the new workspace skeleton
- set up TypeScript, pnpm, linting, testing, and build tooling
- choose the package compilation and resolution strategy (`tsc` project references, package
  `exports`, and any conditional entrypoints)
- create `apps/desktop`, `apps/cli`, `apps/docs`
- create the shared packages listed above

Exit criteria:

- the new repo builds as an empty but coherent workspace
- package resolution behaves consistently across desktop, CLI, tests, and docs

### Phase 2: Domain and Port Foundations

- define the canonical domain types
- define all host contracts
- define progress/event/error models for all cross-boundary flows
- define persistence file formats for the new app
- define runtime validation only for boundary inputs (settings/profile files, imports, bridge payloads)

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

- define typed task plans for clone/delete and other filesystem-heavy repository operations
- implement `TaskRunnerPort` execution paths without embedding business rules in the host
- add parity tests with each host-executed repository workflow

Exit criteria:

- all former Rust behaviors exist in TS packages with tests
- parity coverage is added alongside each migrated concern, not deferred

### Phase 4: React App Refactor

- port current UI into `packages/app`
- replace backend command invocations with direct use-case calls
- wire the renderer against host contracts
- validate state-management behavior for undo/redo, async checkpoints, and optimistic updates

Exit criteria:

- the app runs in a browser-safe environment against mocks
- state transitions remain coherent under async use-cases

### Phase 5: Electron Shell

- implement the Electron main/preload host
- wire the renderer into the desktop shell
- add packaging and desktop build scripts
- implement desktop updater integration

Exit criteria:

- the desktop app runs with the real Node host adapter
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
- ensure the mock simulation remains behaviorally aligned with the app
- keep docs parity checks aligned with the same shared workflows used by app and CLI

Exit criteria:

- docs build and the standalone demo works without Electron

## Testing Strategy

The new project should be validated at the architecture boundaries, not only through UI smoke tests.

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

## Explicit Non-Goals

- No legacy settings migration.
- No attempt to preserve Rust module structure.
- No recreation of the generated bindings architecture.
- No incremental hybrid Tauri/Electron bridge.
- No "temporary" business logic in Electron main that later needs to be moved out.

## Recommended Execution Rules

- Treat `../repo-edu` as a clean rewrite and source-of-truth target.
- Copy behavior, not implementation structure.
- Prefer deleting architectural indirection rather than translating it.
- Keep Electron-specific code as a host layer, never as an application layer.
- Refuse convenience shortcuts that reintroduce backend-command sprawl.

## Key Risks

- CORS is an architectural constraint, not an implementation detail. If external HTTP is allowed to
  leak into the renderer, the desktop app will fail against real LMS and Git provider APIs.
- `simple-git` still depends on the system Git CLI. The new Node implementation must explicitly
  handle missing Git installations, version expectations, and platform-specific process behavior.
- Electron packaging increases binary size and startup overhead compared with a browser-only or CLI
  target. Packaging and startup budgets should be measured early so the app does not accumulate
  avoidable platform weight.
- Progress and error transport across preload/IPC boundaries can become a hidden second command
  layer if they are not standardized early in `host-contract`.

## Success Criteria

The migration is successful when all of the following are true:

1. `../repo-edu` is a pure TypeScript monorepo.
2. The desktop app is Electron-based and keeps Electron-specific logic minimal.
3. All current app and CLI functionality still exists with unchanged behavior.
4. The docs demo still runs the real UI against mock infrastructure.
5. There is no Rust, no Tauri, and no generated backend bindings.
6. The shared business logic is reusable across desktop, CLI, and docs without duplication.
