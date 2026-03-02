# Repo Edu Tauri -> Electron Migration Plan: Architecture

This document contains the detailed target system design referenced by [plan.md](./plan.md).

## Target Architecture

### Architectural Direction

Build the new `../repo-edu` as a pnpm TypeScript monorepo with a single canonical domain model and
no code generation layer between UI and native host capabilities.

Use these boundaries:

1. Pure domain logic
2. Workflow invocation contract
3. Application orchestration
4. Host adapters
5. Delivery shells (Electron desktop, CLI, docs demo)

The main process must not become a second application backend. It should expose only narrow host
capabilities and long-running task execution primitives. All business rules, validation, diffing,
LMS behavior, group-set behavior, and profile logic should live in shared TypeScript packages.

Network access to third-party APIs is the key exception to "renderer-first" execution. Canvas,
Moodle, GitHub, GitLab, and Gitea APIs cannot be called directly from a browser renderer because
their CORS policies are not designed for arbitrary browser origins. External HTTP calls must
execute in Node-owned host adapters behind an explicit port, while orchestration and
decision-making remain in shared TypeScript use-cases.

### Execution Model

The placement rule for code execution in the desktop shell follows directly from the package
boundary:

- **`packages/domain`** functions run wherever the caller lives. In the desktop shell, the renderer
  imports and calls domain functions directly. They are pure, synchronous, and have no port
  dependencies.
- **`packages/application-contract`** defines browser-safe workflow invocation types: typed inputs,
  results, progress/error payloads, and a narrow `WorkflowClient` interface. It contains no Node
  APIs, no Electron APIs, and no business logic.
- **`packages/application`** implements the shared use-cases. In the desktop shell, those
  implementations run Node-side behind the Electron main tRPC router. The renderer never imports
  them directly; it invokes an injected `WorkflowClient`, and the desktop shell satisfies that
  interface with an adapter that wraps the inferred tRPC client.

This rule is deterministic and requires no per-workflow analysis. The package a function lives in
determines where it executes, while `packages/app` sees only `packages/domain` plus the
browser-safe `packages/application-contract` invocation surface. The plan's existing package
responsibility definitions already encode the classification: `packages/application` contains
exactly the use-cases that orchestrate ports, and `packages/domain` contains exactly the pure
logic.

This produces two clean error channels with no mixing within a single call site:

- domain calls throw synchronous, deterministic errors handled locally by the component or store
- use-case calls surface typed `AppError` values through `WorkflowClient`; in desktop they cross
  IPC, and in docs/CLI they stay local

Per-shell execution:

- **Desktop renderer**: imports `packages/domain` and `packages/application-contract`; calls an
  injected `WorkflowClient` whose desktop adapter bridges to the tRPC client inferred from the
  Electron main router
- **CLI**: runs everything directly in Node, both domain and application, with no IPC dispatch
- **Docs demo**: runs everything in the browser, both domain and application, against mock port
  implementations from `host-browser-mock`, while `packages/app` still talks only to a local
  `WorkflowClient` adapter

The shared use-case code is identical across all three shells. Only the port implementations and
shell-specific `WorkflowClient` adapters differ. No use-case contains environment-conditional
behavior, and `packages/app` does not need shell-specific branching.

In concrete terms, LMS and remote Git workflows execute in the desktop shell as:

- renderer calls a `WorkflowClient` method
- desktop `WorkflowClient` adapter calls a tRPC subscription on the Electron main router
- Electron main invokes the shared use-case from `packages/application` with real port
  implementations
- use-case calls integration adapter -> `HttpPort` -> Node-owned `fetch`
- typed progress events stream back to renderer as subscription yields
- typed result or error returns as the final subscription value or tRPC error

The Electron main process does not own any business logic. It acts as a runtime host: it
constructs port dependencies, invokes the shared use-case, and relays results. The same use-case
code runs directly (without hosting) in the CLI and docs demo.

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
    ├── application-contract/ # Typed workflow invocation surface for the UI
    ├── application/          # Use-cases orchestrating domain + ports
    ├── host-contract/        # Small typed host capability interfaces
    ├── host-node/            # Shared Node host adapters for desktop and CLI
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
- `electron-trpc` for type-safe IPC between renderer and main process
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
bridges. IPC type safety across the Electron preload boundary is enforced by tRPC: the main
process defines a router over `packages/application` use-cases, and the desktop transport types are
inferred from that router. Separately, `packages/application-contract` defines the browser-safe
`WorkflowClient` interface plus shared workflow payload types used by `packages/app`. This replaces
the current schema-to-bindings pipeline with compiler-enforced consistency that requires no
generation step.

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
- ports own side effects such as HTTP, filesystem access, dialogs, and process execution
- Electron preload exposes the tRPC IPC link and direct host capabilities, not app-specific backend
  commands

### 4. Make Progress Events a First-Class Contract

Long-running workflows must use one explicit progress model across renderer, CLI, tests, and
desktop IPC.

Default rule:

- `packages/application` owns workflow orchestration
- `packages/application-contract` owns workflow input/result/progress/error shapes plus the
  `WorkflowClient` interface
- `packages/application` implements those signatures
- use-cases accept a typed progress callback and yield typed progress events from
  `packages/application-contract`
- in the desktop shell, long-running use-cases are exposed as tRPC subscriptions; the subscription
  yields progress events and resolves with the final result or error
- in the CLI and docs demo, the same use-cases are called directly with a local progress callback
- do not split one workflow across opaque main-process orchestration and renderer-local callbacks

Model progress in `packages/application-contract` and `packages/application` use-case signatures,
not as an ad hoc callback shape hidden inside individual features. The tRPC router and local
`WorkflowClient` adapters reuse these types automatically — no separate IPC progress contract is
needed.

### 5. Use Promises With Typed Errors

Shared TypeScript APIs should use `Promise<T>` results and throw typed `AppError` values instead of
recreating Rust-style `Result<T, E>` wrappers.

Rules:

- `packages/application` use-cases return `Promise<T>` and implement the workflow-facing
  signatures defined in `packages/application-contract`
- host ports return `Promise<T>` or progress-aware async abstractions that resolve/reject normally
- renderer, CLI, and tests handle failures through typed errors, not tagged union boilerplate
- errors that cross the IPC boundary are handled by tRPC's built-in error serialization; the
  desktop `WorkflowClient` adapter must map transport failures onto the shared `AppError` surface
  expected by `packages/app` without introducing a second serialized contract

### 6. Keep the Electron Backend Very Small

The Electron host should expose a narrow capability surface through `contextBridge`:

- file open/save dialogs
- file read/write
- directory listing and path existence checks
- safe path utilities where needed
- shell open for external URLs
- OS/theme/window primitives

Beyond these direct host capabilities, the Electron main process hosts a tRPC router. Each router
procedure maps to a shared use-case from `packages/application`, constructed with real port
implementations from `host-node`. Long-running use-cases are exposed as tRPC subscriptions that
stream typed progress events and resolve with typed results or errors. The renderer reaches these
procedures only through the desktop `WorkflowClient` adapter defined by
`packages/application-contract`.

The Electron main process must not contain app-specific business logic like group-set sync rules or
assignment validation. Those are plain TypeScript functions in shared packages that the router
procedures invoke but do not own. Adding a new use-case requires adding a router procedure — if the
procedure's types don't match the use-case signature, the TypeScript compiler rejects it.

### 7. Use the Package Boundary as the Workflow Placement Rule

The placement of code execution in the desktop shell must be deterministic and must not require
per-workflow analysis. The rule is:

- **`packages/domain`**: runs wherever the caller lives. In the desktop renderer, domain functions
  are imported and called directly. They are pure, synchronous, and side-effect-free.
- **`packages/application-contract`**: runs wherever the caller lives. It defines the typed
  workflow invocation surface used by the UI, but contains no use-case implementation.
- **`packages/application`**: is the implementation layer. It runs Node-side for the desktop shell,
  and directly in the CLI and docs composition roots. `packages/app` must never import it.

This rule derives from the package responsibility definitions already in this plan:
`packages/application` is defined to contain exactly the use-cases that orchestrate ports, and
`packages/domain` is defined to contain exactly the pure logic. If a use-case in
`packages/application` turns out to be pure pass-through with no port dependency, the plan already
requires moving it down into `packages/domain` (see Package Responsibilities). This keeps the
classification self-maintaining.

The rule produces two clean, non-overlapping error channels:

- **Domain errors**: synchronous, deterministic, caught locally by the calling component or store.
  No serialization, no IPC transport.
- **Use-case errors**: surface through `WorkflowClient` as typed `AppError` values. In desktop
  they cross IPC; in docs and CLI they stay local. The call shape in `packages/app` does not
  change.

No call site ever mixes both error channels. A renderer function either calls domain logic (local
errors) or dispatches a use-case through `WorkflowClient`, never both in the same invocation.

In the desktop shell, the transport behind `WorkflowClient` is the tRPC router in `apps/desktop`.
It provides:

- typed request identification: each procedure name and input type maps to a use-case
- typed progress event streaming: subscriptions yield progress events with types inferred from the
  use-case
- typed result or typed error: the procedure return type and error type are inferred from the
  use-case signature
- cancellation propagation: tRPC subscription cleanup triggers cancellation of the running use-case

The router is part of the desktop shell's infrastructure in `apps/desktop`, not a shared port. The
CLI does not need it (it calls use-cases directly), and the docs demo does not need it (its local
adapter calls use-cases directly against mock ports in the browser).

Contract consistency is enforced by the TypeScript compiler: `packages/application-contract`
defines the shared workflow signatures, `packages/application` implements them, the router
procedure types are inferred from those use-cases, and the desktop `WorkflowClient` adapter must
explicitly satisfy the interface. There is no separate hand-authored IPC serialization contract
that can drift. Adding a use-case to `packages/application` without registering it in the router is
not a silent failure — the desktop adapter simply cannot expose it. Registering a procedure or
adapter method with mismatched input/output types is a compile error.

### 8. Preserve a Browser-Safe Simulation Layer

The docs demo must remain a first-class delivery target.

To do that:

- the React app must depend on `host-contract` and `packages/application-contract`, not Electron
  APIs or `packages/application`
- `host-browser-mock` must implement the same host contract in memory
- the docs demo should mount the same app package with the mock host and a local `WorkflowClient`
  adapter that calls `packages/application` directly
- no Electron import may leak into packages consumed by `apps/docs`

This keeps the current "real UI + mock backend" capability, but with a smaller and cleaner
contract.

### 9. Rebuild the CLI on the Same Application Layer

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

### `packages/application-contract`

The browser-safe workflow invocation surface shared by the UI and composition roots.

Responsibilities:

- define use-case input types
- define use-case result types
- define progress event types
- define shared error payload types needed by the UI abstraction
- define the narrow `WorkflowClient` interface that the UI calls

Rules:

- no Node APIs
- no Electron APIs
- no business logic
- every `WorkflowClient` method must correspond 1:1 to a `packages/application` use-case
- do not add query helpers, formatters, convenience methods, or direct host capability methods

### `packages/application`

Use-cases that coordinate domain logic with ports.

Responsibilities:

- implement use-cases whose public signatures are defined in `packages/application-contract`
- load/save profile workflows
- LMS import/sync orchestration
- CSV import/export orchestration
- repo create/clone/delete workflows
- settings loading and normalization
- shared error boundaries for typed `AppError` handling

Depends on abstract ports from `host-contract`, adapter interfaces from integration packages, and
workflow-facing types from `packages/application-contract`. Do not duplicate use-case signatures in
this package.

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

Progress event types and error types are defined in `packages/application-contract` and implemented
by `packages/application`. The tRPC router in `apps/desktop` and local `WorkflowClient` adapters
reuse these types automatically — `host-contract` does not need to define cross-boundary
serialization shapes for workflow progress or errors.

This replaces the huge generated `BackendAPI`.

Only define host ports that have concrete application consumers. Do not add speculative port
families before a real workflow needs them.

`TaskRunnerPort` is specifically for coarse-grained execution of infrastructure-level host batches
such as validated filesystem batches or process invocations. It must accept request shapes defined
in `host-contract`, stream typed progress, and return typed results. Application-specific workflow
plans stay in `packages/application`; `TaskRunnerPort` must not become a generic backdoor for
recreating the current backend-command layer under a different name.

### `packages/host-node`

Concrete Node implementations shared by desktop and CLI.

Responsibilities:

- disk access
- path operations
- Node-side HTTP execution
- child process or library-backed git execution
- secure host execution of long-running tasks

Keep application knowledge out of this package. It should implement ports, not business rules.
Electron-specific IPC glue stays in `apps/desktop` so `packages/host-node` remains reusable by the
CLI without Electron coupling.

Any Node-only repository execution code must stay in this package (or the `apps/desktop`
composition root), not in browser-importable packages such as `packages/application-contract` or
`packages/app`.

### `packages/app`

The React application. This is the evolution of `packages/app-core`, but it must depend on
`packages/application-contract`, not `packages/application`.

Responsibilities:

- components
- stores
- feature controllers
- host wiring through React context
- user-triggered workflows that call an injected `WorkflowClient`

Refactor goal:

- remove "backend command" thinking
- replace it with injected `WorkflowClient` calls for use-cases and direct domain function calls
  for pure logic
- never import `packages/application` implementations into browser bundles

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
- `packages/application-contract` may depend only on `packages/domain`, and only on
  `packages/host-contract` when a workflow payload truly needs a host-owned primitive
- `packages/host-contract` may depend only on `packages/domain`
- `packages/integrations-lms` and `packages/integrations-git` may depend only on
  `packages/domain` and `packages/host-contract`
- `packages/application` may depend only on `packages/application-contract`, `packages/domain`,
  `packages/host-contract`, `packages/integrations-lms`, and `packages/integrations-git`
- `packages/host-node` and `packages/host-browser-mock` may depend only on
  `packages/domain` and `packages/host-contract`
- `packages/app` may depend only on `packages/ui`, `packages/domain`,
  `packages/application-contract`, and `packages/host-contract`, but not `packages/application`
- `apps/desktop` composes `packages/app` with `packages/application`, `packages/host-node`, and a
  desktop `WorkflowClient` adapter backed by a tRPC router in Electron main and the
  `electron-trpc` IPC link in preload, both kept local to `apps/desktop`
- `apps/cli` composes `packages/application` with `packages/host-node`
- `apps/docs` composes `packages/app` with `packages/application`, `packages/host-browser-mock`,
  and a local in-browser `WorkflowClient` adapter

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
- tRPC router: one procedure per `packages/application` use-case, wired with real port
  implementations from `host-node`; long-running use-cases exposed as subscriptions
- security defaults and CSP enforcement
- no domain logic, no business rules, no forked workflow implementations

### Preload

Expose a minimal API to the renderer:

- do not expose raw `ipcRenderer`
- expose the `electron-trpc` IPC link so the renderer can create a tRPC client with types inferred
  from the main process router
- expose typed host capability calls for any direct port access the renderer still needs (dialogs,
  theme, window state)
- payload validation at the bridge boundary is handled by tRPC's transport; add runtime schemas
  only where additional defensive validation is justified

Security invariants:

- `contextIsolation: true`
- `nodeIntegration: false`
- do not use the deprecated `remote` module
- apply an explicit Content Security Policy for renderer content

### Renderer

The renderer owns UI state, component rendering, and direct invocation of pure domain logic from
`packages/domain`. It calls all workflows through an injected `WorkflowClient` from
`packages/application-contract`. In the desktop shell, that client is backed by a tRPC adapter
whose transport types are inferred from the main process router.

Renderer responsibilities:

- UI components and local UI state (tabs, dialogs, selections)
- Zustand stores with deterministic domain transforms (validation, normalization, group-set edits)
- `WorkflowClient` calls for use-cases (LMS sync, profile save/load, repo operations, import/export)
- receiving typed progress events from the active `WorkflowClient` implementation
- direct host calls only for renderer-local capabilities (dialogs, theme, window state)

The renderer must not perform direct third-party HTTP requests, filesystem I/O, or subprocess
execution. These are port-dependent operations that execute Node-side as part of tRPC procedures.

The renderer should remain testable in a plain browser-like environment. In tests, the
`WorkflowClient` can be replaced with a mock or a local adapter that calls use-cases directly
against mock ports, without requiring Node or IPC.

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

For this migration, keep LMS and Git tokens in the new app's persisted settings data as plain
text.

This is not a regression from the current shipped app behavior. Today, credentials are already
persisted in plain text in app-level settings, while profile files reference named connections
instead of storing duplicate secrets.

Rules:

- store credentials only inside the new app's persisted settings/profile data model, not in a
  separate secure store
- validate credential-bearing persisted data with the same boundary validation used for the rest of
  the settings/profile model
- do not add `CredentialPort` or any secure storage / keychain integration in this plan
- no migration of legacy secrets from the existing Tauri/keyring implementation

Secure credential storage should be added in a later hardening plan once the TypeScript
architecture is stable.

### Export Formats

User-visible exports that are part of app functionality should remain compatible unless there is a
clear quality reason to improve them without changing expected output semantics.

This includes:

- roster export
- assignment member export
- group set CSV import/export behavior
