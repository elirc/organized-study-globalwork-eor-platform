# Feature 46 - Client Approval Workflows

## Plan

### Goal

Add configurable client approval workflow definitions and approval step records.

### Impacted Codebase Areas

- `packages/database/prisma/schema.prisma`: add approval workflow and step models.
- `apps/api/src/modules/approvals/*`: new module.
- `docs/features/46-client-approval-workflows.md`: plan and build journal.

### Implementation Approach

1. Add workflow definitions by organization/entity type.
2. Add approval step records for entity approvals.
3. Add endpoints to configure workflows and approve steps.
4. Audit workflow changes and step decisions.
5. Keep integration with expenses/payroll for future slices.

### Design Considerations

- Configurable approvals are a platform capability.
- Approval workflow data should be separate from the business record.
- Future modules can ask approvals whether an action is complete.

## Build Journal

Added approval workflow and approval step models, plus endpoints to create workflows, create entity-specific steps, and review steps.

The important design lesson is decoupling approvals from business records. Expenses, payroll, hires, and other entities can later share the same approval infrastructure.

## Verify It In The Code

- The whole controller is restricted to platform roles `PLATFORM_ADMIN` and `SUPPORT_AGENT`
  (`apps/api/src/modules/approvals/approvals.controller.ts:13-14`), so clients cannot yet approve their own steps
  despite the "client approval" name.
- `reviewStep` reads the step, rejects anything not `PENDING`, then updates it in a separate statement
  (`apps/api/src/modules/approvals/approvals.service.ts:79-92`).

## Exercises

1. **Goal:** make the review race-safe.
   **Check:** replacing the read-then-update with `updateMany({ where: { id, status: "PENDING" } })` and
   checking `count === 1` makes the second of two concurrent reviews fail instead of overwriting the
   first decision.

## Senior Review Notes

- No business module consults approvals yet, so a completed workflow does not gate any action; that is
  consistent with the plan's "future slices" note.
