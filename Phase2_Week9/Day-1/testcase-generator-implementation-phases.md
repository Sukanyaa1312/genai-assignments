# Testcase Generator - Phased Implementation Plan

## 1. Purpose and Scope

This document converts `testcase-generator-architecture.md` into an incremental delivery plan. It is the implementation contract for a modular monolith that can later extract high-volume modules into services without changing domain contracts.

The plan assumes:

- React web application and Node.js/TypeScript/Express API.
- MongoDB for operational metadata, source chunks, vector search, and GridFS files.
- Mistral embeddings with versioned source-specific configuration.
- Provider-neutral LLM routing for OpenAI, Groq, and Anthropic.
- REST APIs under `/api/v1`.
- Asynchronous processing for uploads, synchronization, embeddings, generation, and exports.
- Human approval is required before publication or export.

### Delivery principles

1. Every write is tenant- and project-scoped and is authorized at resource level.
2. Long-running operations return `202 Accepted` with a job resource.
3. Every response includes a correlation ID; every asynchronous job carries it forward.
4. Source lineage and evidence are first-class data, not UI-only metadata.
5. Retries are bounded and idempotent. Duplicate requests must not duplicate domain records.
6. APIs expose stable domain contracts; vendor SDKs remain behind provider interfaces.
7. A phase is complete only when its migration, observability, security, and failure behavior are tested.

## 2. Cross-Cutting API Contract

### 2.1 Authentication and headers

Required on all protected endpoints:

```http
Authorization: Bearer <OIDC access token>
X-Tenant-Id: <tenant id>
X-Correlation-Id: <client-generated UUID, optional>
Idempotency-Key: <unique key, required for retryable POST writes>
```

`X-Tenant-Id` must match the tenant in the token. The server may derive tenant context from the token and reject a conflicting header. `Idempotency-Key` is scoped to tenant, actor, route, and request body hash, and is retained for at least 24 hours.

### 2.2 Success envelopes

Synchronous response:

```json
{
  "data": {},
  "meta": { "correlationId": "9d...", "requestId": "req_..." }
}
```

Collection response:

```json
{
  "data": [],
  "meta": {
    "correlationId": "9d...",
    "requestId": "req_...",
    "page": { "limit": 25, "nextCursor": "..." }
  }
}
```

Asynchronous response:

```json
{
  "data": {
    "jobId": "job_123",
    "status": "QUEUED",
    "statusUrl": "/api/v1/ingestion/jobs/job_123"
  },
  "meta": { "correlationId": "9d...", "requestId": "req_..." }
}
```

### 2.3 Error envelope

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "One or more fields are invalid.",
    "details": [
      { "field": "name", "reason": "must not be empty" }
    ],
    "retryable": false,
    "correlationId": "9d..."
  }
}
```

Standard status mapping:

| Status | Use |
|---|---|
| `400` | Malformed JSON, invalid query, invalid state transition |
| `401` | Missing, expired, or invalid token |
| `403` | Authenticated but not authorized; do not leak resource existence |
| `404` | Resource not found or hidden by authorization policy |
| `409` | Duplicate, stale version, active job, or conflicting state |
| `413` | File or request exceeds configured limit |
| `415` | Unsupported media type or file type |
| `422` | Valid syntax but unsupported or semantically invalid content |
| `429` | Rate or provider quota exceeded; include `Retry-After` |
| `500` | Unexpected server failure |
| `502/503/504` | External provider or dependency failure |

Never return stack traces, access tokens, connector secrets, prompts containing secrets, or cross-tenant identifiers.

### 2.4 Common resource fields

All persisted resources should include `id`, `tenantId`, `projectId` where applicable, `createdAt`, `updatedAt`, and `version`. Dates are ISO-8601 UTC strings. Optimistic updates use `If-Match: <version>` or a request `version`; stale writes return `409 STALE_VERSION`.

## 3. Target Domain States

### Jobs

`QUEUED -> RUNNING -> SUCCEEDED | FAILED | CANCEL_REQUESTED -> CANCELLED`.
A terminal job cannot be retried by mutation; retry creates a new job linked with `parentJobId`.

### Integrations

`PENDING -> CONNECTED -> DEGRADED | DISCONNECTED | REVOKED`.

### Testcases

`DRAFT -> SUBMITTED -> APPROVED | REJECTED | MODIFICATION_REQUIRED`.
Only `APPROVED` testcases can be published or exported. Any substantive edit to an approved testcase creates a new draft version and invalidates the previous approval.

## 4. Phase 0 - Platform Foundation and Contracts

**Outcome:** A deployable modular monolith with tenant-aware security, consistent API behavior, persistence conventions, audit hooks, and contract tests.

### Work packages

- Create workspace packages for `domain`, `contracts`, `prompts`, `connectors`, and `test-utils`.
- Create API modules for auth, projects, audit, and health; create React shell and route guards.
- Configure MongoDB indexes, migrations, GridFS bucket, structured logging, metrics, tracing, and correlation IDs.
- Implement OIDC login validation, RBAC, project membership, tenant isolation, and secret-manager interface.
- Add OpenAPI publication, request schema validation, rate limits, body limits, CORS/CSRF policy, and centralized error handling.
- Add outbox/audit abstraction even before cloud messaging is introduced.

### API contracts

#### `GET /api/v1/me`

Response `200`:

```json
{
  "data": {
    "userId": "usr_123",
    "tenantId": "ten_123",
    "email": "qa@example.com",
    "displayName": "QA User",
    "roles": ["PROJECT_EDITOR"],
    "permissions": ["project:read", "testcase:write"]
  },
  "meta": { "correlationId": "9d..." }
}
```

Edge cases: invalid token returns `401`; disabled user returns `403 USER_DISABLED`; do not trust roles supplied by the client.

#### `GET /api/v1/me/permissions`

Response `200`:

```json
{
  "data": {
    "tenantId": "ten_123",
    "permissions": [
      { "scope": "project", "resourceId": "prj_123", "actions": ["read", "write"] }
    ]
  },
  "meta": { "correlationId": "9d..." }
}
```

#### `POST /api/v1/projects`

Request:

```json
{
  "name": "Payments Portal",
  "key": "PAY",
  "description": "QA knowledge and testcase workspace",
  "settings": {
    "defaultEmbeddingProfile": "requirements-v1",
    "defaultLlmProvider": "openai",
    "timezone": "UTC"
  }
}
```

Response `201`:

```json
{
  "data": {
    "projectId": "prj_123",
    "name": "Payments Portal",
    "key": "PAY",
    "status": "ACTIVE",
    "settings": { "defaultEmbeddingProfile": "requirements-v1", "defaultLlmProvider": "openai" },
    "version": 1,
    "createdAt": "2026-09-03T10:00:00Z"
  },
  "meta": { "correlationId": "9d..." }
}
```

Validation: name 1-120 characters; key 2-20 uppercase alphanumeric plus `_`; key unique within tenant. Duplicate key returns `409 PROJECT_KEY_EXISTS`; unknown provider/profile returns `422`; excessive description returns `400`.

#### `GET /api/v1/projects?cursor=&limit=&status=`

Response `200`: collection envelope containing project summaries. Limit defaults to 25 and is capped at 100. Invalid cursor/filter returns `400`.

#### `GET /api/v1/projects/:projectId`

Response `200`: full project settings, membership summary, integration counts, ingestion counts, and current index status. Unknown or unauthorized project returns `404`.

#### `PATCH /api/v1/projects/:projectId`

Request:

```json
{
  "name": "Payments QA",
  "description": "Updated scope",
  "settings": { "defaultLlmProvider": "groq" },
  "version": 1
}
```

Response `200`: updated project. Immutable `key`, `tenantId`, and IDs are rejected. Stale version returns `409 STALE_VERSION`.

#### `GET /api/v1/projects/:projectId/audit-events?cursor=&action=&from=&to=`

Response `200`: paginated events with actor, action, resource type/id, outcome, correlation ID, timestamp, and redacted change summary. Date range must be valid and bounded; only authorized users can inspect it.

### Phase 0 exit gates

- Contract tests cover success, validation, authentication, authorization, tenant isolation, duplicate idempotency, stale version, pagination, and audit emission.
- No endpoint returns secrets or data from another tenant.
- Health checks distinguish liveness from readiness; readiness fails when required MongoDB/config dependencies are unavailable.
- CI runs typecheck, lint, unit tests, API contract tests, and dependency scanning.

## 5. Phase 1 - File Intake, Ingestion, and Knowledge Index

**Outcome:** Users can upload PDF/Excel artifacts, process them asynchronously, preserve lineage, embed chunks, and search indexed content.

### Work packages

- Implement GridFS upload sessions with streaming, size/type limits, checksum, antivirus hook, and cleanup of abandoned uploads.
- Implement deterministic PDF and workbook extraction, source-aware normalization, metadata extraction, chunking, quality checks, Mistral embedding, and MongoDB vector indexes.
- Store immutable source records, content hashes, chunk versions, embedding model/version/profile, and job logs.
- Implement job worker with bounded retries, dead-letter state, cancellation checkpoints, and idempotent stages.
- Add basic keyword plus vector search; hybrid fusion can remain feature-flagged until Phase 2.

### API contracts

#### `POST /api/v1/projects/:projectId/files`

Use `multipart/form-data` with `file`, `sourceType`, optional `sourceName`, `metadata`, and `contentHash`.

Response `202`:

```json
{
  "data": {
    "sourceId": "src_123",
    "fileId": "file_123",
    "jobId": "job_123",
    "status": "QUEUED",
    "statusUrl": "/api/v1/ingestion/jobs/job_123"
  },
  "meta": { "correlationId": "9d..." }
}
```

Edge cases: unsupported extension/MIME mismatch returns `415`; max file size returns `413`; empty file or corrupt PDF/XLSX returns `422`; same project and content hash returns `409 DUPLICATE_SOURCE` unless `allowDuplicate=true` and caller has permission; interrupted upload must not create a visible source.

#### `GET /api/v1/projects/:projectId/ingestion/jobs?status=&cursor=&limit=`

Response `200`: job summaries including type, sourceId, status, progress, attempt count, timestamps, error code, and `parentJobId`. Never expose worker internals or provider credentials.

#### `GET /api/v1/ingestion/jobs/:jobId`

Response `200`:

```json
{
  "data": {
    "jobId": "job_123",
    "type": "FILE_INGESTION",
    "status": "RUNNING",
    "progress": { "stage": "EMBEDDING", "completed": 80, "total": 100 },
    "sourceId": "src_123",
    "warnings": [{ "code": "TABLE_ROW_SKIPPED", "count": 2 }],
    "error": null,
    "createdAt": "2026-09-03T10:00:00Z",
    "updatedAt": "2026-09-03T10:02:00Z"
  },
  "meta": { "correlationId": "9d..." }
}
```

A job ID from another project/tenant returns `404`. Progress must be monotonic; unknown status is never accepted from a client.

#### `POST /api/v1/ingestion/jobs/:jobId/retry`

Request: `{ "fromStage": "EMBEDDING" }` (optional; only failed retryable stages are allowed).

Response `202`: new job with `parentJobId`. Non-retryable validation, deleted source, or running job returns `409`; provider outage may return `503` with `retryable=true`.

#### `POST /api/v1/ingestion/jobs/:jobId/cancel`

Response `202`: original job with `status: CANCEL_REQUESTED`. A completed job returns `409 JOB_TERMINAL`; worker checks cancellation between chunks and stages. Partial chunks are marked inactive or rolled back atomically so search never sees an incomplete active version.

#### `GET /api/v1/projects/:projectId/sources?sourceType=&status=&cursor=&limit=`

Response `200`: source summaries with lineage, content hash, processing status, active version, and warnings. Deleted/failed sources are filtered by default and can be included only with permission.

#### `GET /api/v1/sources/:sourceId`

Response `200`: source metadata, file metadata, extraction summary, active chunks count, embedding profile/version, and job links. `GET /api/v1/sources/:sourceId/lineage` returns parent source, derived chunks, processing jobs, and downstream evidence references.

#### `POST /api/v1/projects/:projectId/search`

Request:

```json
{
  "query": "refund after payment timeout",
  "filters": { "sourceTypes": ["PDF", "EXCEL"], "tags": ["payments"] },
  "limit": 10,
  "cursor": null
}
```

Response `200`:

```json
{
  "data": {
    "query": "refund after payment timeout",
    "results": [
      {
        "chunkId": "chk_123",
        "sourceId": "src_123",
        "score": 0.87,
        "content": "...",
        "sourceLocation": { "page": 4, "sheet": null, "cellRange": null },
        "embeddingVersion": "mistral-v1",
        "highlights": ["payment timeout"]
      }
    ],
    "truncated": false
  },
  "meta": { "correlationId": "9d..." }
}
```

Blank query returns `400`; limit is capped; filters are allow-listed; no result may cross project scope. If embeddings are unavailable, the API may return keyword-only results with `meta.degraded=true`; if both indexes are unavailable return `503`.

### Phase 1 exit gates

- Reprocessing the same content hash is idempotent and does not create duplicate active chunks.
- A failed stage preserves diagnostics and can resume without redoing successful stages.
- Malicious files are rejected/quarantined; extraction runs with resource/time limits.
- Search results include source evidence and exact lineage locations.
- Backup/restore, GridFS cleanup, index creation, and embedding quota behavior are tested.

## 6. Phase 2 - Integrations, Hybrid Retrieval, and Traceability

**Outcome:** Jira/ADO/Confluence plus test-management integrations can synchronize incrementally, and retrieval uses preprocessing, hybrid fusion, reranking, deduplication, and evidence summaries.

### Work packages

- Add connector SDK interface: `validateCredentials`, `pullPage`, `pullChanges`, `mapRecord`, `subscribeWebhook`, `revoke`.
- Implement Jira, ADO, Confluence/Wiki, TestRail/Xray/Zephyr connectors behind feature flags.
- Store encrypted connector references, never credentials; add OAuth refresh, rate limiting, pagination, retries, circuit breakers, and webhook signature validation.
- Add incremental sync checkpoints, deletion/tombstone handling, conflict policy, source record relationships, and sync audit.
- Add query normalization, abbreviation/synonym expansion, BM25/vector fusion, reranking, deduplication, and bounded summarization.

### API contracts

#### `POST /api/v1/projects/:projectId/integrations`

Request:

```json
{
  "type": "JIRA",
  "name": "Payments Jira",
  "configuration": {
    "baseUrl": "https://jira.example.com",
    "projectKeys": ["PAY"]
  },
  "credentialReference": "secret://tenant/connectors/jira-1"
}
```

Response `201`: integration metadata, `status: PENDING`, capabilities, and redacted configuration. Credential reference must be server-side and caller must have `integration:manage`; arbitrary secret values are rejected.

#### `GET /api/v1/projects/:projectId/integrations`

Response `200`: paginated integrations with health, last sync, next sync, capabilities, and error summary.

#### `GET /api/v1/integrations/:integrationId`

Response `200`: integration details with redacted configuration and recent syncs. Unauthorized IDs return `404`.

#### `POST /api/v1/integrations/:integrationId/test`

Response `200`:

```json
{
  "data": {
    "reachable": true,
    "authenticated": true,
    "capabilities": ["PULL_CHANGES", "WEBHOOKS"],
    "latencyMs": 182,
    "warnings": []
  },
  "meta": { "correlationId": "9d..." }
}
```

Invalid credentials return `422`; timeout returns `504`; provider rate limiting returns `429` and `Retry-After`.

#### `POST /api/v1/integrations/:integrationId/sync`

Request:

```json
{
  "mode": "INCREMENTAL",
  "since": "2026-09-01T00:00:00Z",
  "resourceTypes": ["ISSUE", "PAGE"]
}
```

Response `202`: sync job. A running sync returns `409 SYNC_ALREADY_RUNNING` with its existing job ID. A full sync requires explicit permission and confirmation; a missing/stale checkpoint starts from a bounded fallback and reports a warning.

#### `DELETE /api/v1/integrations/:integrationId`

Response `204`: revoke connector, stop scheduled syncs, retain lineage/audit, and mark remote-derived records inactive according to retention policy. Deletion is idempotent; active sync returns `409` unless `force=true` is authorized.

#### `POST /api/v1/projects/:projectId/retrieval`

Request:

```json
{
  "query": "What happens when a refund request times out?",
  "filters": { "sourceTypes": ["JIRA", "CONFLUENCE", "TESTRAIL"] },
  "topK": 30,
  "rerank": true,
  "summarize": true,
  "includeEvidence": true
}
```

Response `200`:

```json
{
  "data": {
    "normalizedQuery": "refund request timeout behavior",
    "results": [],
    "summary": { "text": "...", "evidenceIds": ["ev_1"] },
    "evidence": [],
    "warnings": [],
    "degraded": false
  },
  "meta": { "correlationId": "9d..." }
}
```

Query length, top-K, filter count, synonym expansion, and summary token limits are capped. If reranking or summarization fails, return retrieved evidence with a warning and `degraded=true`; never present an unsupported summary as fact. A no-result response is `200` with empty results, not `404`.

### Phase 2 exit gates

- Connector contract tests cover pagination, rate limits, token refresh, deleted/renamed records, malformed remote data, webhook replay, and remote outage.
- Sync is resumable and idempotent; a record version conflict is retained for review rather than silently overwritten.
- Retrieval logs preprocessing, component scores, final score, latency, and evidence IDs without logging sensitive content by default.
- Cross-source duplicates are collapsed while preserving all source references.

## 7. Phase 3 - Testcase Generation, Confidence, Review, and Export

**Outcome:** Users can generate evidence-backed testcases, edit them, approve them, and export only approved versions.

### Work packages

- Implement testcase schema, version history, evidence references, prompt registry, provider router, structured output validation, and safety/consistency checks.
- Implement recommendation of test types and user override.
- Implement configurable confidence scoring with factor explanation and thresholds.
- Implement review workflow, comments, audit events, notifications, and export adapters.
- Apply provider timeouts, token/cost budgets, fallback policy, redaction, and provider-specific response normalization.

### API contracts

#### `POST /api/v1/projects/:projectId/testcases/recommend-types`

Request:

```json
{
  "requirement": "A user can request a refund after a payment timeout.",
  "context": { "sourceIds": ["src_123"] }
}
```

Response `200`:

```json
{
  "data": {
    "recommendations": [
      { "type": "FUNCTIONAL", "relevance": 0.96, "reason": "..." },
      { "type": "NEGATIVE", "relevance": 0.82, "reason": "..." }
    ],
    "evidence": [{ "evidenceId": "ev_1", "sourceId": "src_123" }]
  },
  "meta": { "correlationId": "9d..." }
}
```

No recommendation is a valid result. Missing requirement returns `400`; unsupported type selections are rejected rather than silently ignored.

#### `POST /api/v1/projects/:projectId/testcases/generate`

Request:

```json
{
  "requirement": "A user can request a refund after a payment timeout.",
  "testTypes": ["FUNCTIONAL", "NEGATIVE", "BOUNDARY"],
  "provider": { "name": "openai", "model": "gpt-4.1" },
  "retrieval": { "topK": 20, "sourceIds": ["src_123"] },
  "parameters": { "temperature": 0.2, "maxOutputTokens": 4000 },
  "promptVersion": "testcase-v3"
}
```

Response `202`: generation job with `generationId`, job status URL, selected provider/model, and redacted request summary. The server validates provider availability, project allow-list, token budget, and prompt version before enqueueing. If no evidence is found, generation is rejected with `422 INSUFFICIENT_EVIDENCE` unless the project policy explicitly permits an `UNVERIFIED` draft.

#### `GET /api/v1/testcases/:testcaseId`

Response `200`: current testcase version, complete steps/expected results, traceability references, evidence, provider/model metadata, prompt version, confidence score/explanation, review status, and version number. Sensitive test data is masked according to project policy.

#### `GET /api/v1/projects/:projectId/testcases?status=&type=&confidenceBelow=&cursor=&limit=`

Response `200`: paginated testcase summaries. Filters are allow-listed and stable-sorted by `(updatedAt, id)` for cursor correctness.

#### `PATCH /api/v1/testcases/:testcaseId`

Request:

```json
{
  "title": "Refund after payment timeout",
  "steps": [
    { "number": 1, "action": "Submit a payment until the client times out", "expectedResult": "The payment status is retrievable" }
  ],
  "reviewerComments": "Clarified timeout behavior",
  "version": 1
}
```

Response `200`: new testcase version with `reviewStatus: DRAFT`. Required fields, step numbering, evidence reference format, and maximum lengths are validated. Editing an approved testcase does not mutate the approved version; it creates a draft and records `supersedesVersion`.

#### `POST /api/v1/testcases/:testcaseId/regenerate`

Request: `{ "testTypes": ["NEGATIVE"], "provider": { "name": "anthropic", "model": "..." }, "version": 2 }`.

Response `202`: generation job linked to the testcase version. It must preserve prior evidence and prompt metadata. Running regeneration returns `409`; provider outage returns `503` or a queued retry depending on policy.

#### `GET /api/v1/testcases/:testcaseId/confidence`

Response `200`:

```json
{
  "data": {
    "score": 0.84,
    "band": "HIGH",
    "explanation": {
      "retrievalRelevance": 0.9,
      "sourceAuthority": 0.8,
      "requirementCoverage": 0.88,
      "traceabilityCompleteness": 0.82,
      "consistency": 0.86
    },
    "scoringVersion": "confidence-v2",
    "calculatedAt": "2026-09-03T10:04:00Z"
  },
  "meta": { "correlationId": "9d..." }
}
```

#### `POST /api/v1/testcases/:testcaseId/recalculate-confidence`

Response `202`: confidence job. It is idempotent for the same testcase version and scoring version; missing evidence or unavailable retrieval marks score `UNAVAILABLE` rather than assigning zero without explanation.

#### `POST /api/v1/testcases/:testcaseId/submit-review`

Request: `{ "version": 2, "comment": "Ready for review" }`.

Response `200`: testcase with `reviewStatus: SUBMITTED`. Only draft/modification-required versions can be submitted; missing required fields or insufficient confidence return `422`.

#### `POST /api/v1/testcases/:testcaseId/approve`

Request: `{ "version": 2, "comment": "Approved for release" }`.

Response `200`: testcase with `reviewStatus: APPROVED`, reviewer identity, timestamp, and immutable approved version. Self-approval can be disabled by policy; if enabled, it remains auditable. Approval of a stale version returns `409`.

#### `POST /api/v1/testcases/:testcaseId/reject`

Request: `{ "version": 2, "comment": "Missing API error case" }`.

Response `200`: `REJECTED`. Comment is required and bounded. Rejection is terminal for that version but a new version may be created.

#### `POST /api/v1/testcases/:testcaseId/request-modification`

Request: `{ "version": 2, "comment": "Add boundary values" }`.

Response `200`: `MODIFICATION_REQUIRED`. Comment and reviewer identity are recorded.

#### `GET /api/v1/testcases/:testcaseId/reviews`

Response `200`: immutable review history, ordered by event time, including action, actor, comment, version, and correlation ID.

#### `POST /api/v1/projects/:projectId/testcases/export`

Request:

```json
{
  "testcaseIds": ["tc_123", "tc_456"],
  "format": "CSV",
  "destination": { "type": "DOWNLOAD" },
  "includeEvidence": true
}
```

Response `202`: export job. Any non-approved testcase causes `422 APPROVAL_REQUIRED` with per-ID results, unless `skipUnapproved=true` is explicitly authorized and reported. Mixed-project IDs, duplicate IDs, too many records, unsupported format, and stale versions are rejected. Export is a snapshot of approved versions; later edits do not alter it.

#### `GET /api/v1/exports/:exportId`

Response `200`: status, format, record count, skipped count, checksum, expiration, and download URL only when succeeded. Download URLs are short-lived and authorization checked. Expired exports return `410`; failed exports expose safe error details.

### Phase 3 exit gates

- Structured LLM output is schema-validated and repaired/rejected within bounded attempts.
- Prompt injection from retrieved documents is treated as untrusted data; system instructions and tenant data are separated.
- Provider credentials never reach the client or logs; timeout, malformed output, quota, and content-filter errors are mapped consistently.
- Every generated testcase has evidence IDs, model/provider, prompt version, confidence explanation, and audit trail.
- Export and publication are impossible for unapproved versions, including through direct API calls.
- Approval, rejection, modification, edit-after-approval, duplicate clicks, and concurrent reviewers have integration tests.

## 8. Phase 4 - API Intelligence, Advanced Sources, and Event-Driven Scale

**Outcome:** OpenAPI, code, Figma, recordings, defects, scheduled synchronization, and cloud-managed messaging extend coverage while preserving the same contracts.

### Work packages

- Parse OpenAPI 3.x/Swagger 2.x into APIs, operations, schemas, security schemes, links, and endpoint sequences.
- Add API testcase generation and schema/boundary/security validation.
- Add Git/code ingestion, Figma/UI extraction, recordings transcription, release notes, and defect correlation through connector interfaces.
- Introduce cloud messaging adapters, transactional outbox, consumer idempotency, retries, dead-letter queues, replay tooling, and event versioning.
- Add Kubernetes workers for ingestion, embeddings, retrieval, generation, and export; retain modular in-process adapters for local development.
- Add coverage dashboards, regression optimization, CI/CD hooks, execution results, and multi-tenant controls.

### API contracts

#### `POST /api/v1/projects/:projectId/openapi/import`

Request:

```json
{
  "documentUrl": "https://api.example.com/openapi.json",
  "content": null,
  "mode": "UPSERT",
  "environment": "staging",
  "generateTestcases": { "types": ["POSITIVE", "NEGATIVE", "BOUNDARY"] }
}
```

Exactly one of `documentUrl` or `content` is required. Response `202`: import job. Validate URL allow-list/SSRF protections, content size, JSON/YAML parsing, supported version, duplicate operation IDs, unresolved `$ref`, circular schemas, invalid security schemes, and unsupported webhooks. Partial import is allowed only with `warnings` and explicit policy; otherwise the operation fails atomically.

#### `GET /api/v1/projects/:projectId/apis`

Response `200`: paginated API summaries including source, version, base URL redacted as required, operation count, schema count, and import status.

#### `GET /api/v1/apis/:apiId`

Response `200`: API metadata, operations, parameters, request/response schemas, security schemes, dependencies, source locations, and warnings. Secrets and example credentials are redacted.

#### `POST /api/v1/projects/:projectId/apis/testcases/generate`

Request:

```json
{
  "apiId": "api_123",
  "operationIds": ["createRefund"],
  "types": ["POSITIVE", "NEGATIVE", "BOUNDARY"],
  "includeAuthCases": true,
  "provider": { "name": "openai", "model": "gpt-4.1" }
}
```

Response `202`: generation job. Unsupported/missing operation IDs return per-operation `422` details. Generated API testcases remain drafts and require the normal review flow.

### Event contract

All events use a versioned envelope:

```json
{
  "eventId": "evt_123",
  "eventType": "testcase.generated",
  "eventVersion": 1,
  "occurredAt": "2026-09-03T10:05:00Z",
  "tenantId": "ten_123",
  "projectId": "prj_123",
  "correlationId": "9d...",
  "causationId": "job_123",
  "subjectId": "tc_123",
  "payload": {}
}
```

Consumers deduplicate by `eventId`, validate tenant/project scope, acknowledge only after durable processing, and send poison messages to a dead-letter queue. Producers publish through an outbox transaction so database state cannot commit without an eventual event. Event payloads contain IDs and summaries, not secrets or large documents.

### Phase 4 exit gates

- OpenAPI imports are deterministic, repeatable, and safe against SSRF/resource exhaustion.
- Event replay produces no duplicate domain effects; schema evolution supports old consumers.
- Dead-letter alerts, replay authorization, consumer lag, provider cost, and job throughput are observable.
- Kubernetes probes, graceful shutdown, worker concurrency limits, autoscaling, and disaster recovery are tested.
- Tenant quotas, retention, export/delete requests, regional data controls, and audit retention are enforced.

## 9. API Edge-Case Matrix

| Area | Required behavior |
|---|---|
| Authorization | Resource not found and unauthorized resource both return `404` where disclosure is unsafe; enforce tenant and project on every query. |
| Idempotency | Same key and same body replays original response; same key with different body returns `409 IDEMPOTENCY_KEY_REUSED`. |
| Concurrency | Require version/ETag for mutable resources; reject stale updates; never silently overwrite approved versions. |
| Pagination | Cursor is opaque, signed or server-validated, bounded, and stable under inserts; reject malformed/foreign cursors. |
| Limits | Enforce request, file, query, result, token, batch, and export limits before expensive work. |
| Time | Store UTC; validate ranges and timezone identifiers; handle clock skew in webhook signatures and token expiry. |
| Dependencies | Map timeout, quota, malformed response, auth failure, and outage separately; apply bounded retry with jitter only to retryable failures. |
| Partial work | Preserve stage-level outcomes and warnings; do not expose incomplete chunks or testcases as complete. |
| Deletion | Use tombstones where lineage requires them; remove indexes/files according to retention policy and audit the action. |
| Sensitive data | Redact credentials, authorization headers, secrets, PII, and raw recordings from logs, prompts, exports, and errors. |
| Webhooks | Verify signature, timestamp, event ID, and source; deduplicate replays; acknowledge quickly and process asynchronously. |
| LLM output | Validate JSON/schema, required fields, allowed enums, step ordering, evidence IDs, and hallucinated references; reject unsupported claims. |
| Search | Empty/no-result/degraded states are distinct; filters cannot broaden authorization scope; evidence remains traceable after dedupe. |
| Jobs | Status transitions are monotonic; cancel is cooperative; retry creates a linked job; terminal jobs are immutable. |

## 10. Data and Indexing Requirements

Minimum collections: `tenants`, `users`, `projects`, `memberships`, `integrations`, `sources`, `sourceVersions`, `chunks`, `ingestionJobs`, `retrievalRuns`, `testcases`, `testcaseVersions`, `reviews`, `exports`, `auditEvents`, `outboxEvents`, and `idempotencyRecords`.

Every chunk stores `embeddingModel`, `embeddingVersion`, `embeddingProfile`, `vector`, `contentHash`, `sourceType`, `sourceId`, `projectId`, `tenantId`, `chunkId`, `sourceLocation`, `createdAt`, and `updatedAt`. Unique indexes must include tenant/project scope where appropriate. Never rely on application filtering alone for isolation.

Use soft deletion/tombstones for source lineage, immutable testcase versions, TTL indexes only for temporary artifacts, and encryption/key rotation through the platform secret manager.

## 11. Testing and Operational Readiness

### Test layers

- Unit: domain state machines, parsers, chunking, scoring, redaction, retry policy.
- Contract: OpenAPI request/response schemas, error codes, event schemas, connector SDK behavior.
- Integration: MongoDB indexes/transactions, GridFS, worker resume/cancel, provider adapters, outbox consumers.
- Security: tenant isolation, RBAC, SSRF, upload abuse, prompt injection, secret leakage, webhook forgery, rate limits.
- End-to-end: upload to search, sync to evidence, generation to approval, edit-after-approval, export, and failure recovery.
- Load/resilience: large files, concurrent syncs, provider outage, MongoDB failover, queue lag, duplicate events, and worker termination.

### Required observability

Emit structured metrics and traces for request latency, job duration/stage, queue lag, extraction failures, index freshness, retrieval scores, evidence coverage, LLM latency/tokens/cost, confidence distribution, approval outcomes, export failures, connector health, and authorization denials. Correlate all records by request, job, source, retrieval, generation, and event IDs.

### Release and rollback

Use feature flags for each connector, provider, retrieval stage, and event consumer. Database migrations must be backward-compatible for at least one release. Deploy API and workers separately, drain workers before shutdown, and retain a rollback procedure that does not delete lineage or approved testcase history.

## 12. Recommended Delivery Order

1. Phase 0 foundation and contracts.
2. Phase 1 PDF/Excel upload, indexing, and basic search.
3. Phase 2 integrations and advanced retrieval.
4. Phase 3 generation, confidence, review, and export.
5. Phase 4 OpenAPI intelligence, additional sources, messaging, and scale extraction.

The first production release should be declared only after Phase 3’s approval and export gates pass. Phase 4 extends coverage and scale; it must not weaken the Phase 0 security, Phase 1 lineage, or Phase 3 human-approval invariants.
