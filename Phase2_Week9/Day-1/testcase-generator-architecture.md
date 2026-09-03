# Testcase Generator — System Architecture

## 1. Overview

The Testcase Generator is an AI-assisted QA platform that ingests requirements and engineering artifacts, indexes them for semantic and keyword retrieval, and generates traceable testcases using configurable LLM providers.

Primary stack:
- Frontend: React
- Backend/API: Node.js + TypeScript + Express
- Vector/search database: MongoDB
- Embeddings: Mistral AI, with source-specific embedding configuration
- LLM providers: OpenAI, Groq, Anthropic (user-selectable)
- Messaging: cloud-managed messaging
- Deployment: enterprise hybrid, containerized/Kubernetes-oriented

The architecture starts as a modular monolith with strong module boundaries so high-load components can later be extracted into services.

## 2. Goals

- Ingest heterogeneous QA/engineering sources.
- Preserve rich source lineage and traceability.
- Support hybrid keyword + vector retrieval.
- Apply query preprocessing, reranking, deduplication, and summarization.
- Generate multiple testcase types with AI assistance.
- Correlate requirements, UI, APIs, code, existing tests, and defects.
- Require human approval before testcase publication/export.
- Provide evidence-based confidence scoring.
- Expose APIs for all major capabilities.
- Support future service extraction and multi-tenant operation.

## 3. Supported Inputs

- BRD / user stories
- Jira / ADO
- TestRail / Xray / Zephyr
- Figma / UI specifications
- Confluence / Wiki
- Release notes
- Excel / PDF
- Grooming-session recordings
- Developer code repositories
- Swagger / OpenAPI
- Defect databases

## 4. High-Level Architecture

```text
                    +----------------------+
                    |      React Web UI    |
                    +----------+-----------+
                               |
                         HTTPS / REST
                               |
                    +----------v-----------+
                    | Node.js + Express    |
                    | API / Modular Core    |
                    +----------+-----------+
                               |
       +-----------------------+------------------------+
       |                       |                        |
+------v------+         +------v------+          +------v------+
| Integration |         | Retrieval   |          | Testcase    |
| Layer       |         | Pipeline    |          | Generation  |
+------+------+         +------+------+          +------+------+
       |                       |                        |
       |                       |                        |
+------v------+         +------v----------------+       |
| Ingestion   |         | Query preprocessing  |       |
| Pipeline    |         | Hybrid search        |       |
+------+------+         | Reranking            |       |
       |                 | Deduplication        |       |
       |                 | Summarization        |       |
       |                 +----------+-----------+       |
       |                            |                   |
       +----------------------------+-------------------+
                                    |
                              +-----v------+
                              |  MongoDB   |
                              | Vector +   |
                              | metadata   |
                              +------------+
                                    |
                              +-----v------+
                              | LLM Router |
                              | OpenAI /   |
                              | Groq /     |
                              | Anthropic  |
                              +------------+

 Large files / recordings:
 Integration -> ingestion -> MongoDB/GridFS

 Event-driven communication:
 Services/modules -> Cloud-managed messaging -> consumers/workers
```

## 5. Architectural Style

### Initial architecture
Use a modular monolith for the first production implementation.

Recommended modules:
- Auth & IAM
- Projects/Tenants
- Integrations
- Ingestion
- Document Processing
- Embeddings
- Retrieval
- Testcase Generation
- API Intelligence
- Review & Approval
- Confidence Scoring
- Export
- Audit
- Notifications

### Evolution path
Modules should communicate through explicit interfaces/events. High-volume modules can later become independently deployable services without redesigning domain contracts.

## 6. Integration Layer

A dedicated integration layer isolates external systems from core business logic.

Responsibilities:
- OAuth/API authentication
- Credential/secret retrieval
- API calls
- Pagination
- Rate limiting
- Retry with backoff
- Incremental synchronization
- Webhook processing
- Source normalization
- Connector health/status
- Mapping source records to canonical ingestion records

Connectors should include:
- Jira
- Azure DevOps
- TestRail
- Xray
- Zephyr
- Confluence/Wiki
- Figma
- Git providers
- Defect systems
- OpenAPI/Swagger
- File upload/import

## 7. Ingestion Pipeline

```text
Source
  -> Integration/File Intake
  -> Validation
  -> Extraction
  -> Source-aware Parsing
  -> Normalization
  -> Metadata/Lineage Extraction
  -> Chunking
  -> Quality Checks
  -> Mistral Embedding
  -> MongoDB
  -> Ingestion Event
```

Use deterministic parsing first. LLM enrichment is applied only where it provides value.

Examples:
- PDF: text/layout extraction
- Excel: workbook/sheet/table extraction
- Jira/ADO: issue fields and relationships
- Figma: UI structure/specification
- Code: files, symbols, modules, dependencies
- OpenAPI: endpoints, schemas, parameters, security
- Recording: speech-to-text followed by structured extraction

## 8. Embedding Strategy

Use Mistral AI embeddings with source-specific embedding configuration.

Every embedding should be versioned so model/configuration changes can trigger controlled re-embedding.

Recommended stored fields:
- embeddingModel
- embeddingVersion
- vector
- contentHash
- sourceType
- sourceId
- projectId
- tenantId
- chunkId
- sourceLocation
- createdAt
- updatedAt

## 9. MongoDB Strategy

MongoDB serves as the vector/search store and stores rich retrieval metadata.

Support:
- Vector similarity search
- Keyword/BM25-style search
- Metadata filtering
- Hybrid retrieval
- Versioned chunks
- Source lineage
- Relationships
- Content hashes
- Ingestion job identifiers

Original uploaded files are stored in MongoDB GridFS according to the selected architecture decision.

## 10. Retrieval Pipeline

```text
User Query
   |
   v
Normalization
   |
Abbreviation Expansion
   |
Synonym Expansion
   |
   v
Hybrid Search
(BM25 + Vector)
   |
   v
Reranking
   |
   v
Deduplication
   |
   v
Summarization
   |
   v
Prompt + Query + Context
   |
   v
Selected LLM Provider
```

Retrieval should retain evidence references so every generated testcase can point back to supporting source records.

## 11. Testcase Generation

The system recommends relevant test types using AI and lets the user modify the selection.

Possible types:
- Functional
- Positive
- Negative
- Boundary
- Regression
- Smoke
- Integration
- API
- UI
- Accessibility
- Security-oriented
- Data validation
- Compatibility

Generation flow:

```text
Requirement/User Request
        |
        v
Test Type Recommendation
        |
User selects/modifies types
        |
        v
Retrieval Pipeline
        |
        v
Context Assembly
        |
        v
LLM Generation
        |
        v
Validation / Consistency Checks
        |
        v
Confidence Scoring
        |
        v
Human Approval
        |
 Approved
        |
        v
Export / Publish
```

## 12. Testcase Data Model

A testcase should contain:
- testcaseId
- title
- objective
- testType
- preconditions
- testData
- environment
- steps
- expectedResults
- priority
- severity
- tags
- requirementReferences
- UIReferences
- APIReferences
- codeReferences
- defectReferences
- sourceEvidence
- generatedBy
- model/provider
- promptVersion
- confidenceScore
- reviewStatus
- reviewer
- reviewerComments
- version
- createdAt
- updatedAt

## 13. Human-in-the-Loop

Approval is mandatory before testcase publication/export.

Statuses:
- DRAFT
- SUBMITTED
- APPROVED
- REJECTED
- MODIFICATION_REQUIRED

Reviewer actions:
- Approve
- Reject
- Modify with comments

All review actions must be auditable.

## 14. Confidence Scoring

Use an evidence-based weighted score.

Inputs can include:
- Retrieval relevance
- Reranker score
- Source authority
- Requirement coverage
- Number/quality of supporting evidence records
- Traceability completeness
- Testcase consistency
- LLM assessment

Scores should be configurable by project/team and stored with an explanation of contributing factors.

## 15. LLM Provider Layer

The UI allows selection among:
- OpenAI
- Groq
- Anthropic

Create a provider abstraction:

```text
LLMProvider
  -> OpenAIProvider
  -> GroqProvider
  -> AnthropicProvider
```

The core generation module must not depend directly on a vendor SDK.

Store:
- provider
- model
- model version where available
- prompt version
- generation parameters
- token/cost metadata where available
- request correlation ID

## 16. API Intelligence

OpenAPI/Swagger should receive full treatment.

Capabilities:
- Parse endpoints
- Parse HTTP methods
- Parameters
- Request/response schemas
- Authentication/security schemes
- Dependencies and endpoint sequences
- Schema validation
- Positive/negative tests
- Boundary tests
- Existing API testcase comparison
- Requirement/API correlation
- Defect/API correlation
- API-specific traceability

## 17. Event-Driven Architecture

Use cloud-managed messaging for asynchronous/event-driven communication.

Example events:
- `source.connected`
- `source.sync.requested`
- `source.sync.completed`
- `ingestion.started`
- `document.extracted`
- `document.chunked`
- `embedding.completed`
- `index.updated`
- `retrieval.completed`
- `testcase.generation.requested`
- `testcase.generated`
- `review.requested`
- `testcase.approved`
- `testcase.rejected`

Events should be idempotent and carry correlation IDs.

## 18. Security Architecture

Use enterprise-grade IAM:
- SSO/OIDC
- RBAC
- Resource-level permissions
- Tenant/project/source authorization
- Service-to-service authentication
- Central secrets management
- Encryption in transit and at rest
- Audit logging
- Configurable authorization policies

Do not expose connector credentials to the React client.

## 19. API Endpoints

### Authentication / user context
- `GET /api/v1/me`
- `GET /api/v1/me/permissions`

### Projects
- `POST /api/v1/projects`
- `GET /api/v1/projects`
- `GET /api/v1/projects/:projectId`
- `PATCH /api/v1/projects/:projectId`

### Integrations
- `POST /api/v1/projects/:projectId/integrations`
- `GET /api/v1/projects/:projectId/integrations`
- `GET /api/v1/integrations/:integrationId`
- `POST /api/v1/integrations/:integrationId/test`
- `POST /api/v1/integrations/:integrationId/sync`
- `DELETE /api/v1/integrations/:integrationId`

### File ingestion
- `POST /api/v1/projects/:projectId/files`
- `GET /api/v1/projects/:projectId/ingestion/jobs`
- `GET /api/v1/ingestion/jobs/:jobId`
- `POST /api/v1/ingestion/jobs/:jobId/retry`
- `POST /api/v1/ingestion/jobs/:jobId/cancel`

### Knowledge / retrieval
- `POST /api/v1/projects/:projectId/search`
- `POST /api/v1/projects/:projectId/retrieval`
- `GET /api/v1/projects/:projectId/sources`
- `GET /api/v1/sources/:sourceId`
- `GET /api/v1/sources/:sourceId/lineage`

### Testcase generation
- `POST /api/v1/projects/:projectId/testcases/recommend-types`
- `POST /api/v1/projects/:projectId/testcases/generate`
- `GET /api/v1/testcases/:testcaseId`
- `GET /api/v1/projects/:projectId/testcases`
- `PATCH /api/v1/testcases/:testcaseId`
- `POST /api/v1/testcases/:testcaseId/regenerate`

### API intelligence
- `POST /api/v1/projects/:projectId/openapi/import`
- `GET /api/v1/projects/:projectId/apis`
- `GET /api/v1/apis/:apiId`
- `POST /api/v1/projects/:projectId/apis/testcases/generate`

### Review
- `POST /api/v1/testcases/:testcaseId/submit-review`
- `POST /api/v1/testcases/:testcaseId/approve`
- `POST /api/v1/testcases/:testcaseId/reject`
- `POST /api/v1/testcases/:testcaseId/request-modification`
- `GET /api/v1/testcases/:testcaseId/reviews`

### Confidence
- `GET /api/v1/testcases/:testcaseId/confidence`
- `POST /api/v1/testcases/:testcaseId/recalculate-confidence`

### Export
- `POST /api/v1/projects/:projectId/testcases/export`
- `GET /api/v1/exports/:exportId`

### Audit
- `GET /api/v1/projects/:projectId/audit-events`

## 20. Project Folder Structure

```text
testcase-generator/
├── apps/
│   ├── web/                         # React application
│   │   ├── src/
│   │   │   ├── components/
│   │   │   ├── pages/
│   │   │   ├── features/
│   │   │   │   ├── projects/
│   │   │   │   ├── ingestion/
│   │   │   │   ├── search/
│   │   │   │   ├── testcases/
│   │   │   │   ├── review/
│   │   │   │   └── settings/
│   │   │   ├── services/
│   │   │   ├── hooks/
│   │   │   ├── state/
│   │   │   └── routes/
│   │   └── package.json
│   │
│   └── api/                         # Node + TypeScript + Express
│       ├── src/
│       │   ├── config/
│       │   ├── middleware/
│       │   ├── routes/
│       │   ├── controllers/
│       │   ├── modules/
│       │   │   ├── auth/
│       │   │   ├── projects/
│       │   │   ├── integrations/
│       │   │   ├── ingestion/
│       │   │   ├── documents/
│       │   │   ├── embeddings/
│       │   │   ├── retrieval/
│       │   │   ├── testcase-generation/
│       │   │   ├── api-intelligence/
│       │   │   ├── review/
│       │   │   ├── confidence/
│       │   │   ├── exports/
│       │   │   └── audit/
│       │   ├── providers/
│       │   │   ├── llm/
│       │   │   │   ├── openai/
│       │   │   │   ├── groq/
│       │   │   │   └── anthropic/
│       │   │   ├── embeddings/
│       │   │   ├── search/
│       │   │   ├── storage/
│       │   │   └── messaging/
│       │   ├── shared/
│       │   └── server.ts
│       └── package.json
│
├── packages/
│   ├── domain/                      # Shared domain models
│   ├── contracts/                   # API/event contracts
│   ├── prompts/                     # Versioned LLM prompts
│   ├── connectors/                  # Connector SDK/interfaces
│   └── test-utils/
│
├── infrastructure/
│   ├── docker/
│   ├── kubernetes/
│   ├── terraform/
│   ├── messaging/
│   └── monitoring/
│
├── docs/
│   ├── architecture/
│   ├── api/
│   ├── adr/
│   ├── ingestion/
│   └── retrieval/
│
├── tests/
│   ├── unit/
│   ├── integration/
│   ├── contract/
│   └── e2e/
│
├── .env.example
├── docker-compose.yml
├── package.json
├── tsconfig.base.json
└── README.md
```

## 21. Recommended Domain Boundaries

Keep these boundaries explicit even inside the modular monolith:

1. Source/Integration Management
2. Ingestion & Document Processing
3. Knowledge Index
4. Retrieval
5. LLM/Generation
6. Testcase Domain
7. Review/Approval
8. Confidence/Evaluation
9. Export
10. IAM/Audit

This enables later extraction into services with minimal changes.

## 22. Non-Functional Requirements

### Performance
- Async processing for large ingestion jobs.
- Streaming/progress updates where appropriate.
- Configurable retrieval limits.
- Cache repeated retrieval/generation requests where safe.

### Reliability
- Retryable event handlers.
- Idempotent ingestion.
- Dead-letter handling.
- Connector health checks.
- Model-provider error handling.

### Observability
Track:
- Request correlation ID
- Ingestion job ID
- Source ID
- Retrieval latency
- Search/reranking scores
- LLM latency
- Token usage/cost where available
- Generation outcome
- Approval outcome

### Data governance
- Source lineage
- Content hashing
- Versioned embeddings
- Prompt versioning
- Model/provider metadata
- Audit history
- Resource-level authorization

## 23. Future Enhancements

- Voice chat in the application.
- Streaming voice-to-testcase workflows.
- More advanced confidence calibration.
- Automatic regression-suite optimization.
- Testcase execution integration.
- CI/CD integration.
- Automatic defect-to-testcase generation.
- Requirement coverage dashboards.
- Multi-tenant SaaS controls.
- Specialized code and UI agents.

## 24. Key Architectural Decisions

| Decision | Selected approach |
|---|---|
| Overall architecture | Hybrid: modular monolith with extraction-ready boundaries |
| Ingestion trigger | Manual + scheduled + event-driven |
| Integration architecture | Dedicated integration layer |
| Ingestion processing | Deterministic + source-aware + selective AI enrichment |
| Metadata | Enterprise-grade lineage/traceability metadata |
| Search | Vector + hybrid keyword/vector |
| Embeddings | Mistral with source-specific configuration |
| LLM | User-selectable OpenAI/Groq/Anthropic |
| Retrieval | Multi-stage retrieval pipeline |
| Human review | Mandatory approval |
| Confidence | Evidence-based weighted score |
| Async/eventing | Event-driven |
| Messaging | Cloud-managed messaging |
| Security | Enterprise IAM + fine-grained authorization |
| Data stores | MongoDB + specialized stores where needed |
| Original files | MongoDB GridFS |
| OpenAPI | Full API intelligence |
| Deployment | Enterprise hybrid/Kubernetes-oriented |

## 25. Suggested MVP Sequence

### Phase 1
- React UI
- Node/Express API
- Project management
- File upload
- PDF/Excel ingestion
- MongoDB/GridFS
- Mistral embeddings
- Vector search
- Basic testcase generation
- Mandatory approval

### Phase 2
- Hybrid BM25 + vector search
- Jira/ADO/Confluence integrations
- TestRail/Xray/Zephyr integration
- Reranking/deduplication
- Confidence scoring
- Rich traceability

### Phase 3
- Figma
- Git/code ingestion
- OpenAPI intelligence
- Defect correlation
- Scheduled/event-driven synchronization

### Phase 4
- Cloud-managed messaging
- Kubernetes deployment
- Advanced observability
- Multi-provider LLM routing improvements
- Voice chat
- Advanced multi-tenant capabilities
