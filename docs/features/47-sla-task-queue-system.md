# Feature 47 - SLA And Task Queue System

## Plan

### Goal

Create an internal task queue with owners, due dates, statuses, and SLA priority.

### Impacted Codebase Areas

- `packages/database/prisma/schema.prisma`: add task status/priority/model.
- `apps/api/src/modules/tasks/*`: new module.
- `docs/features/47-sla-task-queue-system.md`: plan and build journal.

### Implementation Approach

1. Add task model with organization, assignee, entity reference, status, priority, due date.
2. Add create/list/update endpoints.
3. Add overdue-friendly ordering.
4. Audit task lifecycle changes.
5. Leave automatic task creation for future features.

### Design Considerations

- Internal operations teams need queueable work, not just domain records.
- SLA fields help prioritize repeated operational workflows.
- Task records can point at many entity types through generic references.

## Build Journal

Added a task queue module with task create/list/update endpoints, assignees, entity references, priority, status, due dates, and audit logging.

The important design lesson is operational work tracking. Real internal teams need tasks that can point at domain records without becoming part of every domain model.

## Verify It In The Code

- Access: platform roles `PLATFORM_ADMIN`, `SUPPORT_AGENT`, `ONBOARDING_SPECIALIST`,
  `PAYROLL_SPECIALIST` (`apps/api/src/modules/tasks/tasks.controller.ts:12-13`).
- `TasksService.list` filters by optional organization and status (`apps/api/src/modules/tasks/tasks.service.ts:15-24`);
  `create` (`:26`) and `update` (`:51`) audit with the task's organization when it has one.

## Exercises

1. **Goal:** make priority count.
   **Check:** `list` orders by `dueAt` ascending then `createdAt` descending (`tasks.service.ts:21`) and
   ignores `priority`; after adding priority to the sort, an `URGENT` task due tomorrow is listed before a
   `LOW` task due today only if that is the rule you choose, and a test pins it.

## Senior Review Notes

- The plan's "SLA priority" is stored but not used for ordering; ordering is purely by due date.
- The `status` query parameter is cast with `as never` (`tasks.service.ts:19`) instead of being
  validated, so an unknown status string reaches Prisma and fails as a server error.
- `entityType`/`entityId` references are free-form, so nothing checks that the referenced record exists
  or belongs to the task's organization.
