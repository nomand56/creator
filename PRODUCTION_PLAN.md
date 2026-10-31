# Production restructuring plan

## Current assessment

This is a working prototype, not a production release. The portal uses React, TypeScript, Vite and React Router. The API uses JavaScript, Express and a synchronous JSON file database. The two directories have independent Git repositories; preserve their histories and release boundaries until a coordinated migration is ready.

Verified blockers:
- Authentication compares plaintext passwords, uses a fixed OTP of 1234, and creates sessions without expiry (`src/routes.js`, `src/auth.js`).
- All records share a JSON file. Writes are synchronous and not transactional; concurrent processes can lose changes (`src/store.js`).
- Browser origins are reflected with credentials enabled (`src/index.js`).
- Bearer tokens are stored in localStorage (`src/api/client.ts`).
- Long running discovery jobs execute inside HTTP requests (`src/liveRoutes.js`).
- The media proxy follows redirects after validating only the initial hostname (`src/mediaProxy.js`).
- The seed command references a missing `src/seed-run.js`.
- No automated test suite or release workflow was found in the inspected source.

## Proposed stack

Retain React, TypeScript, Vite and Express to preserve existing work. Convert the API to TypeScript incrementally rather than rewriting the product. Use PostgreSQL for durable relational records and transactions, Prisma for schema migrations and typed queries, and a separate worker with Redis/BullMQ for discovery and enrichment jobs. These are architectural proposals; validate exact supported versions and provider constraints before installation.

Use an established identity provider with verified email, recovery and expiring sessions. Prefer secure, HttpOnly cookies behind the same origin, with CSRF protection. Keep secrets server-side. Store exported files in private object storage with expiring download links. Serve the portal and `/api` behind one TLS endpoint. Start with managed database, worker and application hosting; Kubernetes is unnecessary for this scope.

## Target structure

Within the existing backend repository:
```
src/
  app.ts
  server.ts
  config/
  middleware/
  modules/
    identity/
    workspaces/
    creators/
    lists/
    campaigns/
    outreach/
    integrations/
    exports/
  infrastructure/
    database/
    queue/
    providers/
  workers/
prisma/
tests/
```

Within the portal repository, move pages, components and API hooks into `src/features/<domain>/`; retain `src/app/` for routing/providers and `src/shared/` for reusable UI and utilities. Replace the growing WorkspaceContext with domain hooks and server-state queries incrementally. Preserve URLs and current response shapes during migration.

## Delivery sequence and acceptance gates

1. Baseline: repair the portal build, record behavior, add API integration tests around authentication, list ownership and discovery; establish CI with clean installs, type checking, lint and builds. Acceptance: reproducible green checks in both repositories.
2. Identity and authorization: replace demo auth, expire and revoke sessions, rate-limit authentication, validate requests, enforce workspace ownership on every read/write/export. Acceptance: cross-workspace access tests fail closed and no plaintext credentials or fixed OTPs remain.
3. Persistence: define users, workspaces, memberships, creators, list membership, campaigns, outreach, jobs and audit tables. Put database access behind repositories before moving routes. Build a repeatable JSON importer with row counts, stable identifiers and validation. Acceptance: transaction tests and backup/restore rehearsal pass; reconcile imported records before cutover. Retain an encrypted source snapshot and a rollback procedure.
4. Module migration: separate route, validation, service and repository responsibilities one domain at a time. Introduce shared API schemas without coupling the two deployment pipelines. Acceptance: existing API contract tests pass after each move.
5. Jobs and providers: return job IDs, execute sync in a separate worker, add progress polling, retries with backoff, deduplication, timeouts and provider quotas. Restrict outbound destinations and validate every redirect; cap response sizes. Acceptance: worker restart does not lose or duplicate jobs and proxy security tests pass.
6. Operations: add validated environment configuration, restricted CORS, security headers, structured logs with secret redaction, request IDs, error reporting, readiness checks and graceful shutdown. Deploy to staging with private database/Redis and managed secrets. Acceptance: monitoring, migration rollback and restore exercises pass.
7. Release: test complete signup/login, discovery, list editing, outreach and export workflows; test realistic concurrent traffic and provider failures. Agree latency/error targets and retention policy. Promote the same tested artifact, run migrations as a controlled release step and verify smoke tests. Production remains blocked until all preceding gates pass.

## Verification from this review

### Implementation progress

- Sessions now expire after eight hours; sessions without an expiry fail closed, requiring existing users to sign in again.
- Saved searches, campaigns, outreach and automations now carry account ownership and filter reads; outreach and automation updates enforce ownership. Legacy unowned records remain hidden pending verified migration. This is interim account isolation, not the planned workspace membership model: workspace settings, team, platform activity and the shared discovery catalog still need redesign.
- Browser origin access now uses `CORS_ORIGINS` (comma-separated; default `http://localhost:5173`). Added no-sniff and no-store headers and explicit malformed/oversized JSON responses. Integration tests cover these behaviors and session expiry.

- Extracted backend app creation into `src/app.js`, keeping startup and the production block in `src/index.js`.
- Added isolated HTTP integration coverage for authentication rejection, logout revocation, discovery authentication, and list ownership across reads, updates, membership changes, deletion and exports. Tests use a temporary database and do not call live providers.
- Added owner IDs to newly created lists and the development seed. Existing unowned lists fail closed; an operator must verify and assign ownership during migration. Other domain records still require workspace authorization work.
- Added independent CI workflows to both repositories with clean dependency installation and their available checks. Remote CI has not been run.
- Repaired the missing seed entry point with non-destructive initialization. Added backend syntax checks and a test command.
- Backend syntax and HTTP integration checks pass locally. Database replacement, real identity, complete workspace isolation and live provider tests remain outstanding; the production startup block remains enabled.

The initial portal build failed because WorkspaceContext imported an unused useRef; that import was removed. Lint completed with 12 warnings before the fix, including effect-driven state updates and ref access during render. Resolve remaining warnings during the domain migration. Deployment, database migration, provider verification and end-to-end validation have not been performed.
