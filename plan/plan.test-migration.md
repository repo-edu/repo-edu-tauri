# Repo Edu Tauri -> Electron Migration Plan: Test Migration

This document defines how the existing Tauri/Rust test suite should be
evaluated, migrated, rewritten, or removed during the Electron rewrite.

## Purpose

The existing test suite is an input to the migration, not a correctness oracle.

The current codebase can contain bugs, incidental behavior, and
implementation-shaped tests that should not be preserved automatically. The new
project should use the migration to improve the test suite so it better encodes:

- stable domain invariants
- intended user-facing semantics
- retained external behavior where compatibility matters
- architectural boundaries in the new TypeScript system

Test migration therefore has two goals:

1. Improve the long-term quality of the test suite.
2. Reduce accidental behavioral drift during the rewrite where stability is
   still required.

## Test Triage Policy

Every existing test should be classified before or during migration, but the
inventory does not need to become bureaucratic. Default to a high-signal,
area-level inventory grouped by major feature area or legacy suite unless a
specific risky area needs finer-grained decisions.

### 1. Keep and Migrate

Keep and migrate tests that encode durable product value:

- domain invariants
- validation rules
- file-format guarantees
- import/export semantics
- repository planning semantics
- intended CLI command semantics and user-visible output shape
- intended UI behavior that remains part of the redesigned app

These tests should be rewritten into the new package structure where needed, but
their behavioral intent should be preserved.

### 2. Rewrite

Rewrite tests when the behavior is still valuable but the current test is tied
to old architecture or poor seams.

Common rewrite cases:

- Tauri command or generated-binding tests that should become application-layer
  workflow tests
- Rust module tests that should become shared TypeScript domain tests
- UI tests coupled to old store internals that should target clearer feature
  boundaries
- CLI tests that currently assert formatting through brittle implementation
  details rather than stable user-facing output

The new test should preserve the intended behavioral guarantee while aligning
with the new architecture.

### 3. Remove

Remove tests that do not add long-term value.

This includes tests that only preserve:

- incidental implementation details
- generated binding shapes
- Tauri/Electron transport mechanics that no longer exist
- Rust-specific module boundaries
- known-bug behavior
- obsolete workflows or persistence assumptions that are intentionally dropped

Do not carry tests forward simply because they existed before.

### 4. Add

Add new tests where the current suite is weak or where the new architecture
introduces important boundaries.

Priority additions:

- `packages/application` workflow tests
- host port contract tests
- Node host adapter tests
- browser-mock adapter tests
- boundary validation tests for persisted files and imports
- Electron end-to-end coverage for the core desktop flows
- workflow output-channel tests where shared workflows emit logs/diagnostics

## Relationship to Retained Contract Testing

Retained contract testing is scoped, not absolute.

Use retained-contract tests only for retained external semantics where stability
matters. Examples:

- persisted file formats in the new app
- user-visible import/export behavior
- CLI command shape and expected output semantics
- workflow-level diagnostic output semantics where logs/stdout/stderr are part
  of the retained contract
- high-risk workflow results that users rely on

For persistence specifically, "retained" means the new app's defined file
formats after the rewrite, not backward compatibility with legacy Tauri
settings/profile files. As defined in
[plan.architecture.md](./plan.architecture.md), the new code may use different
file locations, schemas, and internal representation, and legacy files are not a
compatibility target.

Do not use retained-contract tests to require bug-for-bug equivalence with the
Tauri/Rust implementation.

When an old behavior is incorrect, unclear, or low-value, prefer replacing it
with a clearer invariant test that expresses the intended behavior.

## Migration Workflow

Apply this workflow as each area is migrated:

1. Inventory the existing tests for that feature area.
2. Classify each test as keep, rewrite, remove, or supersede.
3. Identify missing high-value coverage in the target architecture.
4. Implement the migrated and newly added tests alongside the migrated code.
5. Document any intentionally dropped legacy behavior when it affects a
   user-visible contract.

Confidence rules for that inventory:

- when a feature area already has strong test backing, AI-assisted code analysis
  is an acceptable first draft for the inventory
- areas with stronger current test backing can be treated as higher-confidence
  inputs; in the current codebase, this includes `Roster` and `Group sets &
  Assignments`
- areas with weak or missing current test backing must be marked as
  lower-confidence, code-derived inventories; in the current codebase, this
  includes `Operations`
- a lower-confidence inventory is still useful for planning, but ambiguous
  behaviors in that area must not be treated as retained contracts until they
  have been explicitly reviewed

Do not defer test triage until the end of the rewrite. The migration should
improve the test suite incrementally as the code moves.

## Required Outputs

The migration plan should maintain:

- a lean test triage inventory for major feature areas, not an exhaustive
  per-test ledger
- explicit notes on retained external semantics that require stable automated
  coverage
- confidence annotations where a feature area's current behavior is inferred
  primarily from code structure rather than existing tests
- explicit notes for intentional behavior changes that break with the old system

The rewrite is not complete until the new project has a coherent test suite that
reflects the target architecture rather than the old implementation structure.
User-based acceptance testing should happen only after the rewrite is fully
complete; partial user-based testing during the migration is intentionally not a
phase gate.
