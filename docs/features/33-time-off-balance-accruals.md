# Feature 33 - Time-Off Balance Accruals

## Plan

### Goal

Track annual time-off allowance, used days, and remaining balance per worker/year/type.

### Impacted Codebase Areas

- `packages/database/prisma/schema.prisma`: add time-off balance model.
- `apps/api/src/modules/time-off/*`: add balance endpoint and update balances on approval.
- `docs/features/33-time-off-balance-accruals.md`: plan and build journal.

### Implementation Approach

1. Add `TimeOffBalance`.
2. Initialize or upsert balances from country annual PTO defaults.
3. When vacation time off is approved, increment used days.
4. Add balance list endpoint.
5. Block approval if balance would go negative.

### Design Considerations

- Balances are derived operational records but need persistence for audit and performance.
- Real accrual rules vary by country; this first version is annual allowance based.
- Decimal days require care because partial-day leave is common.

## Build Journal

Added `TimeOffBalance` and worker balance endpoints. Vacation approval now ensures an annual balance exists, blocks overuse, and increments used days transactionally.

The important design lesson is derived state with guardrails. Balances are derived from policy and requests, but persisting them makes review and reporting practical.

## Verify It In The Code

- Routes: `GET /api/workers/:workerId/time-off-balances`, `POST /api/workers/:workerId/time-off-requests`
  and `PATCH /api/time-off-requests/:id/review` use only `AuthGuard`
  (`apps/api/src/modules/time-off/time-off.controller.ts:22-46`).
- `balances` lazily creates the vacation balance before listing it
  (`apps/api/src/modules/time-off/time-off.service.ts:24-30`).
- `review` validates the transition, then for approved vacation compares `usedDays + days` with
  `allowanceDays` (`:61-73`) **before** opening the transaction that updates the request and increments
  `usedDays` (`:76-97`).
- `days` comes from the client (`apps/api/src/modules/time-off/dto/create-time-off-request.dto.ts`,
  `@Min(0.5)`) and is not checked against the start and end dates.

## Exercises

1. **Goal:** confirm the missing tenant check.
   **Check:** as a user with no membership in a worker's organization, `PATCH` one of that worker's
   pending requests to `APPROVED`; it succeeds.
2. **Goal:** reproduce the balance race.
   **Check:** approve two pending vacation requests concurrently that each fit the allowance alone but
   not together; both approvals succeed and `usedDays` exceeds `allowanceDays`.
3. **Goal:** derive days.
   **Check:** a request for Monday to Friday with `days: 1` is rejected after you compute working days
   from the dates on the server.

## Senior Review Notes

- The three routes need the same organization-role guard as `GET /time-off-requests`
  (`time-off.controller.ts:15-17`), plus a check that the reviewer is not the requester.
- Move the allowance check inside the transaction, or use a conditional update
  (`usedDays + days <= allowanceDays`) and check the affected row count.
