# Feature 31 - Employment Contract Generation

## Plan

### Goal

Generate a simple employment contract artifact from worker, organization, and employment data.

### Impacted Codebase Areas

- `packages/database/prisma/schema.prisma`: add employment contract model.
- `apps/api/src/modules/contracts/*`: new module for generation/listing.
- `apps/api/src/app.module.ts`: register module.
- `docs/features/31-employment-contract-generation.md`: plan and build journal.

### Implementation Approach

1. Add contract status/model.
2. Generate deterministic HTML from employment data.
3. Store storage key, status, and generated timestamp.
4. Audit generation.
5. Keep signature workflow for a future feature.

### Design Considerations

- Contracts are artifacts derived from employment records.
- The generated output should be reproducible enough for local learning.
- Country-specific templates can be layered later.

## Build Journal

Added an employment contracts module. It generates deterministic HTML from worker, organization, and employment data, stores a contract record, and audits generation.

The important design lesson is artifact modeling. The employment record is the source; the contract is a generated artifact that can later gain signature and template workflows.

## Verify It In The Code

- `ContractsService.generate` (`apps/api/src/modules/contracts/contracts.service.ts:21`) builds the contract
  number as `CTR-<year>-<last 6 chars of worker id>` (`:30`) and the storage key from it (`:44`).
- `EmploymentContract.contractNumber` is `@unique` in `packages/database/prisma/schema.prisma`.
- The HTML template escapes organization name, worker name, job title and country with `escapeHtml`
  (`contracts.service.ts:76-86`).

## Exercises

1. **Goal:** find the regeneration failure.
   **Check:** generate a contract for the same worker twice in one calendar year; the second call hits
   the unique `contractNumber` and fails with an unhandled database error instead of a clear 409 or a
   new version.

## Senior Review Notes

- Contract numbers are derived, not allocated, so they collide on regeneration and could collide across
  workers whose ids share the same last six characters. A sequence or a version suffix would fix both.
