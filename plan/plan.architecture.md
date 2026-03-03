# Repo Edu Tauri -> Electron Migration Plan: Architecture

This document contains the detailed target system design referenced by
[plan.md](./plan.md).

## Target Architecture

### Architectural Direction

Build the new `../repo-edu` as a pnpm TypeScript monorepo with a single
canonical domain model and no code generation layer between UI and native host
capabilities.

Use these boundaries:

1. Pure domain logic
2. Workflow invocation contract
3. Application orchestration
4. Host adapters
5. Delivery shells (Electron desktop, CLI, docs demo)

The main process must not become a second application backend. It should expose
only narrow host capabilities and long-running task execution primitives. All
business rules, validation, diffing, LMS behavior, group-set behavior, and
profile logic should live in shared TypeScript packages.

Network access to third-party APIs is the key exception to "renderer-first"
execution. Canvas, Moodle, GitHub, GitLab, and Gitea APIs cannot be called
directly from a browser renderer because their CORS policies are not designed
for arbitrary browser origins. External HTTP calls must execute in Node-owned
host adapters behind an explicit port, while orchestration and decision-making
remain in shared TypeScript use-cases.

### Execution Model

The placement rule for code execution in the desktop shell follows directly from
the package boundary:

- **`packages/domain`** functions run wherever the caller lives. In the desktop
  shell, the renderer imports and calls domain functions directly. They are
  pure, synchronous, and have no port dependencies.
- **`packages/application-contract`** defines browser-safe workflow invocation
  types: typed inputs, results, progress/error payloads, a canonical workflow
  definition map, and a narrow `WorkflowClient` interface. It contains no Node
  APIs, no Electron APIs, and no business logic.
- **`packages/application`** implements the shared use-cases. In the desktop
  shell, those implementations run Node-side behind the Electron main tRPC
  router. The renderer never imports them directly; it invokes an injected
  `WorkflowClient`, and the desktop shell satisfies that interface with an
  adapter that wraps the inferred tRPC client.

This rule is deterministic and requires no per-workflow analysis. The package a
function lives in determines where it executes, while `packages/app` sees only
`packages/domain` plus the browser-safe `packages/application-contract`
invocation surface. The plan's existing package responsibility definitions
already encode the classification: `packages/application` contains exactly the
use-cases that orchestrate ports, and `packages/domain` contains exactly the
pure logic.

This produces two explicit error channels that must remain distinct even when
one UI flow uses both:

- domain calls throw synchronous, deterministic errors handled locally by the
  component or store before or after workflow dispatch
- use-case calls surface typed `AppError` values through `WorkflowClient`; in
  desktop they cross IPC, and in docs/CLI they stay local

Per-shell execution:

- **Desktop renderer**: imports `packages/domain` and
  `packages/application-contract`; calls an injected `WorkflowClient` whose
  desktop adapter bridges to the tRPC client inferred from the Electron main
  router
- **CLI**: runs everything directly in Node, both domain and application, with
  no IPC dispatch
- **Docs demo**: runs everything in the browser, both domain and application,
  against mock port implementations from `host-browser-mock`, while
  `packages/app` still talks only to a local `WorkflowClient` adapter

The shared use-case code is identical across all three shells. Only the port
implementations and shell-specific `WorkflowClient` adapters differ. No use-case
contains environment-conditional behavior, and `packages/app` does not need
shell-specific branching.

In concrete terms, LMS and remote Git workflows execute in the desktop shell as:

- renderer calls a `WorkflowClient` method
- desktop `WorkflowClient` adapter opens a tRPC subscription on the Electron
  main router
- Electron main invokes the shared use-case from `packages/application` with
  real port implementations
- use-case calls integration adapter -> `HttpPort` -> Node-owned `fetch`
- the subscription yields a discriminated event union (`progress | completed |
  failed`); see [Subscription Event Protocol](#subscription-event-protocol)
  below
- the desktop adapter unwraps the stream: forwarding progress events to the
  caller, resolving with the result on `completed`, and rejecting with a typed
  `AppError` on `failed`

The Electron main process does not own any business logic. It acts as a runtime
host: it constructs port dependencies, invokes the shared use-case, and relays
results. The same use-case code runs directly (without hosting) in the CLI and
docs demo.

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
    ├── application-contract/   # Typed workflow invocation surface for the UI
    ├── application/            # Use-cases orchestrating domain + ports
    ├── renderer-host-contract/ # Renderer-safe direct host capability interfaces
    ├── host-runtime-contract/  # Application-side runtime port interfaces
    ├── host-node/              # Shared Node host adapters for desktop and CLI
    ├── host-browser-mock/      # Browser-safe mock runtime ports for docs/tests
    ├── integrations-lms/     # Canvas/Moodle TS clients
    └── integrations-git/     # Git and provider integrations in TS
```

This replaces the current split between `app-core`, generated bindings, Rust
core, and Rust CLI with one shared TypeScript architecture.

### Execution Defaults

Use these default implementation choices unless a concrete constraint forces a
change:

- `electron-vite` for Electron development/build wiring
- `electron-builder` for desktop packaging
- `electron-trpc` for type-safe IPC between renderer and main process
- built-in Node `fetch` inside `host-node` for HTTP clients
- `simple-git` for git CLI-backed repository operations
- `papaparse` for CSV parsing/serialization
- `xlsx` (SheetJS) for Excel import/export
- `commander` for the CLI command tree

These are implementation defaults, not architectural boundaries. They can change
without changing the package model described above.

Workspace defaults should also be fixed early:

- use TypeScript project references for package compilation boundaries
- define explicit `exports` maps for every package
- use conditional entrypoints only where Node vs browser delivery actually
  differs
- keep package resolution rules identical across Electron, CLI, tests, and docs
  builds

## Core Design Decisions

### 1. Remove Generated Bindings Entirely

The generated bindings layer should not be recreated.

Instead:

- define the domain model directly in TypeScript inside `packages/domain`
- keep hand-authored TypeScript types as the default source of truth for
  internal domain and application models
- use runtime schemas (for example Zod or Valibot) only at untrusted boundaries
  where validation is required
- share the same domain types everywhere: app, CLI, docs demo, tests, and host
  adapters

This removes:

- `packages/backend-interface`
- `packages/app-core/src/bindings/commands.ts`
- `apps/repo-manage/src/bindings/tauri.ts`
- all schema-driven Rust DTO generation
- the command manifest as an IPC contract definition

The new system should use hand-authored, explicit TypeScript module APIs, not
generated command bridges. IPC type safety across the Electron preload boundary
is enforced by tRPC: the main process defines a router over
`packages/application` use-cases, and the desktop transport types are inferred
from that router. Separately, `packages/application-contract` defines the
browser-safe `WorkflowClient` interface plus shared workflow payload types used
by `packages/app`. This replaces the current schema-to-bindings pipeline with
compiler-enforced consistency that requires no generation step.

Runtime schemas should be applied selectively, not universally. Use them for:

- settings/profile file loading
- import file decoding
- preload-to-renderer payload validation where needed
- external API response normalization when defensive validation is justified

Do not force every internal domain type to be declared through a runtime schema
if a plain TypeScript type is sufficient.

### 2. Move Business Logic to Shared TS Packages

Everything currently implemented in Rust that is not inherently host-specific
should move into shared TS packages:

- roster normalization
- system group set generation
- group import/export parsing
- LMS sync merge logic
- assignment validation
- glob / pattern matching
- slug and naming rules
- repository operation planning
- profile read/write validation

This logic belongs in `packages/domain` and `packages/application`, not in
Electron main.

The key rule is:

- decision-making in shared code
- side effects in adapters

### 3. Route External HTTP Through a Host Port

Do not let shared integrations call global `fetch` directly.

Instead:

- add an `HttpPort` (or `FetchPort`) to `packages/host-runtime-contract`
- require `packages/integrations-lms` and remote-provider portions of
  `packages/integrations-git` to depend on that port
- implement the real desktop/CLI version in `host-node` using Node `fetch`
- implement the docs/test version in `host-browser-mock` with canned or
  simulated responses

This keeps network code shared and testable while respecting renderer CORS
constraints.

In production Node contexts (`apps/desktop` main-side adapters and `apps/cli`),
`host-node` should provide the default `HttpPort` implementation backed by
native Node `fetch`. Consumers should depend on the abstraction, but production
wiring should not require repetitive custom plumbing.

The default architecture should be:

- use-cases run in shared TypeScript (`packages/application`)
- use-cases orchestrate domain logic and call ports
- ports own side effects such as HTTP, filesystem access, and process execution
- direct renderer-only host capabilities (dialogs, shell open, theme, window
  state) live in `packages/renderer-host-contract`, not in the runtime ports
- Electron preload exposes the tRPC IPC link and direct host capabilities, not
  app-specific backend commands

### 4. Make Progress Events a First-Class Contract

Long-running workflows must use one explicit progress model across renderer,
CLI, tests, and desktop IPC.

Default rule:

- `packages/application` owns workflow orchestration
- `packages/application-contract` owns workflow input/result/progress/error
  shapes plus the `WorkflowClient` interface
- `packages/application` implements those signatures
- use-cases accept a typed progress callback plus a standard `AbortSignal`, and
  yield typed progress events from `packages/application-contract` at the
  highest fidelity the underlying work can honestly support
- in the desktop shell, long-running use-cases are exposed as tRPC subscriptions
  that yield a discriminated event union; the desktop `WorkflowClient` adapter
  unwraps the stream into a promise-with-progress-callback shape whose
  caller-owned `AbortSignal` controls cancellation (see protocol below)
- in the CLI and docs demo, the same use-cases are called directly with a local
  progress callback and the same `AbortSignal`
- do not split one workflow across opaque main-process orchestration and
  renderer-local callbacks

Model progress in `packages/application-contract` and `packages/application`
use-case signatures, not as an ad hoc callback shape hidden inside individual
features. The tRPC router and local `WorkflowClient` adapters reuse these types
automatically — no separate IPC progress contract is needed.

Every long-running workflow definition in `packages/application-contract` must
also declare its execution capability profile so the UI and adapters know what
they can rely on before dispatch:

- progress granularity: `none`, `milestone`, or `granular`
- cancellation guarantee: `non-cancellable`, `best-effort`, or `cooperative`

Rules for those declarations:

- `none` progress is valid when the underlying operation cannot report useful
  intermediate state; the workflow may emit zero `progress` events and the UI
  must treat it as an indeterminate wait
- `milestone` progress is for coarse step boundaries only; adapters must not
  fabricate percentages or fake fine-grained sub-steps
- `granular` progress is reserved for workflows that can report meaningful
  incremental progress beyond step transitions
- `best-effort` cancellation means abort requests are advisory: the use-case
  still forwards the shared `AbortSignal`, but the underlying work may finish
  the current batch, subprocess, or library call before stopping
- `non-cancellable` means the use-case still accepts the shared call options for
  consistency, but once dispatched it may ignore abort until a terminal event;
  the UI must not present it as a guaranteed stop action
- `cooperative` means the use-case checks `signal.aborted` at explicit
  boundaries and all owned ports/integrations must propagate the same signal
  anywhere the underlying mechanism can actually stop work

#### Subscription Event Protocol

A tRPC subscription yields values of a single type — it has no built-in notion
of a "final return value" distinct from the streamed values. The plan must not
assume otherwise.

Every long-running tRPC subscription in the desktop shell yields a
**discriminated event union**:

```ts
type WorkflowEvent<TProgress, TResult> =
  | { type: "progress"; data: TProgress }
  | { type: "completed"; data: TResult }
  | { type: "failed"; error: AppError }
```

Protocol rules:

- The subscription emits zero or more `progress` events, followed by exactly one
  terminal event (`completed` or `failed`), then the observable completes.
- The tRPC error channel (observable error) is reserved for transport-level
  failures (connection lost, serialization error, unexpected main-process
  crash). Application-level errors use the `failed` event variant.
- `WorkflowEvent` is a generic defined once in `packages/application-contract`.
  Each workflow parameterizes it with its own `TProgress` and `TResult` types,
  which are also defined in `packages/application-contract`.

Cancellation rules:

- `packages/application-contract` defines one shared workflow call options shape
  (for example `{ onProgress?, signal? }`) used by `WorkflowClient`,
  `packages/application`, and all local adapters.
- Every long-running use-case in `packages/application` must accept that
  `signal`, check `signal.aborted` at explicit boundaries, and pass the same
  signal through to any port or integration that can block, stream, or spawn
  child work. If the underlying mechanism cannot provide prompt, reliable
  cancellation, the workflow must declare `best-effort` or `non-cancellable`
  semantics instead of pretending to be fully abortable.
- Desktop transport cancellation is a transport projection of the same contract:
  unsubscribing the tRPC subscription aborts the same underlying signal that the
  use-case and ports received.

The desktop `WorkflowClient` adapter unwraps this stream for each call:

1. Forwards `progress` events to the caller's progress callback.
2. On `completed`, resolves the returned `Promise<TResult>` with the result.
3. On `failed`, rejects the promise with the typed `AppError`.
4. On observable error (transport failure), rejects with a transport-level error
   mapped to `AppError`.
5. If the caller aborts the supplied `signal`, unsubscribes the tRPC
   subscription immediately and rejects with the shared cancellation-shaped
   `AppError`.

This keeps the `WorkflowClient` interface identical across shells — callers
always see a promise with a progress callback — while the desktop adapter
handles the stream-to-promise unwrapping internally. The CLI and docs adapters
skip the event union entirely because they call use-cases directly.

### 5. Use Promises With Typed Errors

Shared TypeScript APIs should use `Promise<T>` results and throw typed
`AppError` values instead of recreating Rust-style `Result<T, E>` wrappers.

Rules:

- `packages/application-contract` defines one shared workflow call signature,
  including a standard call options object that carries `onProgress` and
  optional `AbortSignal`
- `packages/application` use-cases return `Promise<T>`, implement the
  workflow-facing signatures defined in `packages/application-contract`, and
  must honor `signal.aborted`
- host ports return `Promise<T>` or progress-aware async abstractions that
  resolve/reject normally, and any port that can block, stream, or spawn work
  must accept and honor the same `AbortSignal`
- renderer, CLI, and tests handle failures through typed errors, not tagged
  union boilerplate
- application-level failures must use the shared `AppError` payload defined in
  `packages/application-contract`
- the tRPC observable error channel is reserved for transport failures only
- the desktop `WorkflowClient` adapter must normalize every transport failure
  through one explicit mapper (for example `toTransportAppError`) that converts
  transport faults into the same shared `AppError` surface expected by
  `packages/app`

### 6. Keep the Electron Backend Very Small

The Electron host should expose a narrow capability surface through
`contextBridge`:

- file open/save dialogs
- shell open for external URLs
- OS/theme/window primitives

Do not expose raw filesystem APIs, directory traversal helpers, or subprocess
access directly to the renderer. File reads/writes, path checks, and other
persistence work happen inside `packages/application` use-cases invoked through
the tRPC router, not through ad hoc preload helpers.

Beyond these direct host capabilities, the Electron main process hosts a tRPC
router. Each router procedure maps to a shared use-case from
`packages/application`, constructed with real port implementations from
`host-node`. Long-running use-cases are exposed as tRPC subscriptions that
stream a discriminated event union (see [Subscription Event
Protocol](#subscription-event-protocol)). The renderer reaches these procedures
only through the desktop `WorkflowClient` adapter defined by
`packages/application-contract`.

The Electron main process must not contain app-specific business logic like
group-set sync rules or assignment validation. Those are plain TypeScript
functions in shared packages that the router procedures invoke but do not own.

### 7. Use the Package Boundary as the Workflow Placement Rule

The placement of code execution in the desktop shell must be deterministic and
must not require per-workflow analysis. The rule is:

- **`packages/domain`**: runs wherever the caller lives. In the desktop
  renderer, domain functions are imported and called directly. They are pure,
  synchronous, and side-effect-free.
- **`packages/application-contract`**: runs wherever the caller lives. It
  defines the typed workflow invocation surface used by the UI, but contains no
  use-case implementation.
- **`packages/application`**: is the implementation layer. It runs Node-side for
  the desktop shell, and directly in the CLI and docs composition roots.
  `packages/app` must never import it.

This rule derives from the package responsibility definitions already in this
plan: `packages/application` is defined to contain exactly the use-cases that
orchestrate ports, and `packages/domain` is defined to contain exactly the pure
logic. If a use-case in `packages/application` turns out to be pure pass-through
with no port dependency, the plan already requires moving it down into
`packages/domain` (see Package Responsibilities). This keeps the classification
self-maintaining.

The rule produces two clean, non-overlapping error channels:

- **Domain errors**: synchronous, deterministic, caught locally by the calling
  component or store. No serialization, no IPC transport.
- **Use-case errors**: surface through `WorkflowClient` as typed `AppError`
  values. In desktop they cross IPC; in docs and CLI they stay local. The call
  shape in `packages/app` does not change.

A renderer flow may use both channels in sequence, but it must not blur them
together. Local domain validation and normalization errors are handled at the
renderer boundary; once that flow dispatches a use-case, failures surface only
as `AppError` values through `WorkflowClient`.

In the desktop shell, the transport behind `WorkflowClient` is the tRPC router
in `apps/desktop`. It provides:

- typed request identification: each procedure name and input type maps to a
  use-case
- typed event streaming: subscriptions yield a `WorkflowEvent<TProgress,
  TResult>` discriminated union with types inferred from the use-case (see
  [Subscription Event Protocol](#subscription-event-protocol))
- cancellation propagation: the router creates one `AbortController` per
  invocation, passes its `signal` into the use-case, and tRPC subscription
  unsubscribe aborts that controller

The router is part of the desktop shell's infrastructure in `apps/desktop`, not
a shared port. The CLI does not need it (it calls use-cases directly), and the
docs demo does not need it (its local adapter calls use-cases directly against
mock ports in the browser).

Contract consistency must be enforced, not merely reviewed.
`packages/application-contract` defines the canonical workflow definition map,
`WorkflowClient` is derived from that map, `packages/application` implements
that same keyed surface, and `apps/desktop` owns two explicit exhaustive
bindings over every workflow key: a main-side registry that maps keys to router
procedures and a renderer-side adapter binding that maps keys to inferred tRPC
calls. Missing keys, extra keys, or mismatched input/result/progress types in
either binding must fail compilation. No code generation is required; this is a
TypeScript type-level invariant. The invariant is compile-time, not a shared
cross-process runtime object.

### 8. Preserve a Browser-Safe Simulation Layer

The docs demo must remain a first-class delivery target.

To do that:

- the React app must depend on `packages/renderer-host-contract` and
  `packages/application-contract`, not Electron APIs, `packages/application`, or
  the runtime port contracts
- `host-browser-mock` must implement the runtime port contracts in memory
- the docs demo should mount the same app package with browser implementations
  of `packages/renderer-host-contract`, mock runtime ports, and a local
  `WorkflowClient` adapter that calls `packages/application` directly
- no Electron import may leak into packages consumed by `apps/docs`

This keeps the current "real UI + mock backend" capability, but with a smaller
and cleaner contract.

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

- if a function accepts only domain data, applies deterministic rules, and
  returns domain data, it belongs in `packages/domain`
- if a function depends on a port directly or transitively, coordinates side
  effects, or represents an end-to-end workflow, it belongs in
  `packages/application`
- if `packages/application` becomes thin pass-through glue for a behavior, move
  the pure logic down into `packages/domain` and keep only orchestration in
  `packages/application`

### `packages/domain`

Pure, deterministic, side-effect-free logic.

Responsibilities:

- canonical domain types and invariants
- boundary codecs/schemas only for domain data that enters from untrusted
  sources
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

The browser-safe workflow invocation surface shared by the UI and composition
roots.

Responsibilities:

- define the canonical workflow definition map keyed by workflow id
- define use-case input types
- define use-case result types
- define progress event types
- define shared error payload types needed by the UI abstraction
- define the narrow `WorkflowClient` interface that the UI calls
- define the shared workflow call options type that carries progress and
  cancellation

Rules:

- no Node APIs
- no Electron APIs
- no business logic
- define one canonical workflow definition map keyed by workflow id; derive
  `WorkflowClient` from that map instead of hand-authoring a second parallel
  surface
- every workflow key in that map must correspond 1:1 to a `packages/application`
  use-case
- keep workflow payloads browser-safe and serializable: flatten any host-shaped
  inputs/outputs into DTOs here instead of importing or re-exporting
  `packages/host-runtime-contract` primitives
- if a workflow needs host capabilities, model that dependency in
  `packages/application` through a host port, not by leaking host-owned types
  into `packages/application-contract`
- the same workflow key set must drive exhaustive main-side and renderer-side
  desktop bindings through mapped types so registration completeness is
  compile-time enforced on both sides
- do not add query helpers, formatters, convenience methods, or direct host
  capability methods

### `packages/application`

Use-cases that coordinate domain logic with ports.

Responsibilities:

- implement use-cases whose public signatures are defined in
  `packages/application-contract`
- load/save profile workflows
- LMS import/sync orchestration
- CSV import/export orchestration
- repo create/clone/delete workflows
- settings loading and normalization
- shared error boundaries for typed `AppError` handling

Depends on abstract ports from `packages/host-runtime-contract`, adapter
interfaces from integration packages, and workflow-facing types from
`packages/application-contract`. Do not duplicate use-case signatures in this
package.

### `packages/renderer-host-contract`

The renderer-safe direct host capability surface used by `packages/app`.

Responsibilities:

- direct user-triggered dialogs
- open-external shell actions
- theme, appearance, and window state primitives

Rules:

- browser-safe types only
- no filesystem traversal, subprocess execution, or arbitrary network access
- no workflow orchestration
- no business logic

This package exists specifically so the UI can use a tiny direct bridge without
gaining access to Node-owned runtime ports.

### `packages/host-runtime-contract`

The application-side runtime port surface implemented by host adapters.

Implemented by:

- Electron main composition
- CLI Node runtime
- docs/test mocks where browser-safe simulation is needed

Recommended contract families:

- `FileSystemPort`
- `HttpPort`
- `TaskRunnerPort`

Progress event types and error types are defined in
`packages/application-contract` and implemented by `packages/application`. The
tRPC router in `apps/desktop` and local `WorkflowClient` adapters reuse these
types automatically — `packages/host-runtime-contract` does not need to define
cross-boundary serialization shapes for workflow progress or errors.

This replaces the large generated backend surface for application-side effects.

Only define runtime ports that have concrete application consumers. Do not add
speculative port families before a real workflow needs them.

`TaskRunnerPort` is specifically for coarse-grained execution of
infrastructure-level host batches such as validated filesystem batches or
process invocations. It must accept request shapes defined in
`packages/host-runtime-contract`, surface typed progress only at the highest
fidelity the underlying mechanism can truthfully provide (including zero
progress events), accept the shared `AbortSignal`, and return typed results.
`TaskRunnerPort` implementations must explicitly document whether a task is
`cooperative`, `best-effort`, or `non-cancellable`; they must not fake
fine-grained progress or guaranteed abort for subprocesses or libraries that
cannot support it. Application-specific workflow plans stay in
`packages/application`; `TaskRunnerPort` must not become a generic backdoor for
recreating the current backend-command layer under a different name.

### `packages/host-node`

Concrete Node implementations shared by desktop and CLI.

Responsibilities:

- disk access
- path operations
- Node-side HTTP execution
- child process or library-backed git execution
- secure host execution of long-running tasks

Keep application knowledge out of this package. It should implement runtime
ports, not business rules. Electron-specific IPC glue stays in `apps/desktop` so
`packages/host-node` remains reusable by the CLI without Electron coupling.

Any Node-only repository execution code must stay in this package (or the
`apps/desktop` composition root), not in browser-importable packages such as
`packages/application-contract` or `packages/app`.

### `packages/app`

The React application. This is the evolution of `packages/app-core`, but it must
depend on `packages/application-contract`, not `packages/application`.

Responsibilities:

- components
- stores
- feature controllers
- host wiring through React context
- user-triggered workflows that call an injected `WorkflowClient`

Refactor goal:

- remove "backend command" thinking
- replace it with injected `WorkflowClient` calls for use-cases and direct
  domain function calls for pure logic
- never import `packages/application` implementations into browser bundles

### `packages/integrations-lms`

TypeScript LMS clients for Canvas and Moodle.

Responsibilities:

- API clients
- auth/token helpers
- response normalization into domain types
- dependency on `HttpPort`, not direct renderer `fetch`

These should expose thin external adapters. Merge and business interpretation
stay in `packages/application` / `packages/domain`.

### `packages/integrations-git`

TypeScript Git platform and repository adapters.

Responsibilities:

- platform verification
- repository provisioning integration
- remote provider API calls through `HttpPort`
- browser-safe normalization of remote provider payloads only

The target design should split:

- repository planning in shared application logic
- repository execution in host-backed adapters (`packages/host-node` or the
  desktop/CLI composition root), never in `packages/integrations-git`

## Package Dependency Rules

Enforce a strict one-way dependency graph:

- `packages/domain` depends on no other workspace package
- `packages/application-contract` may depend only on `packages/domain`
- `packages/renderer-host-contract` may depend only on `packages/domain`
- `packages/host-runtime-contract` may depend only on `packages/domain`
- `packages/integrations-lms` and `packages/integrations-git` may depend only on
  `packages/domain` and `packages/host-runtime-contract`
- `packages/application` may depend only on `packages/application-contract`,
  `packages/domain`, `packages/host-runtime-contract`,
  `packages/integrations-lms`, and `packages/integrations-git`
- `packages/host-node` and `packages/host-browser-mock` may depend only on
  `packages/domain` and `packages/host-runtime-contract`
- `packages/app` may depend only on `packages/ui`, `packages/domain`,
  `packages/application-contract`, and `packages/renderer-host-contract`, but
  not `packages/application` or `packages/host-runtime-contract`
- `apps/desktop` composes `packages/app` with a preload-backed
  `packages/renderer-host-contract` implementation, `packages/application`,
  `packages/host-node`, and a desktop `WorkflowClient` adapter backed by a tRPC
  router in Electron main and the `electron-trpc` IPC link in preload, both kept
  local to `apps/desktop`
- `apps/cli` composes `packages/application` with `packages/host-node`
- `apps/docs` composes `packages/app` with a browser implementation of
  `packages/renderer-host-contract`, `packages/application`,
  `packages/host-browser-mock`, and a local in-browser `WorkflowClient` adapter

Delivery shells are the composition roots. They construct port implementations,
inject those ports into integration clients, and inject the resulting clients
into application use-cases at startup. Shared packages must not reach for global
port singletons.

Do not allow delivery shells to become backchannels that bypass the shared
application layer.

## Desktop App Structure

### Electron Main

Keep `apps/desktop` intentionally small:

- window lifecycle
- preload registration
- updater/packaging integration
- host capability implementation wiring
- tRPC router: one procedure per `packages/application` use-case, wired with
  real port implementations from `host-node`; long-running use-cases exposed as
  subscriptions
- security defaults and CSP enforcement
- no domain logic, no business rules, no forked workflow implementations

### Preload

Expose a minimal API to the renderer:

- do not expose raw `ipcRenderer`
- expose the `electron-trpc` IPC link so the renderer can create a tRPC client
  with types inferred from the main process router
- expose typed calls from `packages/renderer-host-contract` for any direct
  renderer capability the UI still needs (dialogs, shell open, theme, window
  state)
- payload validation at the bridge boundary is handled by tRPC's transport; add
  runtime schemas only where additional defensive validation is justified

Security invariants:

- `contextIsolation: true`
- `nodeIntegration: false`
- do not use the deprecated `remote` module
- apply an explicit Content Security Policy for renderer content

### Renderer

The renderer owns UI state, component rendering, and direct invocation of pure
domain logic from `packages/domain`. It calls all workflows through an injected
`WorkflowClient` from `packages/application-contract`. In the desktop shell,
that client is backed by a tRPC adapter whose transport types are inferred from
the main process router.

Renderer responsibilities:

- UI components and local UI state (tabs, dialogs, selections)
- Zustand stores with deterministic domain transforms (validation,
  normalization, group-set edits)
- `WorkflowClient` calls for use-cases (LMS sync, profile save/load, repo
  operations, import/export)
- receiving typed progress events from the active `WorkflowClient`
  implementation
- direct calls only through `packages/renderer-host-contract` for renderer-local
  capabilities (dialogs, shell open, theme, window state)

The renderer must not perform direct third-party HTTP requests, filesystem I/O,
or subprocess execution. These are port-dependent operations that execute
Node-side as part of tRPC procedures.

The renderer should remain testable in a plain browser-like environment. In
tests, the `WorkflowClient` can be replaced with a mock or a local adapter that
calls use-cases directly against mock ports, without requiring Node or IPC.

## Persistence Strategy

### Settings and Profiles

Do not implement any migration from the existing Tauri settings or profile
storage.

Rules:

- define a new storage layout for the Electron project
- use explicit runtime validation on every load
- normalize invalid or partial data to defaults only within the new format
- if legacy files exist, do not auto-import, transform, or upgrade them

The new code may intentionally use different file locations, schemas, or
internal representation. Preserving intended behavior matters; backward
compatibility with old persisted files does not.

### Credentials

For this migration, keep LMS and Git tokens in the new app's persisted settings
data as plain text.

This is not a regression from the current shipped app behavior. Today,
credentials are already persisted in plain text in app-level settings, while
profile files reference named connections instead of storing duplicate secrets.

Rules:

- store credentials only inside the new app's persisted settings/profile data
  model, not in a separate secure store
- validate credential-bearing persisted data with the same boundary validation
  used for the rest of the settings/profile model
- do not add `CredentialPort` or any secure storage / keychain integration in
  this plan
- no import or migration of any legacy Tauri-stored credential values

Secure credential storage should be added in a later hardening plan once the
TypeScript architecture is stable.

### Export Formats

User-visible exports that are part of app functionality should remain compatible
unless there is a clear quality reason to improve them without changing expected
output semantics.

This includes:

- roster export
- assignment member export
- group set CSV import/export behavior
