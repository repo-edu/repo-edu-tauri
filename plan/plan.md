# Repo Edu Tauri -> Electron Migration Plan

This file is the canonical executive summary for the migration. Detailed
specifications live in:

- [Architecture specification](./plan.architecture.md)
- [Delivery phases and testing](./plan.delivery.md)
- [Implementation checklist](./plan.implementation-checklist.md)
- [Test migration policy](./plan.test-migration.md)
- [Library replacement matrix](./plan.library-replacement-matrix.md)
- [Retained contract inventory](./plan.retained-contract-inventory.md)
- [Workflow mapping inventory](./plan.workflow-mapping.md)
- [Test triage inventory](./plan.test-triage-inventory.md)
- [Legacy test migration map](./plan.legacy-test-migration-map.md)

For actual implementation sequencing, interruption-safe resumption, and
task-by-task execution tracking, use
[plan.implementation-checklist.md](./plan.implementation-checklist.md) as the
canonical execution ledger. The other plan documents define scope, architecture,
and policy; the checklist defines the work order.

## Objective

Rebuild the current `repo-edu-tauri` application as a greenfield Electron
application in `../repo-edu` with preserved intended user-facing functionality,
higher long-term code quality, and a strictly smaller platform layer.

The migration is a redesign, not a port. Migration effort is intentionally
ignored. The target architecture should optimize for:

- clear ownership boundaries
- minimal Electron-specific code
- maximum TypeScript reuse across desktop, CLI, tests, and docs demo
- zero generated backend bindings
- zero Rust in the new project
- zero settings migration logic
- compiler-enforced IPC type safety via tRPC plus a shared compile-time workflow
  definition map with exhaustive per-process bindings

## Hard Requirements

- Preserve intended app behavior and feature scope unless a redesign is
  explicit.
- Do not preserve known defects or incidental implementation quirks only for
  migration symmetry.
- Maintain explicit automated test coverage for retained user-visible contracts
  throughout implementation, and defer user-based acceptance testing until the
  rewrite is fully complete.
- Replace the complete Rust backend with TypeScript.
- Apply a library-first migration policy for infrastructure and adapter
  concerns: before rewriting any Rust module by hand, evaluate whether a mature
  TypeScript library can replace the protocol, file-format, transport, or CLI
  plumbing with lower long-term maintenance burden.
- Keep product-specific domain behavior hand-authored in shared TypeScript
  packages; do not outsource roster semantics, group-set semantics, assignment
  validation, or repository planning to generic libraries.
- Replace the Rust CLI with a TypeScript CLI.
- Remove all generated backend bindings and the schema-to-bindings pipeline.
- Keep the desktop host limited to host concerns: direct host capabilities plus
  transport and composition for shared use-cases, primarily file I/O and
  process-native integration.
- Migrate the docs demo / mock simulation so it still runs outside the desktop
  shell.
- Do not add any migration path for legacy settings or profiles from Tauri.

## Current State Summary

The current codebase is already split in a way that makes a clean rewrite
possible:

- `packages/app-core` is the main React application and already contains most UI
  and state logic.
- `packages/backend-interface` defines a very large generated `BackendAPI`
  contract.
- `packages/backend-mock` provides an in-memory implementation used by tests and
  the docs demo.
- `apps/repo-manage/src` is a thin Tauri entrypoint.
- `apps/repo-manage/src-tauri` exposes Tauri commands over a large Rust backend.
- `apps/repo-manage/core` contains the shared Rust business logic currently
  reused by GUI and CLI.
- `apps/repo-manage/cli` is a Rust CLI with domain-based subcommands.
- `docs` mounts the same React app against the mock backend for simulation.
- The shipped app currently persists LMS and Git credentials in plain text in
  app-level settings, while profile files store references to named connections
  rather than duplicating secrets.

This means the right target is not "Electron + translated Rust command
handlers". The right target is "shared TypeScript domain/app services + a very
thin Electron host".

## Architectural Direction Summary

Build `../repo-edu` as a pnpm TypeScript monorepo with one shared domain model
and no generated backend binding layer.

The target boundary model is:

1. Pure domain logic in shared TypeScript packages
2. Workflow invocation contracts in a browser-safe shared package
3. Application orchestration in shared TypeScript use-cases
4. Host adapters for side effects and platform access
5. Delivery shells for Electron desktop, CLI, and docs demo

Electron main must stay a small runtime host, not a second backend. It exposes a
tRPC router that maps each `packages/application` use-case to a typed procedure.
The renderer does not import `packages/application` directly: `packages/app`
consumes a small `WorkflowClient` from `packages/application-contract`, and the
desktop shell satisfies that interface with an adapter that wraps the inferred
tRPC client. Both the main-side router bindings and the renderer-side adapter
bindings are derived independently from the same shared workflow definition map
in `packages/application-contract`; no single runtime registration object spans
both processes. This keeps IPC typing compiler-enforced via tRPC while keeping
the UI invocation surface explicit and browser-safe. Business rules, validation,
profile logic, diffing, LMS behavior, and repository planning belong in shared
TypeScript packages reused across desktop, CLI, tests, and docs.

Third-party HTTP is the main exception to renderer-local execution: external API
calls must run in Node-owned adapters behind explicit ports so the desktop app
is not blocked by browser CORS constraints.

## Library-First Migration Policy

The migration is not "translate all Rust into equivalent TypeScript modules". It
is:

1. Replace low-value infrastructure code with mature TypeScript libraries where
   the library owns protocol and transport details.
2. Rewrite product-specific business rules as explicit shared TypeScript
   modules.
3. Delete legacy generated bindings and command transport layers rather than
   recreating them.

This policy applies especially to:

- Git provider API clients
- CSV/XLSX file parsing and serialization
- CLI command parsing
- IPC transport typing
- boundary validation of untrusted files and payloads

This policy does not apply to:

- roster normalization
- system group set generation
- group-set import/reimport semantics
- assignment validation and selection rules
- repository planning and collision handling
- progress semantics and workflow orchestration

## Explicit Non-Goals

- No bug-for-bug parity with the existing Tauri/Rust implementation.
- No legacy settings migration.
- No secure credential storage hardening in this migration plan; the
  architecture should still leave a credential-storage seam in place, and the
  initial implementation should continue to use plain-text credential storage
  because that preserves the current shipped behavior and is not a regression.
- Secure credential storage is deferred to a later hardening plan once the
  TypeScript architecture is stable.
- No attempt to preserve Rust module structure.
- No recreation of the generated bindings architecture.
- No hand-authored IPC-level serialization contracts. IPC request/response
  typing is inferred from the tRPC router. A small UI invocation abstraction
  (`WorkflowClient`) is allowed and is not the IPC contract.
- No incremental hybrid Tauri/Electron bridge.
- No "temporary" business logic in Electron main that later needs to be moved
  out.

## Key Risks

- CORS is an architectural constraint, not an implementation detail. If external
  HTTP is allowed to leak into the renderer, the desktop app will fail against
  real LMS and Git provider APIs.
- Rewriting mature infrastructure concerns by hand in TypeScript would recreate
  the current maintenance burden in a new language. The migration must
  aggressively replace low-value protocol and file-format code with maintained
  libraries where those libraries are strong.
- Git execution should not sit behind an ambiguous `simple-git` convenience
  layer. The plan commits to one explicit adapter over the system Git CLI using
  `child_process.spawn`, and that adapter must explicitly handle missing Git
  installations, version expectations, and platform-specific process behavior.
- Future in-app `gitinspectorgui` integration will require substantially richer
  local Git inspection capabilities than the current repo workflows, including
  follow-aware history and blame queries such as `git log --follow` and `git
  blame --follow`. The chosen Git boundary must be broad enough to absorb that
  later expansion without introducing a second local Git execution stack.
- Electron packaging increases binary size and startup overhead compared with a
  browser-only or CLI target. Packaging and startup budgets should be measured
  early so the app does not accumulate avoidable platform weight.
- Desktop distribution is more than choosing a packager. Code signing, macOS
  notarization, per-platform installer formats, updater feed topology, release
  channels, and the decision to support or defer delta updates will still need
  explicit follow-up decisions later, but they are intentionally out of scope
  for this migration.
- IPC type safety depends on the shared workflow definition map in
  `packages/application-contract` remaining the compile-time source of truth,
  and on both the main-side router bindings and the desktop `WorkflowClient`
  adapter staying thin 1:1 projections of that map. If any IPC path bypasses the
  router (raw `ipcRenderer.invoke`, ad hoc channels), contract drift reappears.
  The desktop shell must enforce that all desktop `packages/application`
  use-case dispatch goes through the tRPC router with no exceptions.

## Success Criteria

The migration is successful when all of the following are true:

1. `../repo-edu` is a pure TypeScript monorepo.
2. The desktop app is Electron-based and keeps Electron-specific logic minimal.
3. All current app and CLI functionality still exists with preserved intended
   behavior, except where an intentional redesign is documented.
4. The docs demo still runs the real UI against mock infrastructure.
5. There is no Rust, no Tauri, and no generated backend bindings.
6. The shared business logic is reusable across desktop, CLI, and docs without
   duplication.
7. The desktop shell is locally runnable as an Electron app with the intended
   retained functionality, while release/distribution concerns remain explicitly
   deferred.
