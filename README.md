# GlobalWork EOR Platform

GlobalWork is a fictional Employer of Record SaaS platform built as a TypeScript modular monolith. It is designed as a living software project for practicing real-world engineering: domain modeling, tenant isolation, workflow state, auditability, background jobs, testing, and maintainable architecture.

## Stack

- Next.js and React for the web app
- NestJS for the API
- PostgreSQL and Prisma for persistence
- Redis and BullMQ for background jobs
- Docker Compose for local infrastructure
- Playwright, Vitest/Jest, and Testcontainers for testing
- OpenTelemetry-ready structure for observability

## Local Setup

Corepack can run pnpm even when a global `pnpm` shim is unavailable:

```powershell
corepack pnpm install
Copy-Item .env.example .env
docker compose up -d
corepack pnpm db:generate
corepack pnpm db:migrate
corepack pnpm db:seed
corepack pnpm dev
```

The web app runs on `http://localhost:3000`.
The API runs on `http://localhost:4000/api`.
Mailpit runs on `http://localhost:8025`.
MinIO console runs on `http://localhost:9001`.

## Architecture Journal

Start with [docs/ARCHITECTURE_JOURNAL.md](docs/ARCHITECTURE_JOURNAL.md). It explains the project structure, how the modules connect, and why the codebase is shaped this way.

For a deeper file-by-file learning path, read [docs/code-walkthroughs/README.md](docs/code-walkthroughs/README.md). It annotates the highest-value source files and explains how to trace workflows through the codebase.

For the forward-looking architecture plan behind the new feature work, read [docs/SYSTEM_DESIGN_FORWARD_PLAN.md](docs/SYSTEM_DESIGN_FORWARD_PLAN.md).

Feature implementation plans and journals live in `docs/features/`.

## Security Review Highlights (read before reusing the code)

These were found by reading the current source; each is a good first exercise.

1. **Self-registration into any tenant.** `POST /api/auth/register` is public
   (`apps/api/src/modules/auth/auth.controller.ts:15-18`), accepts an optional `organizationId`
   (`auth/dto/register.dto.ts`), and creates a `CLIENT_HR_ADMIN` membership in that organization
   with no invitation or check (`auth/auth.service.ts:24-38`). Anyone who learns an organization id
   becomes its HR admin.
2. **Time-off endpoints have no tenant check.** Balances, request creation and review use only
   `AuthGuard` (`time-off/time-off.controller.ts:22-46`), and the service loads workers and requests by
   id without comparing organizations (`time-off/time-off.service.ts:24-61`). Any authenticated user
   can file or approve time off for any worker, including their own requests.
3. **Check-then-act races on money and balances.** The overpayment check runs before the payment
   transaction (`invoices/invoices.service.ts:160-165`), and the vacation-balance check runs before the
   approval transaction (`time-off/time-off.service.ts:70-76`); two concurrent requests can both pass.
4. **Impersonation tokens are indistinguishable from real logins.** `impersonate` records a `tokenId`
   in `ImpersonationSession` but signs a token that contains only `sub`, `email`, `iat`, `exp`
   (`auth/auth.service.ts:74-90`, `auth/token.ts:3-8`), so actions taken while impersonating are
   audited as the target user and the session cannot be revoked.

Feature docs 28-47 now end with "Verify It In The Code" sections that tie the build journals to file
and line anchors.
