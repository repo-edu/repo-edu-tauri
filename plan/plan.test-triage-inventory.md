# Repo Edu Tauri -> Electron Migration Plan: Test Triage Inventory

This document is the lean, area-level test triage inventory referenced by
[plan.delivery.md](./plan.delivery.md) and
[plan.test-migration.md](./plan.test-migration.md).

It is intentionally:

- grouped by major legacy test area, not by every file
- focused on migration decisions
- explicit about confidence where current behavior is inferred more from code
  structure than test evidence

Decision meanings:

- `keep and rewrite`: preserve the behavioral guarantee, but move the tests to
  the new architecture
- `supersede`: replace the old test shape with a different test layer that owns
  the same risk better
- `remove`: do not carry the old test shape forward
- `add`: there is a major current gap and the new project must introduce tests

## Area-Level Triage

| Legacy test area | Current locations | What is covered today | Confidence signal | Migration decision | Target in new architecture |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Roster UI and local roster state | `packages/app-core/src/components/tabs/roster/*.test.tsx`, `packages/app-core/src/hooks/useDirtyState.test.ts`, parts of `stores/__tests__/smoke.test.ts` and `selectors.test.ts` | Sorting, dirty-state transitions, roster mutations, local state invariants | `high` | `keep and rewrite` | `packages/app` tests plus shared `packages/domain` tests for pure transforms |
| Group sets and assignments UI/state | `packages/app-core/src/components/tabs/groups-assignments/*.test.tsx`, `packages/app-core/src/stores/__tests__/actions.test.ts`, `selectors.test.ts`, `utils/__tests__/groupNaming.test.ts`, `utils/__tests__/issues.test.ts` | Group-set CRUD, group reference semantics, assignment selection, group naming, issues derivation | `high` | `keep and rewrite` | Split across `packages/app`, `packages/domain`, and `packages/application` where workflows cross boundaries |
| Settings/profile UI helpers | `packages/app-core/src/hooks/useAppSettings.test.ts`, `packages/app-core/src/adapters/settingsAdapter.test.ts`, parts of `smoke.test.ts` | Store wiring, settings adapter round-trips, defaults | `medium` | `keep and rewrite` | `packages/app` tests for UI wiring, `packages/application` tests for settings/persistence behavior |
| Generic frontend utilities | `packages/app-core/src/utils/sorting.test.ts`, `snapshot.test.ts`, `components/ActionBar.test.tsx`, `components/UtilityBar.test.tsx` | Local utility behavior and basic component rendering | `medium` | `keep and rewrite` | `packages/app` for UI helpers; `packages/domain` for pure utility logic where it remains product-relevant |
| Operations tab UI | No `OperationTab` tests; no frontend tests covering preflight or repo execution flows | Essentially untested in the UI layer | `low` | `add` | `packages/app` workflow tests plus Electron E2E for core desktop operation flows |
| Roster domain logic in Rust | `apps/repo-manage/core/src/roster/*.rs` tests, `roster/tests.rs`, `roster/system.rs`, `roster/naming.rs`, `roster/glob.rs` | Naming, system group sets, glob matching, core roster invariants | `high` | `keep and rewrite` | `packages/domain` |
| Group-set import/export and sync logic in Rust | `apps/repo-manage/core/src/operations/group_set.rs`, `apps/repo-manage/core/src/import/tests.rs` | CSV parsing, import/export semantics, duplicate detection, sync-related logic | `high` | `keep and rewrite` | `packages/domain` for invariants and parsing rules; `packages/application` for workflow orchestration |
| LMS orchestration in Rust | `apps/repo-manage/core/src/operations/lms.rs`, parts of `src-tauri/src/commands/lms.rs` | LMS merge behavior, member mapping, some command-shaping behavior | `high` | `keep and rewrite` | `packages/application` plus `packages/integrations-lms` adapter tests |
| Settings and persistence in Rust | `apps/repo-manage/core/src/settings/*.rs`, `apps/repo-manage/src-tauri/src/tests/smoke_persistence.rs` | Validation, normalization, merge rules, persistence edge cases | `high` | `keep and rewrite` | `packages/application` for semantics plus `host-node` contract tests for file I/O |
| Git provider clients and local Git adapters in Rust | `apps/repo-manage/core/src/platform/*.rs` | Provider request/response handling, local Git and platform behaviors | `medium` | `supersede` | `packages/integrations-git` adapter tests and `host-node` Git execution tests; SDK-backed adapters reduce need for line-by-line test parity |
| Repository operations in Rust | `apps/repo-manage/core/src/operations/repo.rs` | No tests found | `low` | `add` | `packages/application` workflow tests, `host-node` adapter tests, Electron E2E, and CLI output tests |
| Tauri command handlers and utilities | `apps/repo-manage/src-tauri/src/commands/*.rs`, `src-tauri/src/commands/utils.rs` | Transport shaping, command-level wrappers, some handler behavior | `medium` | `supersede` | Mostly `packages/application` workflow tests; retain only minimal desktop-shell integration checks around the new Electron bridge |
| CLI integration tests | `apps/repo-manage/cli/tests/integration_tests.rs` | Help text, subcommand presence, required-argument enforcement | `medium` | `keep and rewrite` | `apps/cli` output/golden tests; widen coverage beyond help-only assertions |
| Docs demo | No dedicated docs smoke tests found in the current repo | Behavior inferred from wiring, not automated verification | `low` | `add` | `apps/docs` smoke tests that mount the app against browser-safe mocks |

## Explicit Gaps To Fill Early

These are the most important current gaps that should become early migration
tests instead of being deferred:

- repository operation workflows (`repo.create`, `repo.clone`, `repo.delete`)
- repository-operation preflight behavior, if retained
- docs standalone demo smoke coverage
- end-to-end desktop coverage for at least one core workflow path

## Confidence Note

`Roster` and `Group sets & Assignments` are the strongest areas to derive from
because they are backed by meaningful existing tests at both the frontend and
domain levels.

`Operations` is intentionally treated as a code-derived, low-confidence area:

- no current `OperationTab` tests
- no current `core/src/operations/repo.rs` tests
- migrate the capability, but do not treat every current detail as a preserved
  guarantee without review
