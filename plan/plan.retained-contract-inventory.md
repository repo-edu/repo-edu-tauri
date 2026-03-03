# Repo Edu Tauri -> Electron Migration Plan: Retained Contract Inventory

This document is the lean retained-contract inventory referenced by
[plan.delivery.md](./plan.delivery.md).

It is intentionally:

- AI-derived from the current codebase as a first draft
- organized by major feature area, not exhaustively
- explicit about confidence, so weakly evidenced areas do not get treated as
  hard contracts by accident

This inventory is the coarse feature-area baseline, not the final per-workflow
or per-surface gate for phase 2. A single row here may decompose into multiple
workflow-backed contracts with narrower delivery targets in
[plan.workflow-mapping.md](./plan.workflow-mapping.md). Where this document and
the workflow map differ in granularity, use the workflow map as the
authoritative phase-2 artifact for shared workflow ids and invoking delivery
surfaces.

Confidence meanings:

- `high`: current behavior is backed by meaningful existing tests plus clear
  code paths
- `medium`: current behavior is clear from code and partially tested, but not
  comprehensively
- `low`: current behavior is mostly code-derived and should not be treated as a
  hard retained contract until explicitly reviewed

## Inventory

| Feature area | Current retained contract (draft) | Current surfaces | Confidence | Evidence in current repo | Draft migration intent |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Profile lifecycle | Users should be able to manage profiles end-to-end: list, create, activate, inspect/load, save, rename, and delete. The current CLI only partially exposes this today and should not be treated as the full source of truth for target CLI scope. | Desktop UI, CLI (partial today; broader target intended), docs demo | `medium` | `packages/app-core/src/hooks/useProfiles.ts`, `packages/app-core/src/components/dialogs/NewProfileDialog.tsx`, `apps/repo-manage/cli/src/main.rs` (partial profile surface only), `apps/repo-manage/schemas/commands/manifest.json` | Retain as shared profile workflows. The target CLI may expose a broader profile-management surface than the current implementation. |
| App settings and connection management | App-level settings can be loaded/saved; LMS and Git connections can be verified before being persisted. | Desktop UI, docs demo | `medium` | `packages/app-core/src/services/settingsService.ts`, `packages/app-core/src/components/settings/ConnectionsPane.tsx`, `packages/app-core/src/hooks/useAppSettings.test.ts`, `packages/app-core/src/adapters/settingsAdapter.test.ts` | Retain as shared settings and connection-verification workflows. |
| Roster import from LMS | Users can sync roster data from LMS into the current roster, merging students and staff rather than replacing blindly. | Desktop UI, CLI, docs demo | `high` | `packages/app-core/src/components/dialogs/RosterSyncDialog.tsx`, `apps/repo-manage/core/src/operations/lms.rs`, `apps/repo-manage/cli/src/commands/lms.rs`, `packages/backend-mock/src/index.ts` | Retain as a primary shared workflow. |
| Roster import from file | Users can import students from CSV/XLSX into the current roster. | Desktop UI, docs demo | `medium` | `packages/app-core/src/components/dialogs/ImportStudentsFromFileDialog.tsx`, `apps/repo-manage/schemas/commands/manifest.json` | Retain as a shared workflow with boundary validation. |
| Roster editing and clearing | Users can clear roster state and edit roster members locally without losing undo/redo semantics. | Desktop UI, docs demo | `high` | `packages/app-core/src/components/tabs/RosterTab.tsx`, `packages/app-core/src/stores/__tests__/smoke.test.ts`, `packages/app-core/src/hooks/useDirtyState.test.ts` | Retain as local deterministic app/domain behavior, not a host workflow by default. |
| Student export | Users can export students to CSV or XLSX. | Desktop UI, docs demo | `medium` | `packages/app-core/src/components/tabs/RosterTab.tsx`, `apps/repo-manage/schemas/commands/manifest.json` | Retain as a shared export workflow. |
| System group-set normalization | The app can create or repair system group sets and keep their memberships aligned with the roster. | Desktop UI, docs demo | `high` | `packages/app-core/src/stores/profileStore.ts`, `apps/repo-manage/core/src/roster/system.rs`, `apps/repo-manage/schemas/commands/manifest.json` | Retain as explicit shared domain/application behavior. |
| Local group-set and group editing | Users can create, copy, rename, and delete local group sets; add, edit, and remove local groups; and preserve reference semantics across group sets. | Desktop UI, docs demo | `high` | `packages/app-core/src/stores/__tests__/actions.test.ts`, `packages/app-core/src/stores/__tests__/selectors.test.ts`, `packages/app-core/src/components/tabs/groups-assignments/*.tsx` | Retain as local deterministic app/domain behavior. |
| LMS-linked group-set sync | Users can connect and sync LMS-backed group sets and update group memberships in place. | Desktop UI, CLI (cache-related group-set flows), docs demo | `high` | `packages/app-core/src/components/dialogs/ConnectLmsGroupSetDialog.tsx`, `packages/app-core/src/components/tabs/groups-assignments/GroupSetPanel.tsx`, `apps/repo-manage/core/src/operations/group_set.rs`, `apps/repo-manage/cli/src/main.rs` | Retain as shared LMS/group-set workflows. |
| Group-set import/export | Users can preview, import, reimport, and export group sets; export editable group data; and export assignment/team member views. | Desktop UI, docs demo | `high` | `packages/app-core/src/components/dialogs/ImportGroupSetDialog.tsx`, `packages/app-core/src/components/dialogs/ReimportGroupSetDialog.tsx`, `packages/app-core/src/components/sheets/FileImportExportSheet.tsx`, `apps/repo-manage/core/src/operations/group_set.rs` | Retain as shared file workflows backed by domain invariants. |
| Assignment editing and selection rules | Users can create/delete assignments, attach them to group sets, and preview group-selection results from `all` or pattern-based selection. | Desktop UI, docs demo | `high` | `packages/app-core/src/stores/__tests__/actions.test.ts`, `packages/app-core/src/stores/__tests__/selectors.test.ts`, `packages/app-core/src/components/dialogs/NewAssignmentDialog.tsx`, `apps/repo-manage/schemas/commands/manifest.json` | Retain; keep local deterministic selection logic in shared domain/app code. |
| Group naming and pattern filtering | The app normalizes group names, validates patterns, and previews matches before users create or filter groups. | Desktop UI, docs demo | `high` | `packages/app-core/src/components/dialogs/AddGroupDialog.tsx`, `packages/app-core/src/components/dialogs/NewLocalGroupSetDialog.tsx`, `packages/app-core/src/utils/__tests__/groupNaming.test.ts`, `apps/repo-manage/schemas/commands/manifest.json` | Retain as explicit shared domain behavior with boundary validation where needed. |
| Git usernames and validation | Users can import Git usernames, verify username coverage, run roster validation, and run assignment validation that feeds issue reporting. | Desktop UI, CLI, docs demo | `medium` | `packages/app-core/src/components/dialogs/ImportGitUsernamesDialog.tsx`, `packages/app-core/src/components/dialogs/UsernameVerificationDialog.tsx`, `packages/app-core/src/stores/profileStore.ts`, `packages/app-core/src/utils/__tests__/issues.test.ts`, `apps/repo-manage/cli/src/main.rs` | Retain as shared validation and import workflows. |
| Repository operations (Operations tab) | Users can choose an assignment and run repository create/clone/delete operations; code also defines preflight concepts, status, and results. | Desktop UI, CLI, docs demo | `low` | `packages/app-core/src/components/tabs/OperationTab.tsx`, `packages/app-core/src/components/dialogs/PreflightDialog.tsx`, `packages/app-core/src/stores/operationStore.ts`, `apps/repo-manage/cli/src/main.rs`, `apps/repo-manage/schemas/commands/manifest.json` | Retain the high-level create/clone/delete capability. Treat preflight dialog behavior and fine-grained operation semantics as code-derived and require an explicit code-derived scope review plus contract freeze before implementation turns them into hard contracts. |
| CLI command surface | The target CLI should retain the major command families (`profile`, `roster`, `lms`, `lms cache`, `git`, `repo`, `validate`) as the intended capability model. The current implementation is only a partial, lightly validated planning input and should not freeze exact subcommand parity. | CLI | `low` | `apps/repo-manage/cli/src/main.rs`, `apps/repo-manage/cli/tests/integration_tests.rs` (mostly command-tree/help coverage) | Retain the command families and core user-visible intent, but treat the current command tree as incomplete and review-gated rather than as a strict compatibility contract. |
| Docs standalone demo | The docs site mounts the real app against a mock backend and should continue to exercise the same major UI flows without a desktop shell. | Docs demo | `medium` | `docs/src/components/DemoApp.tsx`, `docs/src/pages/demo-standalone.astro`, `packages/backend-mock/src/index.ts` | Retain as a first-class delivery target. |

## Explicit Low-Confidence Areas

`Operations` and the CLI command surface are the two major areas in this
inventory that are intentionally marked `low` confidence.

### `Operations`

Reason:

- there are no current `packages/app-core` tests covering `OperationTab`
- there are no tests in `apps/repo-manage/core/src/operations/repo.rs`
- the code shows the capability surface, but not all of its semantics are
  clearly validated by tests

Rule for migration planning:

- retain the existence of repository create/clone/delete workflows
- do not assume every current preflight, collision, status, or UX detail is a
  must-preserve contract unless it is explicitly reviewed through a code-derived
  scope pass and then promoted

### CLI command surface

Reason:

- current confidence comes primarily from the present command tree plus a small
  set of command/help tests
- the current CLI has not been treated as a well-verified source of exact
  behavior and may be missing intended management flows
- top-level command families are a stronger signal than the current
  subcommand-by-subcommand implementation details

Rule for migration planning:

- retain the major CLI capability families as the intended product surface
- do not assume the current CLI's exact subcommand set, wiring, or formatting is
  complete or must be preserved without explicit review
