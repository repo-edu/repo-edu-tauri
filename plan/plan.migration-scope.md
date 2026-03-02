# Repo Edu Tauri -> Electron Migration Plan: Migration Scope

This document maps the current system's functional areas into the target design described in
[plan.md](./plan.md).

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

- stores and feature controllers may invoke `WorkflowClient` methods, but must not import
  `packages/application` implementations or recreate a generated command facade
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
- docs provides a local browser-safe `WorkflowClient` adapter that calls `packages/application`
  directly against mock ports
- mock data and simulated workflows remain separate from production adapters
- the standalone demo page remains fully functional without Electron
