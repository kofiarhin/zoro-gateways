# Zoro Gateways — Technical Specification

**Status:** Approved architecture specification  
**Version:** 1.0.0  
**Date:** 2026-07-25  
**Repository:** `kofiarhin/zoro-gateways`  
**Default branch:** `main`

## 1. Purpose

Zoro Gateways provides independently deployable Express applications that expose safe, well-described GPT Action surfaces for software engineering and provider integrations.

The system exists to remove the current pressure to place GitHub, Vercel, Heroku, Context API, and engineering capabilities behind one constrained Action schema. Each gateway receives its own deployment URL, authentication credential, OpenAPI contract, operation namespace, and release lifecycle.

The intended result is a Zoro Custom GPT that can operate as a governed full-stack engineer by combining:

- project context and requirements;
- isolated repository workspaces;
- file inspection and modification;
- dependency installation;
- command execution;
- frontend, backend, API, browser, database, and build verification;
- GitHub branch and pull-request workflows;
- Vercel and Heroku deployment operations;
- durable evidence and execution reporting.

## 2. Goals

### 2.1 Primary goals

1. Give Zoro a complete full-stack engineering execution surface.
2. Deploy each gateway as a separate application with its own HTTPS base URL.
3. Register each gateway as a separate GPT Action using its own OpenAPI schema.
4. Preserve explicit security boundaries between read, workspace-write, external-write, production-affecting, and destructive operations.
5. Use one npm-workspaces monorepo for shared code without coupling deployments.
6. Support asynchronous engineering operations that may exceed a single GPT Action request window.
7. Produce traceable evidence for every meaningful operation.
8. Keep provider secrets, decrypted environment values, private keys, and credentials out of API responses and repository files.

### 2.2 Secondary goals

- Reuse shared authentication, policy, error, logging, validation, and response packages.
- Support local development of one gateway or all gateways.
- Allow each gateway to scale and deploy independently.
- Make operation selection reliable through clear operation IDs and bounded Action surfaces.
- Retain compatibility with the current CommonJS-based Zoro service during migration.

## 3. Non-goals

The first release will not:

- provide unrestricted remote shell access;
- merge pull requests automatically without explicit authority;
- write directly to protected default branches by default;
- return decrypted provider environment-variable values;
- execute destructive database operations without exact confirmation;
- permit workers to approve or verify their own work;
- replace Ideas Hub as durable human-readable project context;
- replace Context API as structured machine context;
- require all gateways to use the same hosting provider;
- implement an autonomous background agent that continues indefinitely without scheduled infrastructure.

## 4. System boundaries

### 4.1 Zoro Custom GPT

Zoro is the user-facing orchestrator. It interprets requests, loads context, selects gateway operations, requests approvals when necessary, aggregates evidence, and reports results.

Zoro does not directly hold provider credentials other than the gateway credentials configured in GPT Actions.

### 4.2 Zoro Gateways repository

This repository contains independently deployable gateway applications and shared packages:

```text
zoro-gateways/
├── apps/
│   ├── engineering-gateway/
│   ├── github-gateway/
│   ├── vercel-gateway/
│   └── heroku-gateway/
├── packages/
│   ├── gateway-core/
│   ├── policy-engine/
│   ├── contracts/
│   └── testing/
├── scripts/
├── docs/
├── package.json
├── package-lock.json
├── Procfile
└── README.md
```

### 4.3 Context API

Context API remains a separate service and repository for structured project context, decisions, requirements, evidence, relationships, and durable execution records.

Context API is not required to proxy GitHub, Vercel, Heroku, or engineering-runtime traffic after migration.

### 4.4 Ideas Hub

Ideas Hub remains the durable human-readable project brain and governance layer. It stores project records, decisions, assumptions, open questions, approved specifications, coordination records, and verification history.

## 5. Deployment topology

Each gateway is deployed separately from the same repository.

Recommended public topology:

```text
https://engineering.zoro.dev  → Engineering Gateway
https://github.zoro.dev       → GitHub Gateway
https://vercel.zoro.dev       → Vercel Gateway
https://heroku.zoro.dev       → Heroku Gateway
```

The existing Context API remains available at its independently managed URL.

Each deployment must expose:

```text
GET /health
GET /openapi.json
/api/v1/*
```

Each deployment uses the same root start command:

```bash
npm start
```

The selected application is controlled by:

```env
GATEWAY_APP=engineering|github|vercel|heroku
```

## 6. Package and runtime architecture

### 6.1 Root workspace

The repository uses npm workspaces:

```json
{
  "private": true,
  "type": "commonjs",
  "workspaces": ["apps/*", "packages/*"]
}
```

The root package owns:

- workspace installation;
- local multi-app development;
- per-app development scripts;
- production gateway selection;
- workspace-wide tests;
- linting and formatting;
- CI entrypoints.

### 6.2 Gateway packages

Every gateway package contains:

```text
src/
├── app.js
├── server.js
├── config/
├── middleware/
├── routes/
├── controllers/
├── services/
├── validators/
└── adapters/
tests/
openapi.json
.env.example
package.json
```

`app.js` creates and configures the Express application without opening a network socket. `server.js` loads environment configuration, opens the socket, and handles graceful shutdown.

### 6.3 Shared packages

#### `@zoro/gateway-core`

Owns:

- Bearer authentication middleware;
- request ID generation;
- common error types;
- response envelope creation;
- async route wrappers;
- health and readiness helpers;
- redaction helpers;
- structured logging primitives.

#### `@zoro/policy-engine`

Owns:

- operation classification;
- repository and project allowlists;
- production-target checks;
- approval validation;
- destructive confirmation validation;
- direct-main protection;
- secret-return prevention;
- scope evaluation.

#### `@zoro/contracts`

Owns shared schemas for:

- success and error responses;
- evidence records;
- approval requirements;
- asynchronous jobs;
- pagination;
- operation metadata;
- health responses.

#### `@zoro/testing`

Owns:

- Supertest helpers;
- fake authentication middleware;
- provider adapter mocks;
- response contract assertions;
- approval-policy test fixtures.

## 7. Shared HTTP contract

### 7.1 Success envelope

```json
{
  "ok": true,
  "operation": "github.listRepositories",
  "summary": "Retrieved 12 repositories.",
  "data": {},
  "evidence": [],
  "approval": {
    "required": false,
    "type": null
  },
  "warnings": [],
  "requestId": "req_123"
}
```

### 7.2 Error envelope

```json
{
  "ok": false,
  "operation": "github.mergePullRequest",
  "error": {
    "code": "EXPLICIT_APPROVAL_REQUIRED",
    "message": "Pull request merging requires explicit user authority.",
    "retryable": false,
    "details": null
  },
  "requestId": "req_123"
}
```

### 7.3 Asynchronous acceptance envelope

```json
{
  "ok": true,
  "operation": "engineering.createRun",
  "summary": "Engineering run accepted.",
  "data": {
    "runId": "run_123",
    "status": "queued",
    "statusUrl": "/api/v1/runs/run_123"
  },
  "evidence": [],
  "approval": {
    "required": false,
    "type": null
  },
  "warnings": [],
  "requestId": "req_123"
}
```

### 7.4 Request IDs

Every request receives an `X-Request-ID` response header and matching `requestId` response field. A caller-provided request ID may be accepted only when it satisfies the documented format.

### 7.5 Idempotency

External mutations must support an `Idempotency-Key` header or an equivalent body field. Duplicate operations must return the original result or a deterministic conflict response.

## 8. Authentication and authorization

### 8.1 Gateway authentication

Each gateway has a distinct Bearer credential:

```text
ZORO_ENGINEERING_GATEWAY_KEY
ZORO_GITHUB_GATEWAY_KEY
ZORO_VERCEL_GATEWAY_KEY
ZORO_HEROKU_GATEWAY_KEY
```

No credential is committed to Git. Values are configured in the deployment platform and GPT Action authentication settings.

### 8.2 Provider authentication

- GitHub Gateway uses GitHub App installation authentication.
- Vercel Gateway uses a scoped Vercel access token.
- Heroku Gateway uses an appropriately scoped Heroku Platform API token.
- Engineering Gateway uses provider credentials only through narrowly scoped adapters and should prefer the GitHub Gateway for remote repository writes.

### 8.3 Permission classes

Every operation is assigned one class:

| Class | Examples | Default behaviour |
| --- | --- | --- |
| `read` | inspect files, list deployments, fetch logs | automatic |
| `workspace-write` | patch isolated files, install dependencies | allowed within approved run scope |
| `external-write` | push branch, create PR, set preview variable | requires recorded task authority |
| `production-write` | promote deployment, scale production dynos | explicit user approval |
| `destructive` | delete resource, reset database, remove add-on | exact confirmation plus approval |
| `merge` | merge pull request | explicit user authority |
| `direct-main` | write directly to default branch | explicit user authority and policy validation |

### 8.4 Approval payload

Protected operations must accept a structured approval object:

```json
{
  "approval": {
    "approved": true,
    "approvedBy": "user",
    "scope": "promote deployment dep_123 to production",
    "confirmation": "PROMOTE dep_123 TO PRODUCTION"
  }
}
```

The policy engine validates scope, target, operation, confirmation text, and expiry where applicable.

## 9. Engineering Gateway specification

### 9.1 Purpose

The Engineering Gateway provides isolated, auditable development environments where Zoro can inspect, modify, execute, test, and verify full-stack applications.

It must not expose arbitrary host-level shell access.

### 9.2 Runtime model

The public Express application receives commands and stores durable run state. Worker processes execute jobs inside isolated workspaces.

```text
Zoro GPT
  → Engineering API
  → durable run/job store
  → engineering worker
  → isolated worktree or container
  → evidence and artifacts
```

### 9.3 Run states

```text
queued
planning
waiting_for_approval
provisioning
running
verifying
succeeded
failed
cancelled
expired
```

A run may not move to `succeeded` until required checks and evidence are complete.

### 9.4 Run endpoints

```http
POST   /api/v1/runs
GET    /api/v1/runs/:runId
POST   /api/v1/runs/:runId/cancel
POST   /api/v1/runs/:runId/approve
GET    /api/v1/runs/:runId/events
GET    /api/v1/runs/:runId/artifacts
```

### 9.5 Workspace endpoints

```http
POST   /api/v1/workspaces
GET    /api/v1/workspaces/:workspaceId
DELETE /api/v1/workspaces/:workspaceId
POST   /api/v1/workspaces/:workspaceId/reset
POST   /api/v1/workspaces/:workspaceId/snapshot
```

A workspace records:

- repository;
- base ref and audited revision;
- isolated branch;
- owning run;
- working directory;
- creation and expiry times;
- status;
- applied patches;
- commands;
- artifacts;
- cleanup result.

### 9.6 Repository inspection endpoints

```http
GET  /api/v1/workspaces/:workspaceId/tree
GET  /api/v1/workspaces/:workspaceId/file
POST /api/v1/workspaces/:workspaceId/search
POST /api/v1/workspaces/:workspaceId/symbols
POST /api/v1/workspaces/:workspaceId/dependencies/analyse
POST /api/v1/workspaces/:workspaceId/routes/analyse
```

Path validation must prevent traversal outside the workspace.

### 9.7 File mutation endpoints

```http
PUT    /api/v1/workspaces/:workspaceId/file
POST   /api/v1/workspaces/:workspaceId/patch
POST   /api/v1/workspaces/:workspaceId/files/batch
DELETE /api/v1/workspaces/:workspaceId/file
```

Every mutation returns:

- changed paths;
- patch or diff ID;
- resulting content hashes;
- validation warnings;
- secret-scan result.

Batch writes must be atomic where practical. A failed operation must not silently leave an unknown partial state.

### 9.8 Command endpoints

```http
POST /api/v1/workspaces/:workspaceId/commands
GET  /api/v1/commands/:commandId
POST /api/v1/commands/:commandId/cancel
GET  /api/v1/commands/:commandId/logs
```

Commands are selected from an allowlisted registry. The request supplies a command identifier and structured arguments, not arbitrary shell text.

Initial command registry:

```text
npm-install
npm-ci
npm-test
npm-run
npm-audit
eslint
prettier-check
vitest
jest
tsc
vite-build
node-script
git-status
git-diff
git-log
```

Each command has:

- executable and fixed argument template;
- allowed working-directory policy;
- timeout;
- output limit;
- environment allowlist;
- cancellation behaviour;
- permission class.

### 9.9 Dependency endpoints

```http
GET  /api/v1/workspaces/:workspaceId/dependencies
POST /api/v1/workspaces/:workspaceId/dependencies/install
POST /api/v1/workspaces/:workspaceId/dependencies/remove
POST /api/v1/workspaces/:workspaceId/dependencies/update
POST /api/v1/workspaces/:workspaceId/dependencies/audit
```

Dependency operations must record exact package versions and lockfile changes.

### 9.10 Check endpoints

```http
POST /api/v1/workspaces/:workspaceId/checks/lint
POST /api/v1/workspaces/:workspaceId/checks/typecheck
POST /api/v1/workspaces/:workspaceId/checks/test
POST /api/v1/workspaces/:workspaceId/checks/build
POST /api/v1/workspaces/:workspaceId/checks/security
POST /api/v1/workspaces/:workspaceId/checks/all
```

A check result records:

- command;
- revision;
- start and finish times;
- exit code;
- status;
- bounded logs;
- failing test names or diagnostics;
- artifact references.

### 9.11 Service runtime endpoints

```http
POST   /api/v1/workspaces/:workspaceId/services
GET    /api/v1/services/:serviceId
GET    /api/v1/services/:serviceId/logs
POST   /api/v1/services/:serviceId/restart
DELETE /api/v1/services/:serviceId
```

A service request uses a named, allowlisted package script and an allocated internal port. Services must run inside the workspace isolation boundary.

### 9.12 HTTP verification endpoints

```http
POST /api/v1/workspaces/:workspaceId/http/request
POST /api/v1/workspaces/:workspaceId/http/collection
POST /api/v1/workspaces/:workspaceId/http/health-check
```

The gateway must restrict outbound requests to the active workspace, approved local service URLs, or allowlisted test targets.

### 9.13 Browser verification endpoints

```http
POST   /api/v1/workspaces/:workspaceId/browser/session
POST   /api/v1/browser/:sessionId/navigate
POST   /api/v1/browser/:sessionId/interact
POST   /api/v1/browser/:sessionId/assert
POST   /api/v1/browser/:sessionId/screenshot
GET    /api/v1/browser/:sessionId/console
GET    /api/v1/browser/:sessionId/network
DELETE /api/v1/browser/:sessionId
```

Playwright is the recommended internal browser automation engine.

Browser actions must use structured action types such as `navigate`, `click`, `fill`, `select`, `assertText`, `assertVisible`, and `screenshot`.

### 9.14 Database endpoints

```http
POST /api/v1/workspaces/:workspaceId/database/introspect
POST /api/v1/workspaces/:workspaceId/database/query
POST /api/v1/workspaces/:workspaceId/database/migrations/plan
POST /api/v1/workspaces/:workspaceId/database/migrations/apply
POST /api/v1/workspaces/:workspaceId/database/seed
POST /api/v1/workspaces/:workspaceId/database/reset
```

Database access must default to disposable test databases. Migration application requires approval. Reset is destructive and requires exact confirmation.

### 9.15 Local Git endpoints

```http
GET  /api/v1/workspaces/:workspaceId/git/status
GET  /api/v1/workspaces/:workspaceId/git/diff
GET  /api/v1/workspaces/:workspaceId/git/history
POST /api/v1/workspaces/:workspaceId/git/branch
POST /api/v1/workspaces/:workspaceId/git/stage
POST /api/v1/workspaces/:workspaceId/git/commit
POST /api/v1/workspaces/:workspaceId/git/push
```

Push must be rejected when:

- the branch is the protected default branch without explicit direct-main authority;
- required tests have not passed;
- secret scanning fails;
- the repository is outside the approved allowlist;
- the audited base revision has become materially stale without revalidation.

## 10. GitHub Gateway specification

### 10.1 Purpose

The GitHub Gateway exposes governed GitHub App-backed repository, branch, content, issue, pull-request, review, Actions, release, and configuration operations.

### 10.2 Initial endpoints

```http
GET  /api/v1/repositories
GET  /api/v1/repositories/:owner/:repo
GET  /api/v1/repositories/:owner/:repo/contents
GET  /api/v1/repositories/:owner/:repo/branches
GET  /api/v1/repositories/:owner/:repo/commits

GET  /api/v1/repositories/:owner/:repo/issues
POST /api/v1/repositories/:owner/:repo/issues
PATCH /api/v1/repositories/:owner/:repo/issues/:number
POST /api/v1/repositories/:owner/:repo/issues/:number/comments

GET  /api/v1/repositories/:owner/:repo/pulls
POST /api/v1/repositories/:owner/:repo/pulls
GET  /api/v1/repositories/:owner/:repo/pulls/:number
GET  /api/v1/repositories/:owner/:repo/pulls/:number/files
GET  /api/v1/repositories/:owner/:repo/pulls/:number/reviews
POST /api/v1/repositories/:owner/:repo/pulls/:number/comments
POST /api/v1/repositories/:owner/:repo/pulls/:number/merge

GET  /api/v1/repositories/:owner/:repo/actions/runs
GET  /api/v1/repositories/:owner/:repo/actions/runs/:runId
GET  /api/v1/repositories/:owner/:repo/actions/runs/:runId/jobs
GET  /api/v1/repositories/:owner/:repo/actions/jobs/:jobId/logs
POST /api/v1/repositories/:owner/:repo/actions/runs/:runId/rerun
```

### 10.3 GitHub operation rules

- Reads may run automatically within repository scope.
- Branch creation and pull-request creation require approved work scope.
- Merge requires explicit user authority.
- Direct default-branch writes require explicit direct-main authority.
- Provider tokens and private keys must never be returned.
- Repository access must be constrained by GitHub App installation permissions and optional repository allowlists.

## 11. Vercel Gateway specification

### 11.1 Purpose

The Vercel Gateway exposes project, deployment, domain, environment metadata, build, log, promotion, rollback, and configuration operations.

### 11.2 Initial endpoints

```http
GET    /api/v1/projects
GET    /api/v1/projects/:projectId
GET    /api/v1/projects/:projectId/deployments
GET    /api/v1/deployments/:deploymentId
GET    /api/v1/deployments/:deploymentId/logs
POST   /api/v1/deployments
POST   /api/v1/deployments/:deploymentId/promote
POST   /api/v1/deployments/:deploymentId/rollback
GET    /api/v1/projects/:projectId/domains
GET    /api/v1/projects/:projectId/environment-variables
POST   /api/v1/projects/:projectId/environment-variables
DELETE /api/v1/projects/:projectId/environment-variables/:variableId
```

### 11.3 Vercel operation rules

- Environment values are never returned decrypted.
- Preview deployment creation is an external write.
- Production promotion is a production write.
- Rollback and deletion operations require exact confirmation.
- Project allowlisting applies to every mutation.
- Existing read/write/destructive dispatcher logic may be migrated temporarily, but the maintained Action schema should prefer descriptive direct operations for common workflows.

## 12. Heroku Gateway specification

### 12.1 Purpose

The Heroku Gateway exposes application, pipeline, release, dyno, build, add-on, config metadata, log, deployment, scale, and rollback operations.

### 12.2 Initial endpoints

```http
GET    /api/v1/apps
GET    /api/v1/apps/:appId
GET    /api/v1/apps/:appId/releases
GET    /api/v1/apps/:appId/dynos
GET    /api/v1/apps/:appId/builds
GET    /api/v1/apps/:appId/logs
GET    /api/v1/apps/:appId/config
POST   /api/v1/apps/:appId/builds
POST   /api/v1/apps/:appId/releases/:releaseId/rollback
POST   /api/v1/apps/:appId/dynos/restart
POST   /api/v1/apps/:appId/scale
GET    /api/v1/apps/:appId/addons
POST   /api/v1/apps/:appId/addons
DELETE /api/v1/apps/:appId/addons/:addonId
```

### 12.3 Heroku operation rules

- Config responses expose keys and metadata only.
- Build creation and dyno restart are external writes.
- Production scaling requires explicit approval.
- Rollback and add-on deletion require exact confirmation.
- App allowlisting applies to every mutation.

## 13. OpenAPI and GPT Action requirements

Each gateway owns one OpenAPI 3.1 schema available at `/openapi.json`.

Requirements:

- one stable `operationId` per operation;
- operation IDs namespaced by gateway;
- concise summaries written for model tool selection;
- explicit request and response schemas;
- no unsupported circular references;
- no duplicate operation IDs across Actions;
- documented security scheme;
- production server URL generated from `PUBLIC_BASE_URL` or maintained during release;
- schemas validated in CI;
- schema operation count monitored;
- mutation operations clearly describe approval requirements.

Recommended operation ID format:

```text
engineeringCreateRun
githubListRepositories
vercelGetDeployment
herokuRestartDynos
```

## 14. Persistence and artifacts

### 14.1 Durable records

Engineering run, job, command, approval, evidence, and artifact metadata must persist outside the ephemeral application filesystem.

Recommended first implementation:

- MongoDB for run and job metadata;
- object storage for screenshots, full logs, patches, and large artifacts;
- GitHub branches and commits for source changes.

### 14.2 Artifact types

```text
patch
diff
command-log
test-result
build-result
screenshot
browser-console
network-log
security-scan
coverage-report
implementation-report
```

### 14.3 Retention

Retention periods must be configurable. Secrets and raw credentials must never be stored in artifacts.

## 15. Logging and observability

Every application must log structured JSON with:

- timestamp;
- service;
- environment;
- request ID;
- operation;
- permission class;
- target resource identifier;
- result status;
- duration;
- approval state;
- error code;
- redaction indicator.

Metrics should include:

- request count and latency;
- error count by code;
- provider rate-limit usage;
- queued and active jobs;
- command duration;
- workspace count;
- failed cleanup count;
- approval wait duration.

Health endpoints:

```http
GET /health
GET /ready
```

`/health` confirms process liveness. `/ready` confirms dependencies required to accept work.

## 16. Security requirements

1. All public traffic uses HTTPS.
2. Gateway Bearer credentials are distinct per service.
3. Provider credentials remain server-side only.
4. Request bodies have strict size limits.
5. Inputs are validated with explicit schemas.
6. File paths are canonicalized and constrained to the workspace.
7. Arbitrary shell commands are prohibited.
8. Command environment variables use allowlists.
9. Logs and responses pass through secret redaction.
10. Outbound networking is restricted where practical.
11. Workspaces run with limited filesystem and process permissions.
12. Production and destructive operations use explicit policy gates.
13. Default-branch writes are blocked unless direct-main authority is explicit.
14. Every external mutation supports idempotency.
15. Rate limiting applies per credential and operation class.
16. Dependency and command timeouts are enforced.
17. Workspace expiration and cleanup are mandatory.
18. Security scans run before external source publication.

## 17. Testing requirements

### 17.1 Unit tests

Use Jest for backend packages and gateways.

Required unit coverage includes:

- authentication middleware;
- policy classification;
- approval validation;
- path safety;
- secret redaction;
- response contracts;
- provider adapter transformations;
- command registry validation;
- run-state transitions.

### 17.2 Integration tests

Use Supertest with provider adapters mocked at the network boundary.

Every route must test:

- success;
- unauthorized access;
- invalid input;
- provider failure;
- policy rejection;
- response contract shape.

### 17.3 Contract tests

- Validate every OpenAPI document.
- Confirm every documented route is registered.
- Confirm every response follows the shared envelope.
- Confirm operation IDs are unique.
- Confirm protected operations document approval fields.

### 17.4 End-to-end tests

Initial end-to-end tests must use disposable resources:

- a disposable or test repository;
- a preview Vercel project;
- a non-production Heroku app;
- a disposable engineering workspace and database.

No destructive production smoke test is permitted without explicit approval and a cleanup plan.

## 18. CI requirements

GitHub Actions should run on pull requests and pushes to `main`:

```text
npm ci
npm test
npm run lint
npm run openapi:validate
npm run build --if-present
```

CI should also:

- detect committed secrets;
- verify package-lock consistency;
- validate workspace package resolution;
- report per-gateway test results;
- prevent deployment when required checks fail.

## 19. Deployment requirements

### 19.1 Shared repository, separate services

Every service is created from the same repository and revision.

Common build command:

```bash
npm ci
```

Common start command:

```bash
npm start
```

Per-service environment:

```text
Engineering → GATEWAY_APP=engineering
GitHub      → GATEWAY_APP=github
Vercel      → GATEWAY_APP=vercel
Heroku      → GATEWAY_APP=heroku
```

### 19.2 Host guidance

Provider gateways can run on Heroku, Railway, Render, Fly.io, or equivalent Node hosting.

The Engineering Gateway requires a runtime that supports:

- longer-running workers;
- process execution;
- writable isolated storage during a job;
- Git;
- Node and package managers;
- optional browser dependencies;
- worker scaling;
- durable external persistence.

It should not rely on a short-lived serverless function runtime for execution-heavy jobs.

### 19.3 Graceful shutdown

Every app must stop accepting new work, finish or requeue active requests where safe, close provider clients, and terminate within the platform shutdown window.

## 20. Migration strategy

1. Build and deploy the shared gateway foundation.
2. Implement GitHub Gateway read operations.
3. Connect the GitHub Gateway Action to Zoro and verify it in a fresh conversation.
4. Implement Engineering Gateway MVP for isolated read/edit/test/diff workflows.
5. Add branch push and pull-request creation through the GitHub Gateway.
6. Extract Vercel gateway operations from Context API while retaining temporary compatibility.
7. Implement the Heroku Gateway.
8. Remove provider gateway routes from Context API only after the replacement Actions are verified.
9. Keep durable Context API and Ideas Hub responsibilities unchanged.

## 21. Acceptance criteria

The architecture is considered operational when:

1. Each gateway deploys independently from the monorepo.
2. Each deployment has a unique HTTPS URL.
3. `/health` and `/openapi.json` work for every gateway.
4. Each Action authenticates with a distinct Bearer credential.
5. Zoro can select and call each Action in a fresh conversation.
6. GitHub reads work through GitHub App authentication.
7. Engineering Gateway can provision an isolated workspace.
8. Zoro can inspect, patch, test, build, and verify a sample MERN application.
9. Zoro can push an isolated branch and open a pull request after checks pass.
10. Vercel and Heroku reads work without revealing secret values.
11. Production and destructive requests are blocked without required approval.
12. All route, policy, contract, and OpenAPI tests pass in CI.
13. Evidence links the request, run, workspace, revision, command results, branch, and pull request.
14. Workspace cleanup is verified.

## 22. Open questions

1. Which hosting provider will run the Engineering Gateway and workers?
2. Will all gateways initially use Heroku, or will provider gateways use a lighter host?
3. Which object-storage provider will retain screenshots and large artifacts?
4. Should Engineering Gateway persistence use its own MongoDB database or Context API?
5. Which disposable repository and deployment targets will be used for end-to-end tests?
6. What retention period applies to engineering artifacts and command logs?
7. What maximum concurrent workspace count is appropriate for the first production release?
8. Which Vercel write operation will be the first approved live smoke test?
9. Which Heroku write operation will be the first approved live smoke test?

## 23. Recommended first release

The first production release should contain:

- root npm-workspaces structure;
- `gateway-core`, `policy-engine`, `contracts`, and `testing` packages;
- GitHub Gateway health, OpenAPI, authentication, repository listing, repository metadata, file read, branch listing, and pull-request creation;
- Engineering Gateway run creation, workspace provisioning, tree inspection, file read, patch, allowlisted command execution, `checks/all`, Git diff, commit, and branch push;
- MongoDB-backed run metadata;
- Jest, Supertest, OpenAPI validation, secret scanning, and CI;
- separate GitHub and Engineering deployments;
- verified GPT Actions for both deployments.

Vercel and Heroku extraction should follow after this foundation is proven.