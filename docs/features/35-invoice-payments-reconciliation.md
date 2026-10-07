# Feature 35 - Invoice Payments And Reconciliation

## Plan

### Goal

Track invoice payments, partial payments, references, and reconciliation notes.

### Impacted Codebase Areas

- `packages/database/prisma/schema.prisma`: add invoice payment model.
- `apps/api/src/modules/invoices/*`: add payment endpoint and status recalculation.
- `docs/features/35-invoice-payments-reconciliation.md`: plan and build journal.

### Implementation Approach

1. Add `InvoicePayment`.
2. Add payment recording DTO and endpoint.
3. Sum payments to decide `PARTIALLY_PAID` versus `PAID`.
4. Store reference and notes.
5. Audit payment recording.

### Design Considerations

- Payments are source records; invoice status is derived from them.
- Partial payment support makes reconciliation more realistic.
- Overpayments should be blocked in this first version.

## Build Journal

Added `InvoicePayment` and `POST /invoices/:id/payments`. Payments record amount, currency, reference, notes, and received date, then recalculate invoice status as partial or paid.

The important design lesson is source records versus derived status. Payments explain the money movement; invoice status summarizes it.

## Verify It In The Code

- Route: `POST /api/invoices/:id/payments`, platform roles `PLATFORM_ADMIN` or `FINANCE_OPS`
  (`apps/api/src/modules/invoices/invoices.controller.ts:45-47`).
- `InvoicesService.recordPayment` (`apps/api/src/modules/invoices/invoices.service.ts:148`): rejects voided
  invoices (`:153-155`) and currency mismatches (`:156-158`), sums existing payments and rejects an
  overpayment (`:160-163`), then in a transaction creates the payment and sets `PAID` or
  `PARTIALLY_PAID` (`:165-183`), auditing the before/after paid amounts (`:194`).

## Exercises

1. **Goal:** reproduce the overpayment race.
   **Check:** on an invoice of 1000 with nothing paid, send two concurrent 600 payments; both pass the
   check made before the transaction and the invoice ends with 1200 recorded.
2. **Goal:** close it.
   **Check:** re-reading the payment sum inside the transaction (with a row lock on the invoice, or a
   serializable transaction) makes one of the two concurrent requests fail with 400.

## Senior Review Notes

- Only `VOIDED` is blocked, so a `DRAFT` invoice that was never issued can receive payments.
- Status is derived only on payment; there is no path for refunds or reversals to move it back.
