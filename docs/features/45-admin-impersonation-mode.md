# Feature 45 - Admin Impersonation Mode

## Plan

### Goal

Allow platform admins to create audited impersonation sessions for support workflows.

### Impacted Codebase Areas

- `packages/database/prisma/schema.prisma`: add impersonation session model.
- `apps/api/src/modules/auth/*`: add impersonation endpoint.
- `docs/features/45-admin-impersonation-mode.md`: plan and build journal.

### Implementation Approach

1. Add `ImpersonationSession`.
2. Add platform-admin endpoint to impersonate a user.
3. Issue a token for the target user and record impersonator.
4. Include impersonation metadata in response.
5. Audit session creation.

### Design Considerations

- Impersonation is high-risk and must be explicit.
- The UI should show impersonation state later.
- Tokens should carry enough context for audit in a future version.

## Build Journal

Added impersonation sessions and a platform-admin `POST /auth/impersonate` endpoint. The endpoint records the impersonator, target user, reason, token id, audit log, and returns a short-lived token for the target user.

The important design lesson is explicit risk modeling. Impersonation is useful for support, but it must be visible and auditable from the beginning.

## Verify It In The Code

- Route: `POST /api/auth/impersonate` with `AuthGuard`, `PlatformRoleGuard` and
  `@RequirePlatformRoles("PLATFORM_ADMIN")` (`apps/api/src/modules/auth/auth.controller.ts:31-36`).
- Service: `AuthService.impersonate` (`apps/api/src/modules/auth/auth.service.ts:67-114`) loads the target,
  signs a one-hour token for **the target** (`:75-83`), creates the `ImpersonationSession` with a random
  `tokenId` (`:74,85-92`), writes an `auth.impersonation_started` audit row (`:94-103`), and returns the
  token plus session metadata.
- Token format: `signAuthToken` signs only `{ sub, email, iat, exp }` with HMAC-SHA256
  (`apps/api/src/modules/auth/token.ts:3-14`); verification checks signature and expiry only (`:16-33`).

## Exercises

1. **Goal:** see what a downstream audit row records.
   **Check:** impersonate a client HR admin, then call any audited write (for example a time-off
   review); the resulting `AuditLog.actorId` is the target user's id, and nothing on the row links it to
   the `ImpersonationSession`.
2. **Goal:** make the session revocable.
   **Check:** after adding `tid` (the `tokenId`) and `imp` (the impersonator id) to the token payload and
   having `verifyBearerToken` reject tokens whose session has `endedAt` set, ending a session makes the
   next request with that token return 401.

## Senior Review Notes

- The `tokenId` is stored but never placed in the token, so the "records token id" step in the build
  journal has no enforcement value yet.
- There is no restriction on the target: a platform admin can impersonate another platform admin.
- The session insert and the audit insert are separate statements, not one transaction.
- A required, non-empty `reason` is the only friction; there is no second approval or time-boxed
  scope beyond the one-hour expiry.
