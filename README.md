# System Health Check API

A production-grade Python REST API that evaluates the health of a system composed of multiple, interdependent components modeled as a **Directed Acyclic Graph (DAG)**.

---

## Table of Contents
- [Architecture Overview](#architecture-overview)
- [Features Implemented](#features-implemented)
- [Features Intentionally Excluded](#features-intentionally-excluded)
- [Assumptions](#assumptions)
- [Key Design Decisions & Tradeoffs](#key-design-decisions--tradeoffs)
- [Observability](#observability)
- [Quick Start](#quick-start)
- [API Reference](#api-reference)
- [Sample Request & Response](#sample-request--response)
- [Running Tests](#running-tests)
- [Infrastructure (Terraform)](#infrastructure-terraform)
- [CI/CD Pipeline](#cicd-pipeline)
- [AI Tools Usage](#ai-tools-usage)

---

## Architecture Overview

```
Client
  │
  ▼
FastAPI (uvicorn)
  │
  ├── POST /health-check
  │     ├── Parse JSON → validate Pydantic models
  │     ├── Build adjacency list from edges
  │     ├── BFS traversal → determine evaluation order
  │     ├── asyncio.gather() → concurrent health probes (httpx)
  │     └── Aggregate → return SystemHealthReport
  │
  ├── POST /health-check/visualize  (optional)
  │     └── Same as above + networkx/matplotlib PNG (base64)
  │
  ├── GET  /health     → liveness probe
  ├── GET  /ready      → readiness probe
  └── GET  /metrics    → Prometheus metrics
```

```
AWS Infrastructure (Terraform):

Internet → ALB → ECS Fargate (private subnets) → CloudWatch Logs
                      ↑
                ECR (Docker image)
```

---

## Features Implemented

| Feature | Status | Notes |
|---|---|---|
| JSON input → DAG construction | ✅ | Adjacency list; validates node references |
| BFS traversal | ✅ | Topological BFS; detects cycles |
| Async health evaluation | ✅ | `asyncio.gather` across all nodes concurrently |
| HTTP health probing | ✅ | `httpx.AsyncClient` with 5 s timeout |
| Declared-status fallback | ✅ | Used when `health_url` is absent (demo/test mode) |
| Human-readable table output | ✅ | Structured JSON with `overall_status`, counts, per-component detail |
| DAG visualization (PNG) | ✅ | `/health-check/visualize` returns base64 PNG; red = unhealthy |
| Cycle detection | ✅ | 422 returned if cycle found |
| Liveness / readiness probes | ✅ | `/health` and `/ready` |
| Prometheus metrics | ✅ | Request count, duration histogram, per-component status counter |
| Structured JSON logging | ✅ | `python-json-logger`; ELK/Splunk-ready |
| Dockerfile (multi-stage) | ✅ | Builder + slim runtime; non-root user |
| Terraform (AWS ECS Fargate) | ✅ | VPC, ALB, ECR, ECS, auto-scaling, CloudWatch |
| GitHub Actions CI/CD | ✅ | Test → Docker build/push → Terraform plan/apply |
| Unit + integration tests | ✅ | pytest; 15 test cases covering core logic and API contracts |
| OpenTelemetry tracing stub | ✅ | No-op by default; ready to wire to Jaeger/Tempo |

---

## Features Intentionally Excluded

| Feature | Reason |
|---|---|
| **Authentication / API keys** | Out of scope for a health check API in an internal system context. In production: add an API gateway or OAuth2 bearer token validation. |
| **Persistent storage** | Health checks are ephemeral by nature. Adding a DB would increase operational complexity without clear benefit for this assignment. |
| **Webhook / push notifications** | Would require a message broker (Kafka/SQS). Valuable for prod but out of scope here. |
| **Rate limiting** | Valuable in multi-tenant deployments; excluded to keep the solution focused. A middleware like `slowapi` would be a one-liner addition. |
| **gRPC / GraphQL interface** | REST is the simplest fit for this problem shape. |
| **Full OTLP tracing** | The tracer is wired as a no-op to avoid requiring a collector sidecar locally. Enabling it in prod requires adding `opentelemetry-exporter-otlp` and a collector URL. |

---

## Assumptions

1. **DAG validity**: The caller is responsible for providing a valid DAG. The API detects and rejects cycles (422 Unprocessable Entity) but does not attempt to repair them.
2. **Health URL is optional**: When `health_url` is omitted, the API uses the `status` field declared in the input payload. This enables testing/demo without requiring live services.
3. **HTTP 2xx = healthy**: Any non-2xx response from a `health_url` is treated as unhealthy. Custom thresholds (e.g. degraded state) are out of scope.
4. **Probe timeout**: 5 seconds per component. Configurable via environment variable in a production extension.
5. **No partial DAG**: All nodes referenced in edges must be declared in the `nodes` array.
6. **Infrastructure target**: AWS ECS Fargate. The Terraform code is illustrative of a real-world deployment pattern; actual `terraform apply` requires AWS credentials and an existing S3 state bucket.

---

## Key Design Decisions & Tradeoffs

### 1. FastAPI over Flask
FastAPI provides native `async/await` support (critical for concurrent health probing), automatic OpenAPI docs, and Pydantic validation — all with minimal boilerplate.

**Tradeoff**: Slightly higher learning curve than Flask for teams unfamiliar with async Python.

### 2. BFS traversal order
BFS was chosen as specified. It evaluates dependencies layer by layer, which maps naturally to the topological structure of the DAG (roots first, leaves last).

**Tradeoff**: We still evaluate all nodes concurrently regardless of BFS order (via `asyncio.gather`). BFS order is preserved in the response for transparency, but concurrent evaluation is more efficient than sequential layer-by-layer checking.

### 3. asyncio.gather for concurrency
All health checks run concurrently regardless of node depth. This minimises total wall-clock time.

**Alternative considered**: Sequential per-layer evaluation — would allow skipping leaf checks when a parent is unhealthy (propagated failure). Excluded because: the assignment asks for health of all components, and callers may want full visibility even if a dependency is down.

### 4. Pydantic v2 for data validation
Pydantic v2 provides fast, declarative validation with clear error messages. All invalid inputs result in a 422 with structured error details.

### 5. Multi-stage Docker build
Separates build-time dependencies (gcc, pip) from the runtime image, reducing the final image size by ~60%.

### 6. ECS Fargate over Kubernetes
Fargate removes node management overhead (no EC2 fleet to patch). For a single-service workload this is simpler and cost-effective.

**Tradeoff**: Less control over networking/scheduling than EKS. For a fleet of 50+ microservices, EKS would be preferable.

---

## Observability

| Signal | Implementation |
|---|---|
| **Structured logs** | JSON via `python-json-logger`; includes `component`, `status`, `latency_ms` per check |
| **Metrics** | Prometheus counters + histograms exposed at `/metrics` |
| **Tracing** | OpenTelemetry no-op stub; production: wire `OTLP_ENDPOINT` to Jaeger/Tempo |
| **Liveness probe** | `GET /health` → used by ECS/K8s to detect crashed containers |
| **Readiness probe** | `GET /ready` → used by ALB to gate traffic |
| **Container health check** | `HEALTHCHECK` in Dockerfile; ECS replaces unhealthy tasks |
| **CloudWatch Logs** | All stdout logs forwarded via `awslogs` log driver |
| **Container Insights** | Enabled on ECS cluster for CPU/memory dashboards |

---

## Quick Start

### Local (no Docker)
```bash
pip install -r requirements.txt
uvicorn app.main:app --reload --port 8080
```

### Docker
```bash
docker build -t system-health-api .
docker run -p 8080:8080 system-health-api
```

Visit **http://localhost:8080/docs** for the interactive Swagger UI.

---

## API Reference

### `POST /health-check`

**Request body:**
```json
{
  "nodes": [
    { "id": "string", "name": "string", "health_url": "https://...", "status": "healthy|unhealthy" }
  ],
  "edges": [
    { "source": "node_id", "target": "node_id" }
  ]
}
```
- `health_url` — optional. If provided, the API performs an HTTP GET.
- `status` — optional fallback used when `health_url` is absent.

**Response:**
```json
{
  "overall_status": "healthy | unhealthy",
  "total_components": 4,
  "healthy_count": 3,
  "unhealthy_count": 1,
  "bfs_order": ["db", "cache", "api", "web"],
  "components": [
    {
      "id": "db",
      "name": "Database",
      "status": "healthy",
      "message": "Declared status: healthy",
      "latency_ms": 11.2
    }
  ]
}
```

### `POST /health-check/visualize`
Same request shape. Response includes the report plus `dag_image_base64` (PNG).

### `GET /health` — Liveness probe
### `GET /ready` — Readiness probe
### `GET /metrics` — Prometheus metrics

---

## Sample Request & Response

The following example uses the **exact sample DAG provided in the assignment** — 11 steps with two failing nodes (Step 7 and Step 9).

**DAG structure:**
```
Step1 → Step2 → Step3 → Step5 → Step6 ──┐
                    └──→ Step7 → Step8 ──┼──→ Step10 → Step11
         └──→ Step4 → Step9 ─────────────┘
```

```bash
curl -s -X POST http://localhost:8080/health-check \
  -H "Content-Type: application/json" \
  -d '{
    "nodes": [
      {"id": "step1",  "name": "Step 1",  "status": "healthy"},
      {"id": "step2",  "name": "Step 2",  "status": "healthy"},
      {"id": "step3",  "name": "Step 3",  "status": "healthy"},
      {"id": "step4",  "name": "Step 4",  "status": "healthy"},
      {"id": "step5",  "name": "Step 5",  "status": "healthy"},
      {"id": "step6",  "name": "Step 6",  "status": "healthy"},
      {"id": "step7",  "name": "Step 7",  "status": "unhealthy"},
      {"id": "step8",  "name": "Step 8",  "status": "healthy"},
      {"id": "step9",  "name": "Step 9",  "status": "unhealthy"},
      {"id": "step10", "name": "Step 10", "status": "healthy"},
      {"id": "step11", "name": "Step 11", "status": "healthy"}
    ],
    "edges": [
      {"source": "step1",  "target": "step2"},
      {"source": "step2",  "target": "step3"},
      {"source": "step2",  "target": "step4"},
      {"source": "step3",  "target": "step5"},
      {"source": "step3",  "target": "step7"},
      {"source": "step5",  "target": "step6"},
      {"source": "step7",  "target": "step8"},
      {"source": "step4",  "target": "step9"},
      {"source": "step6",  "target": "step10"},
      {"source": "step8",  "target": "step10"},
      {"source": "step9",  "target": "step10"},
      {"source": "step10", "target": "step11"}
    ]
  }'
```

**Response:**
```json
{
  "overall_status": "unhealthy",
  "total_components": 11,
  "healthy_count": 9,
  "unhealthy_count": 2,
  "bfs_order": ["step1","step2","step3","step4","step5","step7","step9","step6","step8","step10","step11"],
  "components": [
    {"id": "step1",  "name": "Step 1",  "status": "healthy",   "message": "Declared status: healthy",   "latency_ms": 22.62},
    {"id": "step2",  "name": "Step 2",  "status": "healthy",   "message": "Declared status: healthy",   "latency_ms": 27.10},
    {"id": "step3",  "name": "Step 3",  "status": "healthy",   "message": "Declared status: healthy",   "latency_ms": 27.47},
    {"id": "step4",  "name": "Step 4",  "status": "healthy",   "message": "Declared status: healthy",   "latency_ms": 27.76},
    {"id": "step5",  "name": "Step 5",  "status": "healthy",   "message": "Declared status: healthy",   "latency_ms": 28.11},
    {"id": "step7",  "name": "Step 7",  "status": "unhealthy", "message": "Declared status: unhealthy", "latency_ms": 28.37},
    {"id": "step9",  "name": "Step 9",  "status": "unhealthy", "message": "Declared status: unhealthy", "latency_ms": 28.74},
    {"id": "step6",  "name": "Step 6",  "status": "healthy",   "message": "Declared status: healthy",   "latency_ms": 29.03},
    {"id": "step8",  "name": "Step 8",  "status": "healthy",   "message": "Declared status: healthy",   "latency_ms": 29.26},
    {"id": "step10", "name": "Step 10", "status": "healthy",   "message": "Declared status: healthy",   "latency_ms": 29.60},
    {"id": "step11", "name": "Step 11", "status": "healthy",   "message": "Declared status: healthy",   "latency_ms": 29.85}
  ]
}
```

**What this demonstrates:**

| Observation | Explanation |
|---|---|
| BFS order: `step1 → step2 → step3 → step4 → step5 → step7 → step9 → step6 → step8 → step10 → step11` | Roots evaluated first (step1), then breadth-wise layer by layer |
| Total wall-clock time ≈ 30ms for 11 nodes | All checks ran **concurrently** via `asyncio.gather` — not ~330ms sequential |
| `overall_status: unhealthy` | Correct — Step 7 and Step 9 are unhealthy |
| Step 10 and Step 11 show `healthy` | Full visibility across all nodes regardless of upstream failures |

---

## Running Tests

```bash
pip install -r requirements.txt
pytest tests/ -v --tb=short
```

15 test cases covering:
- DAG construction and edge validation
- BFS ordering (linear, diamond, disconnected)
- Cycle detection
- API endpoint contracts (happy path, all-healthy, cycle, single node)
- Liveness/readiness/metrics probes

---

## Infrastructure (Terraform)

```bash
cd terraform/
terraform init
terraform plan -var-file="environments/dev.tfvars"
terraform apply -var-file="environments/dev.tfvars"
```

Resources created:
- VPC with public + private subnets across 2 AZs
- Internet Gateway + NAT Gateway
- ECR repository (with image scanning)
- ECS Fargate cluster + service + task definition
- Application Load Balancer + target group
- Auto-scaling policy (target CPU 70%)
- CloudWatch Log Group (30-day retention)
- Security groups (ALB → ECS only)

---

## CI/CD Pipeline

On every **pull request**:
- Run linting (ruff) and all tests with coverage
- `terraform validate` + `terraform plan`

On **merge to main**:
- Run tests
- Build Docker image → push to GHCR
- `terraform apply` (requires `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` secrets)

---

## AI Tools Usage

**Tool used:** Claude (Anthropic)

| Area | How AI was used |
|---|---|
| **Terraform (AWS ECS/VPC)** | Used to generate the HCL boilerplate for VPC, ALB, ECS task definitions, IAM roles, and security groups — reviewed and adjusted for correctness. |
| **GitHub Actions pipeline** | Used to draft the 4-stage CI/CD YAML structure, including job dependencies (`needs:`), GHCR login steps, and conditional Terraform apply logic. |
| **Debugging CI failures** | Used to diagnose two pipeline failures: unused imports caught by `ruff`, and a `ModuleNotFoundError` due to missing `PYTHONPATH` in the test runner. |
| **README structure** | Used to organise and format this document. The content — assumptions, tradeoffs, and design decisions — is my own. |
