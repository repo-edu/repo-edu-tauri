# Repo Edu Tauri -> Electron Migration Plan

This file is the canonical executive summary for the migration. Detailed specifications live in:

- [Architecture specification](./plan.architecture.md)
- [Migration scope mapping](./plan.migration-scope.md)
- [Delivery phases and testing](./plan.delivery.md)

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
- compiler-enforced IPC type safety via tRPC (no manual contract synchronization)

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
- The shipped app currently persists LMS and Git credentials in plain text in app-level settings,
  while profile files store references to named connections rather than duplicating secrets.

This means the right target is not "Electron + translated Rust command handlers". The right target
is "shared TypeScript domain/app services + a very thin Electron host".

## Architectural Direction Summary

Build `../repo-edu` as a pnpm TypeScript monorepo with one shared domain model and no generated
backend binding layer.

The target boundary model is:

1. Pure domain logic in shared TypeScript packages
2. Workflow invocation contracts in a browser-safe shared package
3. Application orchestration in shared TypeScript use-cases
4. Host adapters for side effects and platform access
5. Delivery shells for Electron desktop, CLI, and docs demo

Electron main must stay a small runtime host, not a second backend. It exposes a tRPC router that
maps each `packages/application` use-case to a typed procedure. The renderer does not import
`packages/application` directly: `packages/app` consumes a small hand-authored `WorkflowClient`
from `packages/application-contract`, and the desktop shell satisfies that interface with an
adapter that wraps the inferred tRPC client. This keeps IPC typing compiler-enforced via tRPC
while keeping the UI invocation surface explicit and browser-safe. Business rules, validation,
profile logic, diffing, LMS behavior, and repository planning belong in shared TypeScript packages
reused across desktop, CLI, tests, and docs.

Third-party HTTP is the main exception to renderer-local execution: external API calls must run in
Node-owned adapters behind explicit ports so the desktop app is not blocked by browser CORS
constraints.

## Explicit Non-Goals

- No legacy settings migration.
- No secure credential storage hardening in this migration plan; retaining plain-text credential
  storage in the new app preserves the current shipped behavior and is not a regression.
- Secure credential storage is deferred to a later hardening plan once the TypeScript architecture
  is stable.
- No attempt to preserve Rust module structure.
- No recreation of the generated bindings architecture.
- No hand-authored IPC-level serialization contracts. IPC request/response typing is inferred from
  the tRPC router. A small hand-authored UI invocation abstraction (`WorkflowClient`) is allowed
  and is not the IPC contract.
- No incremental hybrid Tauri/Electron bridge.
- No "temporary" business logic in Electron main that later needs to be moved out.

## Key Risks

- CORS is an architectural constraint, not an implementation detail. If external HTTP is allowed to
  leak into the renderer, the desktop app will fail against real LMS and Git provider APIs.
- `simple-git` still depends on the system Git CLI. The new Node implementation must explicitly
  handle missing Git installations, version expectations, and platform-specific process behavior.
- Electron packaging increases binary size and startup overhead compared with a browser-only or CLI
  target. Packaging and startup budgets should be measured early so the app does not accumulate
  avoidable platform weight.
- IPC type safety depends on the tRPC router in `apps/desktop` remaining the single source of truth
  for desktop transport, and on the desktop `WorkflowClient` adapter staying a thin 1:1 wrapper
  over that router. If any IPC path bypasses the router (raw `ipcRenderer.invoke`, ad hoc
  channels), contract drift reappears. The desktop shell must enforce that all desktop
  `packages/application` use-case dispatch goes through the tRPC router with no exceptions.

## Success Criteria

The migration is successful when all of the following are true:

1. `../repo-edu` is a pure TypeScript monorepo.
2. The desktop app is Electron-based and keeps Electron-specific logic minimal.
3. All current app and CLI functionality still exists with unchanged behavior.
4. The docs demo still runs the real UI against mock infrastructure.
5. There is no Rust, no Tauri, and no generated backend bindings.
6. The shared business logic is reusable across desktop, CLI, and docs without duplication.
