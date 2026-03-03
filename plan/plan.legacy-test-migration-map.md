# Repo Edu Tauri -> Electron Migration Plan: Legacy Test Migration Map

This document maps major legacy test areas to their intended destination layers
in the target TypeScript architecture.

Use it as the broad migration routing table for tests. The more detailed
keep/rewrite/remove decisions live in
[plan.test-triage-inventory.md](./plan.test-triage-inventory.md).

## Destination Map

| Legacy subsystem / test area | Current examples | Primary target layer(s) | Why this is the right destination |
| :--- | :--- | :--- | :--- |
| Roster invariants and transforms | `apps/repo-manage/core/src/roster/*.rs`, roster-related frontend selectors | `packages/domain` | These are pure product rules and should be testable without host adapters. |
| Roster UI behavior | `packages/app-core/src/components/tabs/roster/*.test.tsx`, roster store smoke tests | `packages/app` | UI state, rendering, and local interaction behavior belong in the app layer. |
| Group-set and assignment invariants | `apps/repo-manage/core/src/operations/group_set.rs`, `apps/repo-manage/core/src/import/tests.rs`, frontend group-set store tests | `packages/domain` plus `packages/application` | Parsing, matching, and invariant rules stay pure; file/LMS orchestration belongs in shared workflows. |
| LMS orchestration | `apps/repo-manage/core/src/operations/lms.rs`, `src-tauri/src/commands/lms.rs` | `packages/application` | The new app should test workflow semantics directly instead of transport wrappers. |
| LMS transport/client behavior | `crates/canvas-lms/*`, `crates/moodle-lms/*`, `crates/lms-client/*` | `packages/integrations-lms` | These are adapter concerns behind app-owned ports. |
| Settings and profile semantics | `apps/repo-manage/core/src/settings/*.rs`, persistence smoke tests | `packages/application` | The semantic rules for loading, validation, normalization, and saving live in shared use-cases. |
| Host file persistence details | File-writing and atomic persistence tests in Rust settings modules | `packages/host-node` contract tests | The host adapter owns filesystem mechanics in the new architecture. |
| Git platform/provider behavior | `apps/repo-manage/core/src/platform/github.rs`, `gitlab.rs`, `gitea.rs` | `packages/integrations-git` | Provider-specific request/response mapping should terminate at adapter boundaries. |
| Local Git command execution | `apps/repo-manage/core/src/platform/local.rs` | `packages/host-node` | The system Git CLI boundary becomes an explicit host concern. |
| Repository planning and execution | `apps/repo-manage/core/src/operations/repo.rs`, `OperationTab` | `packages/application`, `packages/host-node`, Electron E2E, `apps/cli` | This is the current weak spot and needs layered coverage: workflow semantics, host execution, desktop path, and CLI path. |
| Tauri command transport | `apps/repo-manage/src-tauri/src/commands/*.rs` | Mostly delete; replace with `packages/application` tests plus minimal `apps/desktop` bridge checks | The old transport layer should not be recreated as a primary test target. |
| Frontend store wiring and shell integration | `packages/app-core/src/hooks/*.test.ts`, `components/UtilityBar.test.tsx` | `packages/app` | These remain app concerns after the rewrite. |
| CLI surface and output | `apps/repo-manage/cli/tests/integration_tests.rs` | `apps/cli` | Command tree, help, and output are owned by the new CLI. |
| Docs standalone demo | `docs/src/components/DemoApp.tsx`, `docs/src/pages/demo-standalone.astro` | `apps/docs` smoke tests | The browser-safe simulation remains a delivery target and needs its own validation path. |

## Tests That Should Mostly Disappear

These legacy test shapes should not be carried forward as primary targets:

- generated binding shape tests
- Tauri-specific command transport tests beyond minimal replacement bridge
  coverage
- Rust-specific module-boundary tests that only reflect the old implementation
  split

They should be replaced by tests at the new architecture boundaries:

- `packages/domain`
- `packages/application`
- adapter contract tests
- Electron E2E for a narrow set of critical desktop flows
- `apps/cli` output/golden tests
- `apps/docs` smoke tests

## High-Priority New Coverage

The new architecture should add these target layers early because the current
repo leaves them weak or absent:

- `packages/application` workflow tests for repository operations
- `packages/host-node` tests for Git CLI execution and file persistence
- Electron E2E for at least one repository-operation flow
- `apps/docs` smoke tests for the standalone demo
