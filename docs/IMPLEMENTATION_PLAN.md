# Zoro Gateways — Implementation Plan

**Status:** Approved implementation plan  
**Version:** 1.0.0  
**Date:** 2026-07-25  
**Repository:** `kofiarhin/zoro-gateways`  
**Target branch:** `main`  
**Related specification:** [`ZORO_GATEWAYS_SPEC.md`](ZORO_GATEWAYS_SPEC.md)

## 1. Delivery objective

Build an npm-workspaces monorepo containing four independently deployable Express gateways:

- Engineering Gateway;
- GitHub Gateway;
- Vercel Gateway;
- Heroku Gateway.

The first usable milestone must allow Zoro to:

1. authenticate to a separately deployed GitHub Action;
2. inspect approved repositories through GitHub App authentication;
3. create an isolated engineering workspace;
4. inspect and patch source files;
5. run allowlisted tests and builds;
6. review the resulting Git diff;
7. commit and push an isolated branch;
8. open a pull request;
9. retain structured execution evidence.

Vercel and Heroku functionality will follow after the shared foundation and Engineering/GitHub loop are verified.

## 2. Delivery principles

1. **Test first:** implement route, policy, and service behaviour from Jest and Supertest tests.
2. **One bounded phase at a time:** do not build all providers simultaneously.
3. **Shared contracts first:** authentication, errors, responses, and policy must not be duplicated across apps.
4. **Provider adapters:** route handlers must not contain raw provider API logic.
5. **No unrestricted shell:** commands are selected from an allowlisted registry.
6. **Isolation before mutation:** source edits and commands run only inside approved workspaces.
7. **Evidence before completion:** every meaningful operation produces traceable results.
8. **Separate deployment verification:** local passing tests do not prove a deployed Action works.
9. **Explicit protected-operation gates:** merge, production, destructive, and direct-main operations remain blocked by default.
10. **No secrets in Git:** repository files contain names and examples only.

## 3. Planned repository structure

```text
zoro-gateways/
├── .github/
│   └── workflows/
│       └── ci.yml
├── apps/
│   ├── engineering-gateway/
│   │   ├── src/
│   │   │   ├── adapters/
│   │   │   ├── commands/
│   │   │   ├── config/
│   │   │   ├── controllers/
│   │   │   ├── middleware/
│   │   │   ├── repositories/
│   │   │   ├── routes/
│   │   │   ├── services/
│   │   │   ├── validators/
│   │   │   ├── app.js
│   │   │   └── server.js
│   │   ├── tests/
│   │   │   ├── integration/
│   │   │   └── unit/
│   │   ├── openapi.json
│   │   ├── .env.example
│   │   └── package.json
│   ├── github-gateway/
│   │   ├── src/
│   │   ├── tests/
│   │   ├── openapi.json
│   │   ├── .env.example
│   │   └── package.json
│   ├── vercel-gateway/
│   │   ├── src/
│   │   ├── tests/
│   │   ├── openapi.json
│   │   ├── .env.example
│   │   └── package.json
│   └── heroku-gateway/
│       ├── src/
│       ├── tests/
│       ├── openapi.json
│       ├── .env.example
│       └── package.json
├── packages/
│   ├── contracts/
│   ├── gateway-core/
│   ├── policy-engine/
│   └── testing/
├── scripts/
│   ├── start-gateway.js
│   ├── validate-env.js
│   └── validate-openapi.js
├── docs/
│   ├── ZORO_GATEWAYS_SPEC.md
│   └── IMPLEMENTATION_PLAN.md
├── .editorconfig
├── .env.example
├── .gitignore
├── eslint.config.js
├── package.json
├── package-lock.json
├── Procfile
└── README.md
```

## 4. Root package design

The root `package.json` will:

- declare npm workspaces;
- select CommonJS for compatibility with the current Zoro server;
- use Node 22;
- install shared development tooling;
- provide per-gateway development and test scripts;
- provide one production start entrypoint controlled by `GATEWAY_APP`.

Planned scripts:

```json
{
  "scripts": {
    "dev": "concurrently -k -n engineering,github,vercel,heroku ...",
    "dev:engineering": "npm run dev --workspace=@zoro/engineering-gateway",
    "dev:github": "npm run dev --workspace=@zoro/github-gateway",
    "dev:vercel": "npm run dev --workspace=@zoro/vercel-gateway",
    "dev:heroku": "npm run dev --workspace=@zoro/heroku-gateway",
    "start": "node scripts/start-gateway.js",
    "start:engineering": "npm run start --workspace=@zoro/engineering-gateway",
    "start:github": "npm run start --workspace=@zoro/github-gateway",
    "start:vercel": "npm run start --workspace=@zoro/vercel-gateway",
    "start:heroku": "npm run start --workspace=@zoro/heroku-gateway",
    "test": "npm run test --workspaces --if-present",
    "test:coverage": "npm run test:coverage --workspaces --if-present",
    "lint": "eslint .",
    "openapi:validate": "node scripts/validate-openapi.js",
    "ci": "npm run lint && npm test && npm run openapi:validate"
  }
}
```

## 5. Phase overview

| Phase | Outcome | Depends on |
| --- | --- | --- |
| 0 | Repository foundation and CI | None |
| 1 | Shared contracts, authentication, policy, and testing packages | Phase 0 |
| 2 | GitHub Gateway read MVP | Phase 1 |
| 3 | GitHub Gateway write and PR workflow | Phase 2 |
| 4 | Engineering Gateway run and workspace core | Phase 1 |
| 5 | Engineering file, command, and check capabilities | Phase 4 |
| 6 | Engineering-to-GitHub delivery loop | Phases 3 and 5 |
| 7 | Deployment and GPT Action verification | Phase 6 |
| 8 | Vercel Gateway extraction | Phase 7 |
| 9 | Heroku Gateway | Phase 7 |
| 10 | Hardening, persistence, scaling, and browser/database extensions | Earlier phases |

## 6. Phase 0 — Repository foundation

### 6.1 Scope

Create the monorepo skeleton and development conventions.

### 6.2 Files

```text
package.json
package-lock.json
.gitignore
.editorconfig
eslint.config.js
Procfile
README.md
scripts/start-gateway.js
scripts/validate-env.js
.github/workflows/ci.yml
apps/*/package.json
packages/*/package.json
```

### 6.3 Tasks

1. Configure npm workspaces.
2. Add Node 22 and npm engine requirements.
3. Add `concurrently`, Jest, Supertest, ESLint, and OpenAPI validation tooling.
4. Add the `GATEWAY_APP` production selector.
5. Add one minimal Express app per gateway.
6. Add `/health` to every gateway.
7. Add environment validation on startup.
8. Add graceful SIGTERM/SIGINT shutdown.
9. Add a root CI workflow.
10. Add `.env.example` files without secret values.

### 6.4 Tests

- `start-gateway.js` rejects missing or unsupported `GATEWAY_APP` values.
- Every app exports `createApp()` without binding a port.
- Every `/health` route returns `200` and identifies the service.
- Missing required production configuration prevents startup.

### 6.5 Acceptance criteria

- `npm ci` succeeds from a clean checkout.
- `npm run dev:github` starts only GitHub Gateway.
- `npm run dev` starts all four gateways locally.
- `GATEWAY_APP=github npm start` starts GitHub Gateway.
- CI runs on pushes and pull requests.

## 7. Phase 1 — Shared platform packages

### 7.1 `@zoro/contracts`

Implement schemas and builders for:

- success envelopes;
- error envelopes;
- approval metadata;
- evidence records;
- pagination;
- async run acceptance;
- health and readiness.

Suggested files:

```text
packages/contracts/src/
├── approval.js
├── evidence.js
├── errors.js
├── pagination.js
├── responses.js
└── index.js
```

### 7.2 `@zoro/gateway-core`

Implement:

- request ID middleware;
- Bearer authentication middleware;
- centralized error middleware;
- structured logger;
- async route wrapper;
- request size limits;
- readiness helper;
- secret redactor.

Suggested files:

```text
packages/gateway-core/src/
├── auth.js
├── errors.js
├── logger.js
├── readiness.js
├── request-id.js
├── response.js
├── redact.js
└── index.js
```

### 7.3 `@zoro/policy-engine`

Implement:

- operation-class registry;
- repository/project/app allowlists;
- explicit approval checks;
- exact confirmation checks;
- production-target checks;
- direct-main checks;
- protected-operation middleware;
- idempotency requirement checks.

Suggested files:

```text
packages/policy-engine/src/
├── approvals.js
├── classifications.js
├── confirmations.js
├── direct-main.js
├── resources.js
├── middleware.js
└── index.js
```

### 7.4 `@zoro/testing`

Implement shared Jest/Supertest helpers:

```text
packages/testing/src/
├── auth.js
├── contracts.js
├── fixtures.js
├── provider-mocks.js
└── index.js
```

### 7.5 Tests

Write tests before implementation for:

- valid and invalid Bearer credentials;
- missing server credential;
- request ID propagation;
- consistent success and error envelopes;
- redaction of token-like values;
- approval scope mismatch;
- confirmation mismatch;
- direct-main rejection;
- allowlisted and non-allowlisted resources;
- idempotency requirements.

### 7.6 Acceptance criteria

- All gateways use the same middleware and envelope packages.
- No gateway duplicates authentication or policy implementation.
- Protected operations fail closed when configuration is missing.
- Contract tests pass for all shared response types.

## 8. Phase 2 — GitHub Gateway read MVP

### 8.1 Scope

Implement safe GitHub App-backed reads first.

### 8.2 Provider adapter

Create a GitHub adapter that owns Octokit usage:

```text
apps/github-gateway/src/adapters/github/
├── app-client.js
├── installation-client.js
├── repositories.js
├── contents.js
├── branches.js
├── commits.js
└── errors.js
```

Controllers and routes must call services, and services must call the adapter. They must not instantiate Octokit directly.

### 8.3 Initial routes

```http
GET /api/v1/repositories
GET /api/v1/repositories/:owner/:repo
GET /api/v1/repositories/:owner/:repo/contents
GET /api/v1/repositories/:owner/:repo/branches
GET /api/v1/repositories/:owner/:repo/commits
```

### 8.4 Validation

Validate:

- owner and repository names;
- paths;
- refs;
- page size and cursor parameters;
- repository allowlist where configured.

### 8.5 Tests

For every route:

1. unauthorized request;
2. validation failure;
3. successful provider response;
4. provider not found;
5. provider rate limit;
6. provider permission failure;
7. response contract compliance;
8. secret-redaction assertion.

### 8.6 OpenAPI

Publish `apps/github-gateway/openapi.json` with stable operation IDs:

```text
githubListRepositories
githubGetRepository
githubGetRepositoryContents
githubListBranches
githubListCommits
```

### 8.7 Acceptance criteria

- Local tests pass with provider adapters mocked.
- A development GitHub App installation can list accessible repositories.
- `/openapi.json` validates as OpenAPI 3.1.
- No private key or installation token appears in logs or responses.

## 9. Phase 3 — GitHub Gateway write and pull-request workflow

### 9.1 Scope

Add branch and pull-request capabilities after read MVP verification.

### 9.2 Routes

```http
POST /api/v1/repositories/:owner/:repo/branches
POST /api/v1/repositories/:owner/:repo/pulls
GET  /api/v1/repositories/:owner/:repo/pulls/:number
GET  /api/v1/repositories/:owner/:repo/pulls/:number/files
GET  /api/v1/repositories/:owner/:repo/pulls/:number/reviews
POST /api/v1/repositories/:owner/:repo/pulls/:number/comments
GET  /api/v1/repositories/:owner/:repo/actions/runs
GET  /api/v1/repositories/:owner/:repo/actions/runs/:runId
GET  /api/v1/repositories/:owner/:repo/actions/jobs/:jobId/logs
```

Merge is implemented but remains disabled until explicit approval policy tests and a disposable end-to-end target exist:

```http
POST /api/v1/repositories/:owner/:repo/pulls/:number/merge
```

### 9.3 Required controls

- repository allowlist;
- protected default branch detection;
- explicit branch naming policy;
- idempotency for create operations;
- approval requirement for merge;
- explicit direct-main gate;
- evidence containing repository, branch, commit, and pull-request identifiers.

### 9.4 Tests

- branch creation on allowed repository;
- branch creation rejection on invalid base;
- duplicate idempotency key;
- pull-request creation;
- direct-main rejection;
- merge rejection without approval;
- merge confirmation scope mismatch;
- CI run and job log normalization.

### 9.5 Acceptance criteria

- Zoro can create an isolated branch and pull request in a disposable repository.
- Direct-main writes remain blocked without explicit authority.
- Merge remains blocked without explicit approval.
- CI evidence can be retrieved for a pull-request revision.

## 10. Phase 4 — Engineering Gateway run and workspace core

### 10.1 Persistence model

Use MongoDB for initial durable metadata.

Collections:

```text
engineering_runs
engineering_jobs
engineering_workspaces
engineering_commands
engineering_approvals
engineering_evidence
engineering_artifacts
engineering_idempotency_keys
```

### 10.2 Run model

Minimum fields:

```text
runId
status
task
repository
baseBranch
baseRevision
workKey
workspaceId
requestedOperations
requiredChecks
approvalState
createdAt
updatedAt
startedAt
finishedAt
error
```

### 10.3 Workspace model

Minimum fields:

```text
workspaceId
runId
repository
baseRevision
branch
isolationType
rootPath
status
expiresAt
cleanupStatus
createdAt
updatedAt
```

### 10.4 Routes

```http
POST /api/v1/runs
GET  /api/v1/runs/:runId
POST /api/v1/runs/:runId/cancel
GET  /api/v1/runs/:runId/events

POST   /api/v1/workspaces
GET    /api/v1/workspaces/:workspaceId
DELETE /api/v1/workspaces/:workspaceId
```

### 10.5 Isolation implementation

MVP implementation uses Git worktrees inside a dedicated workspace root:

```text
WORKSPACE_ROOT=/var/lib/zoro/workspaces
```

Rules:

- canonicalize all paths;
- allocate one directory per workspace ID;
- create one non-default branch per mutating run;
- store the exact base revision;
- reject reuse across unrelated runs;
- enforce expiration;
- clean up worktrees and processes;
- record cleanup failures.

Container isolation can replace or wrap worktrees in a later hardening phase.

### 10.6 Tests

- valid run creation;
- invalid repository rejection;
- duplicate idempotency key;
- valid run-state transitions;
- invalid transition rejection;
- workspace path containment;
- default-branch mutation rejection;
- workspace expiration;
- cleanup success and failure reporting.

### 10.7 Acceptance criteria

- Run records survive API restarts.
- A disposable repository can be provisioned at an exact revision.
- The workspace receives a unique branch and directory.
- Workspace deletion removes its worktree and records the outcome.

## 11. Phase 5 — Engineering file, command, and check capabilities

### 11.1 Repository inspection

Implement:

```http
GET  /api/v1/workspaces/:workspaceId/tree
GET  /api/v1/workspaces/:workspaceId/file
POST /api/v1/workspaces/:workspaceId/search
POST /api/v1/workspaces/:workspaceId/dependencies/analyse
```

Use native filesystem APIs and a bounded search process. Do not return binary file contents through text endpoints.

### 11.2 File writes

Implement:

```http
PUT  /api/v1/workspaces/:workspaceId/file
POST /api/v1/workspaces/:workspaceId/patch
POST /api/v1/workspaces/:workspaceId/files/batch
```

Controls:

- workspace containment;
- allowed file-size limits;
- binary-file rejection;
- precondition content hash;
- atomic temporary-file replacement;
- diff generation;
- secret scanning;
- audit evidence.

### 11.3 Command registry

Implement a declarative registry:

```js
{
  id: "jest",
  executable: "npm",
  args: ["test", "--", "--runInBand"],
  timeoutMs: 120000,
  permission: "workspace-write",
  allowedWorkingDirectories: ["."],
  allowedEnvironmentKeys: ["NODE_ENV", "CI"]
}
```

Initial registry:

- `npm-ci`;
- `npm-install`;
- `npm-test`;
- `npm-run` with allowlisted scripts;
- `jest`;
- `vitest`;
- `eslint`;
- `tsc`;
- `vite-build`;
- `git-status`;
- `git-diff`.

### 11.4 Command routes

```http
POST /api/v1/workspaces/:workspaceId/commands
GET  /api/v1/commands/:commandId
POST /api/v1/commands/:commandId/cancel
GET  /api/v1/commands/:commandId/logs
```

### 11.5 Check routes

```http
POST /api/v1/workspaces/:workspaceId/checks/lint
POST /api/v1/workspaces/:workspaceId/checks/typecheck
POST /api/v1/workspaces/:workspaceId/checks/test
POST /api/v1/workspaces/:workspaceId/checks/build
POST /api/v1/workspaces/:workspaceId/checks/security
POST /api/v1/workspaces/:workspaceId/checks/all
```

`checks/all` detects repository scripts and runs only applicable checks. Missing checks are reported as `not_configured`, not silently treated as passed.

### 11.6 Git routes

```http
GET  /api/v1/workspaces/:workspaceId/git/status
GET  /api/v1/workspaces/:workspaceId/git/diff
POST /api/v1/workspaces/:workspaceId/git/stage
POST /api/v1/workspaces/:workspaceId/git/commit
```

Push is deferred until Phase 6 integration.

### 11.7 Tests

- path traversal attempts;
- stale content hash conflict;
- atomic batch rollback;
- command ID allowlist rejection;
- unsupported package script rejection;
- timeout and cancellation;
- output truncation;
- secret redaction;
- failed and passing checks;
- dirty and clean Git states;
- commit rejection when checks are required but failing.

### 11.8 Acceptance criteria

- Zoro can inspect and patch a disposable MERN repository.
- Jest/Vitest and build commands run through named command definitions.
- Full logs are retained externally when truncated in API responses.
- Diff evidence identifies every changed file.
- Commits are blocked when required policy checks fail.

## 12. Phase 6 — Engineering-to-GitHub delivery loop

### 12.1 Workflow

```text
Create run
→ create workspace
→ inspect repository
→ apply patch
→ run checks
→ inspect diff
→ commit isolated branch
→ push branch
→ create pull request
→ retrieve CI state
→ report evidence
```

### 12.2 Integration design

Preferred boundary:

- Engineering Gateway owns local Git operations and workspace evidence.
- GitHub Gateway owns remote branch, pull request, review, and CI API operations.
- Engineering Gateway may call GitHub Gateway using an internal service credential, or Zoro may orchestrate both gateways explicitly.

Initial recommendation: Zoro orchestrates both gateways explicitly to keep service coupling low and evidence visible.

### 12.3 Required evidence

The final implementation report includes:

```text
runId
workspaceId
repository
baseRevision
branch
changedFiles
test results
build result
security scan result
commit SHA
pull request number and URL
CI state
remaining uncertainty
```

### 12.4 Tests

- end-to-end disposable repository flow;
- branch push after passing checks;
- branch push rejection after failing checks;
- pull-request creation idempotency;
- stale-base detection;
- cleanup after success;
- cleanup after failure;
- evidence linkage across both gateways.

### 12.5 Acceptance criteria

A fresh Zoro conversation can use the two Actions to implement a harmless change in a disposable repository, run verification, push a branch, and open a pull request without direct-main writes or merge authority.

## 13. Phase 7 — Deployment and GPT Action verification

### 13.1 Initial services

Deploy first:

1. GitHub Gateway;
2. Engineering Gateway web API;
3. Engineering worker.

### 13.2 Deployment configuration

Common commands:

```text
Build: npm ci
Start: npm start
```

Per-service variables:

```text
GitHub Gateway:
GATEWAY_APP=github
NODE_ENV=production
PUBLIC_BASE_URL=https://github.zoro.dev
ZORO_GATEWAY_API_KEY=<secret>
GITHUB_APP_ID=<secret>
GITHUB_INSTALLATION_ID=<secret>
GITHUB_PRIVATE_KEY=<secret>

Engineering Gateway:
GATEWAY_APP=engineering
NODE_ENV=production
PUBLIC_BASE_URL=https://engineering.zoro.dev
ZORO_GATEWAY_API_KEY=<different-secret>
MONGODB_URI=<secret>
WORKSPACE_ROOT=<path>
```

No real values belong in this repository.

### 13.3 Deployment checks

For each service:

1. deployment reports healthy;
2. `GET /health` succeeds;
3. `GET /ready` succeeds;
4. `GET /openapi.json` returns the production server URL;
5. unauthorized protected calls return `401`;
6. authenticated harmless reads succeed;
7. logs show request IDs and no secret leakage.

### 13.4 GPT Builder checks

1. Create one Action per gateway.
2. Configure each with its own Bearer key.
3. Import the matching OpenAPI schema.
4. Resolve schema validation issues without weakening route contracts.
5. Test each operation in Builder Preview.
6. Start a fresh standalone Zoro conversation.
7. Confirm Zoro can select the correct Action from a normal-language prompt.
8. Confirm operation output is summarized rather than dumped raw.
9. Confirm protected operations request approval rather than running.

### 13.5 Acceptance criteria

- Both Actions are callable from a fresh Zoro conversation.
- Action schemas use separate domains.
- Gateway credentials are distinct.
- The complete disposable engineering-to-PR workflow succeeds.
- Production and destructive actions remain blocked.

## 14. Phase 8 — Vercel Gateway extraction

### 14.1 Scope

Move Vercel provider execution out of Context API into `apps/vercel-gateway` while retaining temporary compatibility.

### 14.2 Read routes first

```http
GET /api/v1/projects
GET /api/v1/projects/:projectId
GET /api/v1/projects/:projectId/deployments
GET /api/v1/deployments/:deploymentId
GET /api/v1/deployments/:deploymentId/logs
GET /api/v1/projects/:projectId/domains
GET /api/v1/projects/:projectId/environment-variables
```

### 14.3 Write routes second

```http
POST /api/v1/deployments
POST /api/v1/projects/:projectId/environment-variables
POST /api/v1/deployments/:deploymentId/promote
POST /api/v1/deployments/:deploymentId/rollback
DELETE /api/v1/projects/:projectId/environment-variables/:variableId
```

### 14.4 Controls

- project allowlist;
- preview versus production classification;
- no decrypted environment values;
- explicit production approval;
- exact destructive confirmation;
- idempotency;
- evidence of deployment ID and target.

### 14.5 Migration sequence

1. Port adapter tests from current Context API behaviour.
2. Implement reads in the new gateway.
3. Deploy and connect the new Action.
4. Verify parity in a fresh Zoro conversation.
5. Implement one approved safe write against a disposable preview target.
6. Retain old routes temporarily.
7. Remove or deprecate old Context API gateway routes only after verification and explicit approval.

## 15. Phase 9 — Heroku Gateway

### 15.1 Read routes

```http
GET /api/v1/apps
GET /api/v1/apps/:appId
GET /api/v1/apps/:appId/releases
GET /api/v1/apps/:appId/dynos
GET /api/v1/apps/:appId/builds
GET /api/v1/apps/:appId/logs
GET /api/v1/apps/:appId/config
GET /api/v1/apps/:appId/addons
```

### 15.2 Write routes

```http
POST   /api/v1/apps/:appId/builds
POST   /api/v1/apps/:appId/dynos/restart
POST   /api/v1/apps/:appId/scale
POST   /api/v1/apps/:appId/releases/:releaseId/rollback
POST   /api/v1/apps/:appId/addons
DELETE /api/v1/apps/:appId/addons/:addonId
```

### 15.3 Controls

- app allowlist;
- config key metadata only;
- production scale approval;
- rollback confirmation;
- add-on deletion confirmation;
- rate-limit handling;
- bounded log retrieval.

### 15.4 Acceptance criteria

- Zoro can inspect the current Context API Heroku app without seeing config values.
- A harmless read-only deployment audit succeeds.
- The first write test uses a disposable or explicitly approved target.
- Rollback and deletion remain blocked without exact confirmation.

## 16. Phase 10 — Hardening and advanced engineering capabilities

### 16.1 Browser verification

Add Playwright-backed sessions:

```http
POST /api/v1/workspaces/:workspaceId/browser/session
POST /api/v1/browser/:sessionId/navigate
POST /api/v1/browser/:sessionId/interact
POST /api/v1/browser/:sessionId/assert
POST /api/v1/browser/:sessionId/screenshot
GET  /api/v1/browser/:sessionId/console
GET  /api/v1/browser/:sessionId/network
DELETE /api/v1/browser/:sessionId
```

### 16.2 Service runtime

Add managed local services:

```http
POST   /api/v1/workspaces/:workspaceId/services
GET    /api/v1/services/:serviceId
GET    /api/v1/services/:serviceId/logs
POST   /api/v1/services/:serviceId/restart
DELETE /api/v1/services/:serviceId
```

### 16.3 Database tooling

Add disposable database integration and migration controls:

```http
POST /api/v1/workspaces/:workspaceId/database/introspect
POST /api/v1/workspaces/:workspaceId/database/query
POST /api/v1/workspaces/:workspaceId/database/migrations/plan
POST /api/v1/workspaces/:workspaceId/database/migrations/apply
POST /api/v1/workspaces/:workspaceId/database/seed
POST /api/v1/workspaces/:workspaceId/database/reset
```

### 16.4 Worker hardening

- move from process-only worktrees to containers or equivalent sandboxes;
- restrict outbound networking;
- use CPU, memory, process, and disk quotas;
- enforce per-run budgets;
- add dead-letter handling;
- add retry policy by error category;
- add interrupted-run recovery;
- add cleanup reconciliation jobs;
- add worker leases and heartbeats.

### 16.5 Scaling

- separate API and worker deployments;
- queue work through a durable job store;
- use bounded concurrency;
- expose queue depth and worker health metrics;
- scale workers independently from API processes.

## 17. Test strategy

### 17.1 Test pyramid

```text
Many unit tests
→ route/service integration tests
→ provider contract tests
→ a small number of disposable end-to-end tests
```

### 17.2 Test naming

```text
*.unit.test.js
*.integration.test.js
*.contract.test.js
*.e2e.test.js
```

### 17.3 Coverage expectations

The first target is meaningful coverage rather than a vanity percentage. Critical policy, auth, path-safety, state-transition, redaction, and command-registry modules require branch coverage for both allowed and rejected paths.

### 17.4 Required regression suites

- authentication and credential separation;
- approval and confirmation gates;
- no-secret-return behaviour;
- direct-main prevention;
- workspace containment;
- command allowlist;
- idempotency;
- provider error normalization;
- OpenAPI route parity;
- cleanup behaviour.

## 18. CI and release strategy

### 18.1 CI jobs

Recommended jobs:

```text
workspace-validation
lint
unit-tests
integration-tests
openapi-validation
secret-scan
```

Later jobs:

```text
disposable-github-e2e
disposable-engineering-e2e
preview-vercel-e2e
non-production-heroku-e2e
```

### 18.2 Deployment strategy

Each service deploys from the same `main` revision. Deployment configuration selects the gateway with `GATEWAY_APP`.

A release is not considered verified merely because the platform reports a successful build. Verification requires health, readiness, authenticated smoke test, OpenAPI readback, and fresh Zoro Action testing.

### 18.3 Rollback

Provider gateways roll back to a previously verified deployment revision.

Engineering worker rollbacks must also preserve compatibility with persisted run schemas. Database migrations must be backward compatible or separately approved.

## 19. Data model outline

### 19.1 Engineering run

```json
{
  "runId": "run_123",
  "status": "running",
  "task": "Add password reset",
  "repository": "kofiarhin/example",
  "baseBranch": "main",
  "baseRevision": "abc123",
  "branch": "zoro/password-reset",
  "workspaceId": "ws_123",
  "requiredChecks": ["test", "build"],
  "approvalState": "not_required",
  "createdAt": "2026-07-25T00:00:00Z",
  "updatedAt": "2026-07-25T00:00:00Z"
}
```

### 19.2 Evidence

```json
{
  "evidenceId": "ev_123",
  "runId": "run_123",
  "type": "test-result",
  "status": "passed",
  "source": "engineering-gateway",
  "revision": "def456",
  "summary": "24 Jest tests passed",
  "artifactId": "artifact_123",
  "createdAt": "2026-07-25T00:00:00Z"
}
```

### 19.3 Approval

```json
{
  "approvalId": "approval_123",
  "runId": "run_123",
  "operation": "vercel.promoteDeployment",
  "target": "dep_123",
  "scope": "Promote dep_123 to production",
  "approvedBy": "user",
  "status": "approved",
  "expiresAt": "2026-07-25T01:00:00Z"
}
```

## 20. Environment-variable plan

### 20.1 Shared

```text
NODE_ENV
PORT
PUBLIC_BASE_URL
ZORO_GATEWAY_API_KEY
LOG_LEVEL
REQUEST_BODY_LIMIT
RATE_LIMIT_WINDOW_MS
RATE_LIMIT_MAX
```

### 20.2 GitHub Gateway

```text
GITHUB_APP_ID
GITHUB_INSTALLATION_ID
GITHUB_PRIVATE_KEY
GITHUB_ALLOWED_REPOSITORIES
```

### 20.3 Engineering Gateway

```text
MONGODB_URI
WORKSPACE_ROOT
WORKSPACE_TTL_MINUTES
MAX_ACTIVE_WORKSPACES
COMMAND_OUTPUT_LIMIT_BYTES
COMMAND_DEFAULT_TIMEOUT_MS
ARTIFACT_STORAGE_PROVIDER
ARTIFACT_STORAGE_BUCKET
```

### 20.4 Vercel Gateway

```text
VERCEL_ACCESS_TOKEN
VERCEL_TEAM_ID
VERCEL_ALLOWED_PROJECTS
```

### 20.5 Heroku Gateway

```text
HEROKU_API_TOKEN
HEROKU_ALLOWED_APPS
```

All `.env.example` files contain empty placeholders and descriptions only.

## 21. Risk register

| Risk | Impact | Mitigation |
| --- | --- | --- |
| Arbitrary command execution | Host compromise | Named command registry, argument validation, isolation, quotas |
| Workspace path escape | Cross-run data access | Canonical paths, root containment checks, tests |
| Secret leakage | Credential compromise | Redaction, metadata-only config reads, secret scans |
| Direct-main mutation | Unreviewed production change | Default rejection, explicit authority gate |
| Provider credential overreach | Broad account damage | Scoped provider tokens and allowlists |
| Long GPT Action request | Timeout or duplicate work | Async runs, idempotency, status polling |
| Ephemeral filesystem loss | Missing evidence | MongoDB and object storage persistence |
| Worker crash | Orphaned workspaces | leases, cleanup reconciliation, expiry |
| Stale base revision | Conflicting implementation | audited revision and pre-push revalidation |
| OpenAPI model confusion | Wrong operation selected | bounded schemas, clear operation IDs, direct common endpoints |
| Duplicate deployment writes | Multiple mutations | idempotency keys and provider reconciliation |
| Unverified success claims | Incorrect project state | evidence requirements and independent verification |

## 22. Definition of done by milestone

### Milestone A — Monorepo foundation

- clean installation;
- all health routes pass;
- shared packages tested;
- CI passing;
- production selector works.

### Milestone B — GitHub read Action

- deployed unique URL;
- authenticated repository reads;
- valid OpenAPI schema;
- fresh Zoro conversation verified.

### Milestone C — Engineering MVP

- durable run state;
- isolated workspace;
- file read and patch;
- allowlisted tests and builds;
- Git diff and commit;
- cleanup verified.

### Milestone D — Full engineering delivery loop

- isolated branch push;
- pull-request creation;
- CI retrieval;
- linked evidence;
- fresh Zoro conversation completes disposable workflow.

### Milestone E — Deployment gateways

- Vercel Action verified;
- Heroku Action verified;
- no-secret-return checks pass;
- protected writes remain gated.

## 23. Recommended execution order

The exact recommended implementation sequence is:

1. Scaffold root workspace and all package manifests.
2. Add shared response contracts and tests.
3. Add authentication and request ID middleware with tests.
4. Add policy engine and protected-operation tests.
5. Add minimal apps, health, readiness, and OpenAPI endpoints.
6. Add CI and OpenAPI validation.
7. Implement GitHub App client and repository reads.
8. Deploy GitHub Gateway and verify the Action.
9. Add branch, pull-request, and CI operations.
10. Add MongoDB-backed Engineering runs.
11. Add Git worktree provisioning and cleanup.
12. Add tree, file read, search, patch, and batch writes.
13. Add command registry, tests, builds, and Git diff.
14. Add commit and branch push controls.
15. Run the disposable engineering-to-PR demonstration.
16. Deploy Engineering Gateway and worker and verify the Action.
17. Extract Vercel reads, then one approved safe write.
18. Add Heroku reads, then one approved safe write.
19. Add browser, service runtime, and database tooling.
20. Harden isolation and scale workers.

## 24. Decisions recorded by this plan

- The system uses one npm-workspaces monorepo.
- Each gateway is independently deployed.
- Each gateway receives a separate GPT Action and Bearer credential.
- The initial module system is CommonJS.
- GitHub Gateway is implemented before Vercel and Heroku extraction.
- Engineering and GitHub Gateways form the first complete full-stack delivery loop.
- Engineering operations use asynchronous durable runs.
- Git worktrees are the MVP isolation method, with stronger sandboxing planned.
- Jest and Supertest are the backend testing defaults.
- Provider SDK logic is isolated behind adapters.
- Protected operations fail closed.
- Context API remains separate and continues to own structured context.

## 25. Approvals still required later

This plan does not itself authorize:

- production deployment of any gateway;
- creation or rotation of real credentials;
- destructive provider tests;
- production promotion or rollback;
- database migration application;
- pull-request merge;
- direct-main source implementation outside separately authorized work;
- removal of existing provider routes from Context API.

Those operations require explicit authority at execution time.