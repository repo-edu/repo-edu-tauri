# Migration Plan Index

This directory contains the planning documents for the `repo-edu-tauri` to
`repo-edu` Electron rewrite.

If you are a fresh AI or a new reviewer, do not read these files in arbitrary
order. Use the reading sequence below.

## Read Order

1. [plan.md](./plan.md) This is the executive summary: objective, hard
   requirements, architectural direction, risks, and success criteria.
2. [plan.delivery.md](./plan.delivery.md) This defines the phase structure, exit
   criteria, and testing strategy.
3. [plan.implementation-checklist.md](./plan.implementation-checklist.md) This
   is the canonical execution ledger for implementation. It defines the
   resumable task order, checkbox tracking, and resume protocol.
4. Read only the supporting document(s) needed for the next task:
   - [plan.architecture.md](./plan.architecture.md): detailed target
     architecture and package boundaries
   - [plan.workflow-mapping.md](./plan.workflow-mapping.md): retained workflow
     ids and delivery-target mapping
   - [plan.retained-contract-inventory.md](./plan.retained-contract-inventory.md):
     coarse retained user-facing contracts
   - [plan.test-migration.md](./plan.test-migration.md): test migration policy
   - [plan.test-triage-inventory.md](./plan.test-triage-inventory.md): major
     legacy test-area decisions
   - [plan.legacy-test-migration-map.md](./plan.legacy-test-migration-map.md):
     target-layer routing for old test areas
   - [plan.library-replacement-matrix.md](./plan.library-replacement-matrix.md):
     up-front library vs custom replacement decisions

## Where Implementation Starts

Implementation does not start from whichever plan file happens to be open.

Implementation starts in
[plan.implementation-checklist.md](./plan.implementation-checklist.md):

1. Find the first unchecked checkbox whose prerequisites are satisfied.
2. Read any supporting document that is directly relevant to that checkbox.
3. Implement that item.
4. Update the checkbox in the same change set when the item is actually
   complete.
5. If interrupted, resume from the first remaining unchecked item.

## Authority Map

Use each file for its intended purpose:

- [plan.md](./plan.md): scope, constraints, and success criteria
- [plan.delivery.md](./plan.delivery.md): phase structure and exit gates
- [plan.implementation-checklist.md](./plan.implementation-checklist.md):
  implementation sequence and progress tracking
- Supporting files: detailed reference material for the active task only

If two files seem to conflict:

- execution order and progress tracking are controlled by
  [plan.implementation-checklist.md](./plan.implementation-checklist.md)
- architecture and scope are controlled by [plan.md](./plan.md) and
  [plan.architecture.md](./plan.architecture.md)
- phase intent and validation gates are controlled by
  [plan.delivery.md](./plan.delivery.md)

## Minimal Resume Procedure

For a fresh AI context:

1. Read this file.
2. Read [plan.md](./plan.md).
3. Read [plan.delivery.md](./plan.delivery.md).
4. Read [plan.implementation-checklist.md](./plan.implementation-checklist.md).
5. Continue from the first eligible unchecked checkbox.
