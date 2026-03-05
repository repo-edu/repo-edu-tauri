# Repo Edu Tauri -> Electron Migration Plan: Workflow Mapping Inventory

This document maps retained workflow-backed contracts to draft shared workflow
ids for the target TypeScript architecture.

It is intentionally a planning map, not a generated API:

- workflow ids below are proposed names for `packages/application-contract`
- pure local deterministic behavior can remain outside this map
- direct renderer host capabilities (for example dialogs or shell-open) are not
  modeled as application workflows here
- this is the authoritative phase-2 artifact for per-workflow and per-surface
  mapping where [plan.delivery.md](./plan.delivery.md) requires retained
  contracts to be tied to shared workflow ids and delivery targets

## Mapping Rules

- If a behavior crosses a host or integration boundary, it should have a shared
  workflow id.
- If a behavior is purely local deterministic UI/domain state, keep it in shared
  domain/app code instead of inventing a host workflow.
- Low-confidence current behavior is mapped conservatively: preserve the major
  capability, but do not freeze every incidental detail as part of the draft
  workflow contract.
- A single retained-contract inventory row may expand into multiple workflow
  rows here when the per-workflow delivery targets differ.
- For the CLI specifically, treat the current command tree as a partial planning
  input. The target workflow map may intentionally cover intended CLI
  capabilities that are missing or incomplete in today's implementation.

## Draft Workflow Map

| Draft workflow id | Retained contract area | Current command(s) / surface | Delivery targets | Confidence | Notes |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `profile.list` | List profiles | `list_profiles` | desktop, docs, CLI | `medium` | Shared profile discovery surface. |
| `profile.activate` | Set active profile | `set_active_profile`, CLI `profile load` | desktop, docs, CLI | `medium` | Keep top-level intent even if UI wiring changes. |
| `profile.load` | Load profile settings and roster | `load_profile`, `get_roster`, `load_profile_settings` | desktop, docs, CLI | `medium` | Current desktop splits this across commands; target may unify under one workflow-backed load path. CLI `profile show` and `roster show` can remain shell-local renderings over this loaded data rather than separate shared workflows. |
| `profile.create` | Create profile | `create_profile` | desktop, docs, CLI | `medium` | Keep current creation capability. The target CLI may add this even though the current CLI does not expose it yet. |
| `profile.save` | Save profile settings and roster | `save_profile`, `save_profile_and_roster` | desktop, docs, CLI | `medium` | Current desktop has split save commands; target may normalize the write surface. CLI coverage here represents intended future profile-management support, not a claim of current parity. |
| `profile.rename` | Rename profile | `rename_profile` | desktop, docs, CLI | `medium` | Shared profile-management workflow. The target CLI may expose this later. |
| `profile.delete` | Delete profile | `delete_profile` | desktop, docs, CLI | `medium` | Shared profile-management workflow. The target CLI may expose this later. |
| `settings.loadApp` | Load app settings | `load_app_settings` | desktop, docs | `medium` | Keep app-level settings load behavior; CLI settings access remains an internal dependency, not a retained user-facing workflow. |
| `settings.saveApp` | Save app settings | `save_app_settings` | desktop, docs | `medium` | Keep app-level settings persistence; CLI settings writes remain an internal dependency, not a retained user-facing workflow. |
| `connection.verifyLmsDraft` | Verify draft LMS connection | `verify_lms_connection_draft` | desktop, docs | `medium` | Draft-only connection verification before save. |
| `connection.verifyGitDraft` | Verify draft Git connection | `verify_git_connection_draft` | desktop, docs | `medium` | Draft-only Git verification before save. |
| `course.verifyProfileLms` | Verify active profile LMS course | `verify_profile_course`, `verify_lms_connection` | desktop, docs, CLI | `medium` | May remain split internally; user-facing contract is verification. |
| `git.verifyProfileConnection` | Verify the saved Git platform connection for the active or selected profile | `verify_git_connection`, CLI `git verify` | desktop, docs, CLI | `low` | Distinct from draft verification. CLI intent is retained, but the current CLI surface is a low-confidence signal and may be incomplete. |
| `lms.fetchCourses` | Fetch available LMS courses | `fetch_lms_courses`, `fetch_lms_courses_draft` | desktop, docs | `medium` | Profile creation flow. |
| `roster.importFromLms` | Import or sync roster from LMS | `import_roster_from_lms`, CLI `lms import-students` | desktop, docs, CLI | `high` | Core retained workflow. |
| `roster.importFromFile` | Import students from CSV/XLSX | `import_students_from_file` | desktop, docs | `medium` | File-backed workflow. |
| `roster.exportStudents` | Export students to CSV/XLSX | `export_students` | desktop, docs | `medium` | Keep user-visible export capability. |
| `roster.ensureSystemGroupSets` | Create or repair system group sets | `ensure_system_group_sets` | desktop, docs | `high` | May be invoked implicitly by app flows, but still deserves a named shared use-case. |
| `groupSet.fetchAvailableFromLms` | List LMS group sets | `fetch_lms_group_set_list` | desktop, docs, CLI | `high` | Used before attach/sync. |
| `groupSet.syncFromLms` | Fetch and sync a selected LMS group set | `sync_group_set`, `fetch_lms_groups_for_set` | desktop, docs, CLI | `high` | Preserve in-place sync intent. |
| `groupSet.previewImportFromFile` | Preview a group-set import from file | `preview_import_group_set` | desktop, docs | `high` | File parsing and validation cross a host boundary, so the preview stays a shared workflow. Applying a validated preview to local state is deterministic app/domain behavior, not a separate shared workflow id. |
| `groupSet.previewReimportFromFile` | Preview a group-set reimport into an existing group set | `preview_reimport_group_set` | desktop, docs | `high` | File parsing and validation cross a host boundary, so the preview stays a shared workflow. Applying a validated reimport preview to local state is deterministic app/domain behavior, not a separate shared workflow id. |
| `groupSet.export` | Export a group set | `export_group_set` | desktop, docs | `high` | Retain user-visible export behavior. |
| `groupSet.exportEditable` | Export group data for editing | `export_groups_for_edit` | desktop, docs | `high` | Keep explicit editable export surface. |
| `groupSet.cache.list` | List cached LMS group sets | CLI `lms cache list` | CLI | `medium` | CLI-only current surface; retain unless deliberately collapsed into a broader group-set workflow model. |
| `groupSet.cache.link` | Link/fetch a cached LMS group set | CLI `lms cache fetch` | CLI | `medium` | CLI-only current surface. |
| `groupSet.cache.refresh` | Refresh a cached group set | CLI `lms cache refresh` | CLI | `medium` | CLI-only current surface. |
| `groupSet.cache.delete` | Delete a cached group set | CLI `lms cache delete` | CLI | `medium` | CLI-only current surface. |
| `assignment.exportStudents` | Export assignment members | `export_assignment_students` | desktop, docs | `high` | Keep assignment-specific export capability. |
| `teams.export` | Export team views | `export_teams` | desktop, docs | `high` | Keep current team export capability. |
| `gitUsernames.import` | Import Git usernames | `import_git_usernames` | desktop, docs | `medium` | Shared file/data workflow. |
| `gitUsernames.verify` | Verify Git username status | `verify_git_usernames` | desktop, docs | `medium` | Shared validation workflow. |
| `validation.roster` | Validate roster-level issues | `validate_roster` | desktop, docs | `medium` | Keep as explicit validation workflow. |
| `validation.assignment` | Validate assignment readiness | `validate_assignment`, CLI `validate` | desktop, docs, CLI | `medium` | Shared validation workflow with CLI surface. |
| `repo.create` | Create repositories | `create_repos`, CLI `repo create` | desktop, docs, CLI | `low` | Preserve the batch-create capability. The explicit phase-2 scope review below freezes the retained contract at assignment/profile-scoped batch creation plus aggregate user-visible results; preflight UX remains delivery-specific. |
| `repo.clone` | Clone repositories | `clone_repos_from_roster`, CLI `repo clone` | desktop, docs, CLI | `low` | Preserve the batch-clone capability. The explicit phase-2 scope review below freezes the retained contract at assignment/profile-scoped clone intent plus aggregate user-visible results; incidental status cadence and implementation details are not retained contracts. |
| `repo.delete` | Delete repositories | `delete_repos`, CLI `repo delete` | desktop, docs, CLI | `low` | Preserve the batch-delete capability. The explicit phase-2 scope review below freezes the retained contract at destructive batch deletion plus explicit caller-owned confirmation; current preflight UI details and progress wording are not retained contracts. |

## Local-Only Behaviors Intentionally Outside This Map

These retained behaviors are expected to stay local to shared domain/app code,
not application workflows by default:

- roster editing and clearing
- applying validated group-set import/reimport previews to local state
- local group-set CRUD
- local group CRUD
- assignment CRUD and local selection state
- group-name normalization and pattern-based group selection preview
- undo/redo behavior
- deterministic sorting, filtering, and issue-card presentation logic

If any of these later acquire host-side side effects, they should be promoted
into explicit workflows at that time.

## Phase 2 Repository Workflow Scope Review

This section closes the explicit phase-2 review gate for the low-confidence
repository workflows.

### Retained shared contracts

- `repo.create`, `repo.clone`, and `repo.delete` remain first-class shared
  workflows because they cross host/runtime boundaries and are user-visible
  operations.
- Each workflow retains the high-level batch intent scoped by the selected
  profile and, where applicable, the selected assignment or repository set.
- Each workflow retains aggregate user-visible completion results and
  diagnostic-output semantics as shared workflow concerns.

### Intentionally not retained as shared contracts

- The current desktop preflight dialog shape is not a retained shared contract.
  Review/confirm UX may exist, but it is owned by the delivery surface.
- Exact in-flight status cadence, incidental per-item log wording, and current
  store-specific intermediate states are not frozen as retained contracts.
- Current implementation-shaped planning internals, host task breakdown, and
  transport-specific status plumbing are not preserved requirements.

### Workflow-specific freeze

1. `repo.create` Retain template-backed batch repository creation plus aggregate
   success or failure outcomes. Preserve the need for application-owned
   collision and input validation before host execution, but do not freeze the
   current preflight dialog or every intermediate status detail.
2. `repo.clone` Retain assignment/profile-scoped batch clone intent and
   aggregate outcomes. Do not freeze exact filesystem staging behavior, progress
   percentages, or per-repository log wording unless later promoted as explicit
   retained semantics.
3. `repo.delete` Retain destructive batch delete intent with explicit
   confirmation handled by the invoking surface. Do not freeze the current
   preflight UI shape, exact warning copy, or transient progress states as
   shared workflow requirements.
