# Repo Edu Tauri -> Electron Migration Plan: Library Replacement Matrix

This document records the default replacement strategy for selected major Rust
modules and feature areas before implementation begins.

For each legacy area, classify the target as one of:

- `library-backed replacement`
- `thin custom adapter`
- `custom domain/application rewrite`
- `delete`

Use this document as a bounded decision log for areas where an up-front
replacement decision materially constrains implementation. It is not intended to
be an exhaustive inventory of every legacy module or file. Update it when a
concrete product constraint justifies deviating from the default, or when a
missing replacement decision would change package boundaries or implementation
strategy.

## Current Matrix

| Legacy area | Current Rust location | Target classification | Preferred TS library | Keep custom logic | Notes / risks |
| :--- | :--- | :--- | :--- | :--- | :--- |
| GitHub platform client | `apps/repo-manage/core/src/platform/github.rs` | `library-backed replacement` | `@octokit/rest` | provider-neutral mapping, workflow sequencing, app error normalization | Do not leak Octokit response types outside adapter boundaries |
| GitLab platform client | `apps/repo-manage/core/src/platform/gitlab.rs` | `library-backed replacement` | `@gitbeaker/rest` | provider-neutral mapping, workflow sequencing, app error normalization | Do not leak GitBeaker response types outside adapter boundaries |
| Gitea platform client | `apps/repo-manage/core/src/platform/gitea.rs` | `thin custom adapter` | built-in Node `fetch` | full adapter semantics | API surface is likely narrow; avoid adding another heavy provider dependency by default |
| CSV import/export | `apps/repo-manage/core/src/import/csv.rs` | `library-backed replacement` | `papaparse` | validation, matching, diffing, duplicate detection | Parser owns raw CSV mechanics only, not roster/group semantics |
| Excel import/export | `apps/repo-manage/core/src/import/excel.rs`, `apps/repo-manage/core/src/roster/export.rs` | `library-backed replacement` | `xlsx` (SheetJS) | template validation, export shapes, row normalization | Keep browser-safe use in mind where docs or tests need non-Node execution |
| Settings boundary parsing | `apps/repo-manage/core/src/settings/*` | `library-backed replacement` + `custom domain/application rewrite` | `zod` | cross-field validation, normalization, path rules, profile semantics | Keep structural parsing separate from semantic validation |
| LMS clients | `crates/canvas-lms`, `crates/moodle-lms`, `crates/lms-client` | `thin custom adapter` | built-in Node `fetch` | pagination, retry, normalization, provider-specific boundary mapping | No assumption of strong third-party LMS SDKs |
| Repo workflows | `apps/repo-manage/core/src/operations/repo.rs` | `custom domain/application rewrite` + `thin custom adapter` | direct `child_process.spawn` wrapper over system Git CLI | preflight, collisions, progress, directory layout, orchestration | Do not use `simple-git` as the boundary. Keep Git execution explicit and extensible for future in-app inspection features such as follow-aware log/blame queries |
| Generated bindings and Tauri command transport | `packages/backend-interface`, generated bindings, Tauri command wrappers | `delete` | none | none | Replace with explicit TS modules and typed tRPC transport, not regenerated command facades |
