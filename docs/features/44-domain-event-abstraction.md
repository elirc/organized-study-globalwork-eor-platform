# Feature 44 - Domain Event Abstraction

## Plan

### Goal

Add a small domain event service so modules can record durable events without hand-crafting outbox records.

### Impacted Codebase Areas

- `apps/api/src/modules/events/*`: new domain event module/service.
- `packages/database/prisma/schema.prisma`: add domain event model.
- `apps/api/src/modules/notifications/notifications.service.ts`: use event helper where appropriate.
- `docs/features/44-domain-event-abstraction.md`: plan and build journal.

### Implementation Approach

1. Add `DomainEvent`.
2. Add event recording service.
3. Optionally pair event recording with outbox creation.
4. Use it for notification email events.
5. Keep event naming stable and string-based.

### Design Considerations

- Domain events are a history of things that happened.
- Outbox events are side effects to process.
- They are related but not the same record.

## Build Journal

Added `DomainEvent` and a domain-events service. Notification creation now records a `notification.created` domain event in addition to the outbox side effect.

The important design lesson is separating "what happened" from "what to do next." Domain events describe facts; outbox events drive side effects.

## Verify It In The Code

- `DomainEventsService.record` (`apps/api/src/modules/events/domain-events.service.ts:17`) inserts a `DomainEvent`.
- In `NotificationsService.createAndQueue`, the notification and its outbox event are written in one
  transaction (`apps/api/src/modules/notifications/notifications.service.ts:95-116`); the domain event is recorded
  **after** that transaction commits (`:118`), and the BullMQ job is added after that (`:130`).

## Exercises

1. **Goal:** make the fact as durable as the side effect.
   **Check:** passing the transaction client into `record` and calling it inside the `$transaction`
   means a crash between commit and `record` can no longer leave a notification with an outbox event
   but no `notification.created` event.

## Senior Review Notes

- Because the event is written outside the transaction, "domain events are a history of things that
  happened" is best-effort today: a failure after commit loses the event while the notification exists.
