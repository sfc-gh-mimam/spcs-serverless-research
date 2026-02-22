# PRD: Snowflake Serverless Compute Platform

**Author:** Muzzammil Imam
**Status:** Draft
**Last Updated:** February 21, 2026
**Target Audience:** Engineering & Product Teams
**Stakeholders:** SPCS, Notebooks vNext, Streamlit vNext, ML Platform, OpenFlow, Platform Engineering

---

## Executive Summary

### The Problem

Four internal Snowflake teams—Notebooks vNext, Streamlit vNext, ML Platform, and OpenFlow—currently spend approximately 40% of their engineering time managing SPCS infrastructure for simple code execution. As SPCS becomes the platform of choice for more teams (given limitations of other compute options), this problem will compound. Deploying a basic Python function today requires writing ServiceSpec YAML, creating compute pools, building container images, and implementing token refresh logic—a process that can take 30+ minutes.

Teams are also locked into Kubernetes containers with no easy path to adopt newer technologies like microVMs, which offer significantly faster cold starts. Because SPCS APIs are hard-coded into products, migrating to better execution models would require extensive code rewrites across all teams.

### The Solution

We propose building a unified serverless interface using a decorator-based API (`@sf.function`) that simplifies deployment to just a few lines of code. Developers annotate their Python or Java functions with resource requirements, call `sf.deploy()`, and receive a production-ready endpoint—no YAML, no compute pools, no container management required.

The key architectural decision is backend abstraction. By decoupling the developer API from the execution infrastructure, teams can write code once and the platform can migrate between compute technologies (Kubernetes, microVMs, WebAssembly) without requiring code changes. This provides flexibility to adopt better execution models as they become available.

As SPCS adoption grows across Snowflake, this serverless interface will become increasingly critical. It enables teams to leverage SPCS without the infrastructure burden, making the platform accessible to many more teams beyond the current four.

### Scope

The platform targets user code execution: Python/Java functions, notebook runtimes, Streamlit applications, ML model serving, data processing jobs, and APIs. Complex orchestration frameworks like Ray, Temporal, Nifi, and Airflow will continue using SPCS directly, as they require Kubernetes primitives that serverless platforms don't provide.

Our goals: reduce deployment from 50+ lines to 5 lines, cut deployment time from 30+ minutes to under 1 minute, and achieve 80% adoption across internal teams by Month 6.

### Timeline

The rollout spans 12 months. Months 1-2 focus on internal alpha with Notebooks vNext and ML Platform. Months 3-4 expand to all four teams with GPU support and auto-scaling. Months 5-6 target internal GA with production monitoring. Months 7-8 introduce customer preview. Months 9-12 explore alternative backends like microVMs to validate that backend abstraction works in practice.

---

## 1. Problem Statement

### 1.1 Infrastructure Complexity Slows Feature Delivery

Internal teams spend **40% of engineering time** managing SPCS infrastructure for simple code execution instead of building product features.

**What deploying user code requires today:**
- 50+ lines of ServiceSpec YAML
- Compute pool creation and management
- Container image building and versioning
- Token refresh logic (60-minute OAuth expiry)
- CPU/memory/GPU estimation without tooling
- Networking, endpoints, volumes, secrets configuration

**Impact:**
- **6,400 engineering hours/year wasted** on infrastructure (Notebooks: 1,600h + Streamlit: 1,600h + ML: 1,600h + OpenFlow: 1,600h)
- **$960K-$1.3M lost productivity** annually ($150-200/hour fully loaded cost)
- 6 features/quarter instead of potential 10+ (across all 4 teams)
- Engineers frustrated managing infrastructure vs building product
- Competitive disadvantage vs AWS Lambda, Modal.ai

### 1.2 Architectural Lock-In

SPCS locks teams into Kubernetes containers. No path to adopt:
- **microVMs** (Firecracker, gVisor): 10x faster cold starts (<1s vs 30s)
- **WebAssembly**: Near-native speed, extreme isolation
- **Bare metal GPU**: Better utilization for large ML models

Teams hard-code SPCS APIs into products. Migrating to better backends means weeks of rewriting.

### 1.3 Affected Teams

Based on codebase analysis, these teams actively use SPCS today and face significant complexity:

| Team | Current SPCS Usage | Key Pain Points | Time Wasted |
|------|-------------------|-----------------|-------------|
| **Notebooks vNext** | Direct SPCS via `SYSTEM$NOTEBOOKS_VNEXT_CREATE_*` | ServiceSpec YAML, per-user isolation, token refresh (60min), compute pool management | 40% of 2 engineers |
| **Streamlit vNext** | Managed SPCS via `SYSTEM$STREAMLIT_BOOTSTRAP` | 4+ feature flags, bootstrap complexity, cold starts, token refresh | 40% of 2 engineers |
| **ML Platform** | Direct SPCS via `SYSTEM$DEPLOY_MODEL`, `SYSTEM$EXECUTE_ML_JOB` | Image build time (3-5 min), deploy spec complexity, resource estimation | 40% of 2 engineers |
| **OpenFlow** | SPCS as execution backend for workflow tasks | Orchestrating SPCS jobs, compute pool management, state tracking | 30% of 2 engineers |

**Note:** Cortex does NOT use SPCS - it uses managed services (`CREATE CORTEX SEARCH SERVICE`) and versioned stages, which is the model we should follow.

### 1.4 Current SPCS Usage Examples (From Codebase)

#### Notebooks vNext: Direct SPCS Integration

```python
# Current approach - Complex
class NotebookServiceAPI:
    def create(self, schema_name, service_name, compute_pool_name, service_spec, is_interactive=True):
        if is_interactive:
            return self.hybrid_executor.execute_query(
                f"select SYSTEM$NOTEBOOKS_VNEXT_CREATE_INTERACTIVE('{schema_name}', '{service_name}', '{compute_pool_name}', ${service_spec.to_yaml()}$)"
            )
        # Requires: ServiceSpec YAML, compute pool, custom container image
        # Problem: Per-user services, token refresh, manual pool management
```

#### Streamlit vNext: Managed SPCS with Bootstrap

```python
# Current approach - Feature flags + bootstrap complexity
# Requires 4+ feature flags:
# - ENABLE_STREAMLIT_SPCS_RUNTIME_V2
# - ENABLE_STREAMLIT_SPCS_RUNTIME
# - ENABLE_SNOWSERVICES_USER_FACING_FEATURES
# - ENABLE_STREAMLIT_PASS_ALTER_PROPERTIES_TO_SPCS_SERVICE

result = execute_query(f"SYSTEM$STREAMLIT_BOOTSTRAP('{streamlit_name}', '{session_id}')")
# Returns bootstrap JSON after multi-step job creation
# Problem: Complex state management, cold start latency, token refresh
```

#### ML Platform: Deploy Model with SPCS

```yaml
# Current approach - Complex YAML spec
models:
  - name: "my_model"
    version: "v1"

image_build:
  compute_pool: "GPU_POOL"
  image_repo: "repo.snowflakecomputing.com/db/schema/repo"
  force_rebuild: true

service:
  name: "model_service"
  compute_pool: "GPU_INFERENCE_POOL"
  max_instances: 1
  cpu: "4"
  num_workers: 1
  max_batch_rows: 1000

# Problems: Image build 3-5 min, resource guessing, complex YAML
```

#### OpenFlow: Orchestrating SPCS Tasks

```yaml
# Current approach - Full SPCS config per workflow task
tasks:
  - id: train_model
    type: spcs_training
    config:
      compute_pool: GPU_POOL
      image: pytorch-trainer:v1
      gpu: "A10G"
      command: ["python", "train.py"]
      # Problems: Compute pool management, state tracking, long deployment
```

---

## 2. Goals & Requirements

### 2.1 Functional Goals

| Goal | Requirement | Success Metric |
|------|-------------|----------------|
| **G1: Simple API** | Decorator-based, 5 lines of code | 90% code reduction |
| **G2: Fast Deploy** | <1 minute from code to running endpoint | 97% time reduction |
| **G3: Auto-scale** | 0 to N instances based on load | Zero manual scaling |
| **G4: Snowflake Native** | Direct table access, secrets, auth | Zero config overhead |
| **G5: Backend Agnostic** | Pluggable execution backends | Zero migration time for backend switches |

### 2.2 What We Support vs Don't

#### ✅ User Code Execution (In Scope)

| Use Case | Examples |
|----------|----------|
| Python/Java functions | Data transformations, ML inference, APIs |
| Notebook runtimes | Jupyter kernels, interactive sessions |
| Streamlit apps | Dashboards, data apps |
| ML model inference | Serving models, batch predictions |
| Data processing | ETL, aggregations, analytics |
| Scheduled tasks | Periodic jobs, cleanup |

**Principle:** If it's user-written code that needs to execute, serverless handles it.

#### ❌ Complex Frameworks (Out of Scope)

| Framework Type | Examples | Alternative |
|----------------|----------|-------------|
| Distributed ML runtimes | Ray, Dask, Spark | Use SPCS with K8s |
| Workflow orchestrators | Temporal, Airflow, Prefect | Use OpenFlow or SPCS |
| Stream processing | Flink, Nifi, Kafka Streams | Use SPCS with K8s |
| Stateful services | Databases, caches, queues | Use Snowflake or SPCS |
| Custom K8s workloads | StatefulSets, Operators | Use SPCS directly |

**Principle:** Complex frameworks need Kubernetes primitives. Use SPCS directly.

### 2.3 Non-Functional Requirements

| Requirement | Target | Measurement |
|-------------|--------|-------------|
| **Cold start time** | <5 seconds (SPCS), <1 second (microVM) | P50 latency |
| **Availability** | 99.9% uptime | Monthly SLA |
| **Scalability** | 0 to 1000 instances in <60 seconds | Load test |
| **Cost attribution** | Per-function, per-user metering | 100% accuracy |
| **Security** | SOC2, HIPAA, FedRAMP compliance | Audit pass |

---

## 3. User Personas

### 3.1 Primary: Internal Platform Teams (Confirmed SPCS Users)

**Notebooks vNext Team**
- **Current SPCS Usage:** Direct integration via `SYSTEM$NOTEBOOKS_VNEXT_CREATE_INTERACTIVE/NON_INTERACTIVE`
- **Needs:** Deploy notebook runtimes, GPU support, long-running sessions (hours), per-user isolation
- **Pain Points:**
  - ServiceSpec YAML for every notebook type
  - Token refresh every 60 minutes for long sessions
  - Per-user service creation (expensive resource overhead)
  - Custom container image maintenance
- **Success:** Deploy notebook kernel in <1 minute, no YAML, automatic token refresh

**Streamlit vNext Team**
- **Current SPCS Usage:** Managed SPCS via `SYSTEM$STREAMLIT_BOOTSTRAP`
- **Needs:** Deploy Streamlit apps, fast cold start (<5s), concurrent users, CDN integration
- **Pain Points:**
  - 4+ feature flags required (`ENABLE_STREAMLIT_SPCS_RUNTIME_V2`, etc.)
  - Complex bootstrap flow (job creation, timeout handling, state management)
  - Service lifecycle tied to Streamlit object
  - Cold start latency on first load
- **Success:** Deploy app in <1 minute, <5s cold start, zero feature flags

**ML Platform Team**
- **Current SPCS Usage:** Direct SPCS via `SYSTEM$DEPLOY_MODEL`, `SYSTEM$EXECUTE_ML_JOB`
- **Needs:** Deploy ML models for inference, run training jobs, GPU support (A10G, H100), batch predictions
- **Pain Points:**
  - Image build time: 3-5 minutes for PyTorch/TensorFlow
  - Complex deploy spec (YAML with models, image_build, service configs)
  - Resource estimation guesswork (CPU, memory, GPU, batch size, workers)
  - Multiple system functions for different operations
- **Success:** Deploy model endpoint in <1 minute, automatic resource estimation, A10G/H100 support

**OpenFlow Team**
- **Current SPCS Usage:** SPCS as execution backend for workflow orchestration
- **Needs:** Orchestrate multi-step pipelines, coordinate SPCS jobs/services, handle dependencies and retries
- **Pain Points:**
  - Each workflow step requires full SPCS configuration
  - Compute pool management across multiple tasks
  - State tracking and failure handling complexity
  - Long DAG deployment times (minutes per workflow)
- **Success:** Define workflow as Python functions, deploy entire DAG in <3 minutes, automatic state management

### 3.2 Cortex Team (NOT a SPCS user - reference model)

**Why Cortex is not included:**
- Cortex uses **managed services** (`CREATE CORTEX SEARCH SERVICE`) and versioned stages
- No ServiceSpec YAML, no compute pools, no container images
- Single SQL statement deployment - **this is the developer experience we want to replicate**

### 3.3 Secondary: Snowflake Customers (Future)

- Data scientists deploying ML models
- Developers building APIs on Snowflake data
- Analysts creating dashboards

---

## 4. Proposed Solution

### 4.1 Core API

```python
import snowflake.serverless as sf

# Serverless function
@sf.function(
    cpu=2,              # vCPUs (0.5 to 16)
    memory="4GB",       # Memory (1GB to 64GB)
    gpu="A10G",         # Optional: A10G, H100
    timeout=300,        # Max execution seconds
    secrets=["api_key"] # Snowflake secrets
)
def my_function(data: dict) -> dict:
    # Direct Snowflake table access
    session = sf.get_session()
    df = session.table("MY_TABLE")
    return process(df, data)

# Deploy
endpoint = sf.deploy(my_function)
print(endpoint.url)  # https://<account>.snowflakecomputing.com/api/v1/functions/<id>

# Invoke
result = endpoint.invoke({"key": "value"})

# Update
sf.deploy(my_function, update=True)

# Delete
endpoint.delete()
```

### 4.2 Streamlit Apps

```python
import snowflake.serverless as sf

@sf.app(
    cpu=1,
    memory="2GB",
    concurrent_users=10
)
def my_app():
    import streamlit as st
    st.title("My Dashboard")
    # Direct table access works
    df = st.session_state.snowflake_session.table("MY_TABLE")
    st.dataframe(df)

endpoint = sf.deploy(my_app)
print(endpoint.url)  # https://<account>.snowflakecomputing.com/apps/<id>
```

### 4.3 Scheduled Tasks

```python
@sf.scheduled(
    cron="0 * * * *",  # Every hour
    cpu=2,
    memory="4GB"
)
def hourly_job():
    # Process data
    session = sf.get_session()
    session.sql("CALL my_stored_proc()").collect()

sf.deploy(hourly_job)
```

### 4.4 Snowflake Integration

**Built-in capabilities:**
- **Session management:** `sf.get_session()` returns authenticated Snowflake session
- **Secrets:** Automatic injection from Snowflake secrets
- **Table access:** Direct read/write to Snowflake tables
- **UDF integration:** Functions can be called from SQL
- **Auth:** Inherits Snowflake RBAC, no additional auth setup

---

## 5. Technical Architecture

### 5.1 System Components

```
┌─────────────────────────────────────────────────────────────┐
│                      SNOWFLAKE SERVERLESS                    │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌───────────────┐    ┌───────────────┐    ┌──────────────┐ │
│  │   SDK/CLI     │───▶│  Control Plane │───▶│   Backend    │ │
│  │               │    │                │    │   Adapter    │ │
│  │ • Decorators  │    │ • Deploy API   │    │              │ │
│  │ • sf.deploy() │    │ • Image Build  │    │ • SPCS       │ │
│  │ • Endpoint    │    │ • Spec Gen     │    │ • microVM    │ │
│  │   client      │    │ • Pool Mgmt    │    │ • Future     │ │
│  └───────────────┘    └───────────────┘    └──────────────┘ │
│                              │                               │
│                              ▼                               │
│                    ┌───────────────┐                        │
│                    │  Shared Infra  │                        │
│                    │ • Image Reg    │                        │
│                    │ • Pool Fleet   │                        │
│                    │ • Warm Pools   │                        │
│                    │ • Metrics      │                        │
│                    └───────────────┘                        │
└─────────────────────────────────────────────────────────────┘
```

### 5.2 Deployment Flow

```
1. Developer: sf.deploy(my_function)
   │
2. SDK: Analyze function
   • Parse decorator parameters
   • Detect dependencies
   • Hash code for caching
   │
3. Control Plane: Receive request
   • Validate parameters
   • Check quotas
   │
4. Image Builder: Build container (if needed)
   • Generate Dockerfile
   • Build and push to registry
   • Cache by code hash
   │
5. Backend Adapter: Deploy to execution backend
   • SPCS: Generate ServiceSpec, deploy to K8s
   • microVM: Generate Firecracker config, deploy to fleet
   │
6. Return endpoint URL to user
```

### 5.3 Backend Abstraction Layer

**Key design:** Decouple API from execution backend

```python
# Backend interface
class ExecutionBackend:
    def deploy(self, function_spec: FunctionSpec) -> Endpoint
    def invoke(self, endpoint: Endpoint, input: dict) -> dict
    def scale(self, endpoint: Endpoint, instances: int)
    def delete(self, endpoint: Endpoint)
    def get_metrics(self, endpoint: Endpoint) -> Metrics

# Implementations
class SPCSBackend(ExecutionBackend):
    """Kubernetes-based execution via SPCS"""

class MicroVMBackend(ExecutionBackend):
    """Firecracker/gVisor-based execution"""

class WasmBackend(ExecutionBackend):
    """WebAssembly-based execution (future)"""
```

**Benefits:**
- Teams write code once, platform can switch backends
- Experiment with new technologies without breaking consumers
- Gradual migration (canary roll-outs, A/B testing)

### 5.4 Warm Pool Fleet

**Challenge:** Cold starts (30-60s) are too slow

**Solution:** Maintain warm pools of pre-provisioned capacity

| Tier | Pool Size | Target Cold Start |
|------|-----------|-------------------|
| XS (0.5 vCPU, 1GB) | 20 instances | <5s |
| S (1 vCPU, 2GB) | 30 instances | <5s |
| M (2 vCPU, 4GB) | 20 instances | <5s |
| L (4 vCPU, 16GB) | 10 instances | <10s |
| XL (8 vCPU, 32GB) | 5 instances | <15s |
| GPU-S (A10G) | 3 instances | <20s |
| GPU-L (H100) | 2 instances | <30s |

**Cost:** ~$300K/year for warm pools

---

## 6. Detailed Requirements

### 6.1 Functional Requirements

#### FR-1: Function Deployment
- **REQ:** Deploy Python/Java function with single API call
- **INPUT:** Function code + resource specs (CPU, memory, GPU)
- **OUTPUT:** Endpoint URL within <1 minute
- **VALIDATION:** Code analysis, dependency detection, quota checks

#### FR-2: Resource Tiers
- **REQ:** Pre-configured resource tiers (XS, S, M, L, XL, GPU)
- **TIERS:** 0.5-16 vCPU, 1-64GB memory, optional GPU (A10G, H100)
- **VALIDATION:** Validate tier exists, user has quota

#### FR-3: Auto-scaling
- **REQ:** Scale from 0 to N instances based on load
- **MIN:** 0 instances (cost optimization)
- **MAX:** Configurable per function (default: 100)
- **TRIGGER:** Request queue depth, CPU utilization

#### FR-4: Snowflake Integration
- **REQ:** Automatic session injection, table access, secrets
- **SESSION:** `sf.get_session()` returns authenticated session
- **SECRETS:** Decorator parameter `secrets=["name"]` injects env vars
- **AUTH:** Inherits Snowflake RBAC

#### FR-5: Monitoring & Observability
- **REQ:** Real-time metrics, logs, traces
- **METRICS:** Invocations, latency (P50/P95/P99), errors, cost
- **LOGS:** Automatic collection, searchable via Snowsight
- **TRACES:** Distributed tracing for debugging

#### FR-6: Cost Attribution
- **REQ:** Per-function, per-user metering
- **GRANULARITY:** 1-second billing increments
- **ATTRIBUTION:** Tag functions by team, project, customer
- **REPORTING:** Cost dashboard in Snowsight

### 6.2 Non-Functional Requirements

#### NFR-1: Performance
- **Cold start:** <5s (SPCS), <1s (microVM target)
- **Warm start:** <100ms
- **Throughput:** 1000 req/s per function
- **Latency:** P99 < 500ms (application code)

#### NFR-2: Reliability
- **Availability:** 99.9% uptime SLA
- **Durability:** Code and config replicated 3x
- **Fault tolerance:** Automatic retry on failure (3x)
- **Graceful degradation:** Fallback to SPCS if microVM fails

#### NFR-3: Security
- **Isolation:** Functions run in isolated containers/microVMs
- **Auth:** RBAC-based access control
- **Secrets:** Encrypted at rest and in transit
- **Compliance:** SOC2, HIPAA, FedRAMP ready

#### NFR-4: Scalability
- **Concurrent functions:** 10,000+ per account
- **Instances per function:** 0 to 1,000
- **Scale-out time:** <60 seconds to 1000 instances
- **Control plane:** 100+ deployments/minute

---

## 7. Rollout Plan

### Phase 1: Internal Alpha (Months 1-2)

**Goals:**
- Validate core API with 1-2 internal teams
- Deploy Python functions with basic tiers (S, M, L)
- Prove <1 minute deployment time

**Scope:**
- Python support only
- 3 resource tiers (S, M, L)
- SPCS backend only
- Limited to 10 functions per team

**Alpha Teams:**
- **Notebooks vNext** (primary) - Interactive notebook kernels
- **ML Platform** (secondary) - Model inference endpoints

**Success Criteria:**
- 10+ functions deployed by Notebooks vNext team
- 5+ model endpoints deployed by ML Platform team
- 90% code reduction achieved
- <1 minute deployment time (P95)
- Zero critical bugs

### Phase 2: Internal Beta (Months 3-4)

**Goals:**
- Expand to all 4 internal teams (Notebooks vNext, Streamlit vNext, ML Platform, OpenFlow)
- Add GPU support (A10G)
- Implement auto-scaling

**Scope:**
- All resource tiers (XS, S, M, L, XL, GPU-S)
- Auto-scaling (0 to N instances)
- Cost attribution and dashboards
- Increase to 50 functions per team

**Beta Teams (All 4):**
- **Notebooks vNext** - Notebook kernels, GPU sessions
- **Streamlit vNext** - Streamlit apps, dashboards
- **ML Platform** - Model inference, training jobs
- **OpenFlow** - Workflow task execution

**Success Criteria:**
- 50+ functions deployed across 4 teams
- GPU functions running (ML Platform team with A10G)
- Auto-scaling working (0→10→0 instances)
- OpenFlow workflow integration working
- 80%+ developer satisfaction

### Phase 3: Internal GA (Months 5-6)

**Goals:**
- 80% adoption by internal teams
- Production-ready monitoring and alerting
- Migration tooling for existing SPCS deployments

**Scope:**
- Remove function limits
- Full monitoring suite (metrics, logs, traces)
- Migration scripts (SPCS → Serverless)
- SLA enforcement (99.9% uptime)

**Success Criteria:**
- 80%+ of internal workloads migrated (across all 4 teams)
- <5 critical incidents/month
- 90%+ developer satisfaction
- $1.3M/year cost savings achieved (6,400 hours × $200/hour)
- $1.4M/year cost savings achieved

### Phase 4: Customer Preview (Months 7-8)

**Goals:**
- Beta with select customers
- Validate pricing model
- Customer-facing docs and examples

**Scope:**
- Public documentation
- Customer onboarding flow
- Billing integration
- Support runbooks

**Success Criteria:**
- 50+ customer functions deployed
- <10% churn due to issues
- Pricing model validated

### Phase 5: Alternative Backends (Months 9-12)

**Goals:**
- Experiment with microVM backend (Firecracker)
- Prove backend abstraction works
- Achieve <1s cold starts

**Scope:**
- MicroVM backend implementation
- A/B testing framework
- Automatic backend migration
- Performance benchmarking

**Success Criteria:**
- MicroVM backend operational
- <1s cold start time achieved
- Zero migration time for teams (automatic)

---

## 8. Pricing Strategy

### 8.1 Pricing Model

**COGS + 10% base layer with partner markup framework**

- **SPCS charges:** Infrastructure costs + 10% (cost recovery only)
- **Partner teams:** Apply 2-5x markup based on value proposition
- **Billing data:** SPCS exports consumption data to partners

### 8.2 Base Pricing (COGS + 10%)

| Tier | vCPU | Memory | $/second | $/hour |
|------|------|--------|----------|--------|
| XS | 0.5 | 1 GB | $0.000009 | $0.032 |
| S | 1 | 2 GB | $0.000016 | $0.058 |
| M | 2 | 4 GB | $0.000030 | $0.108 |
| L | 4 | 16 GB | $0.000066 | $0.238 |
| XL | 8 | 32 GB | $0.000130 | $0.468 |
| GPU-S | 4 + A10G | 16 GB | $0.000494 | $1.778 |
| GPU-L | 8 + H100 | 32 GB | $0.001352 | $4.867 |

**Billing granularity:** 1-second increments

### 8.3 Partner Markup Guidelines

| Partner | Multiplier | Rationale | Margin |
|---------|-----------|-----------|--------|
| Notebooks | 2.0-2.5x | Session management, collaboration | 50-60% |
| Streamlit | 2.5-3.0x | App hosting, CDN, domains | 60-67% |
| Cortex | 3.0-4.0x | Model optimization, vector DB | 67-75% |

### 8.4 Competitive Positioning

**CPU workloads** (1M requests/month, 2 vCPU, 4GB):
- AWS Lambda: $0.41
- Snowflake (2.5x markup): $9.90

**GPU workloads** (A10G, 10 hours/month):
- AWS SageMaker: $1,200 (always-on)
- Modal.ai: $74.88
- Snowflake (3.5x markup): $62.23 ✅ 17-48% cheaper

---

## 9. Open Questions & Decisions

### 9.1 Technical Decisions

| Question | Options | Recommendation | Rationale |
|----------|---------|----------------|-----------|
| **Backend abstraction complexity** | Build from day 1 vs defer to Phase 5 | Build from day 1 | Enables future flexibility, manageable overhead |
| **Multi-language support** | Python only vs Python + Java | Python first, Java in Phase 2 | 80% of use cases are Python |
| **Warm pool sizing** | Fixed vs dynamic | Dynamic with min thresholds | Cost optimization while maintaining performance |
| **Cold start target** | <5s vs <1s | <5s for Phase 1-3, <1s for Phase 5 (microVM) | Realistic with SPCS, aspirational with microVM |

### 9.2 Product Decisions

| Question | Options | Recommendation | Rationale |
|----------|---------|----------------|-----------|
| **Customer GA timeline** | Month 7 vs Month 12 | Month 7 (Customer Preview) | Early validation, controlled rollout |
| **Pricing model** | Usage-based vs subscription | Hybrid (subscription with usage overage) | Predictable revenue, aligns with usage |
| **Free tier** | Yes vs No | Yes (10 compute-hours/month) | Customer acquisition, trial |
| **SQL integration** | UDF syntax vs separate API | Both (decorator registers as UDF) | Flexibility for SQL users |

### 9.3 Partnership Decisions

| Question | Recommendation |
|----------|----------------|
| **OpenFlow integration timing** | Phase 3 (Month 5-6) - after internal GA |
| **Notebooks vNext migration strategy** | Gradual (20% → 50% → 80% over 3 months), focus on new notebooks first |
| **Streamlit vNext backward compatibility** | Maintain existing SPCS path for 12 months, deprecate feature flags |
| **ML Platform GPU priority** | A10G in Phase 2, H100 in Phase 3 |
| **ML Platform image build optimization** | Cache layers aggressively, 3-5 min → <1 min target |

---

## 10. Success Metrics

### 10.1 Adoption Metrics

| Metric | Target | Measurement |
|--------|--------|-------------|
| Internal team adoption | 80% by Month 6 (4 teams) | % of workloads using serverless |
| Function count | 150+ by Month 6 | Total deployed functions across all teams |
| Daily active functions | 75+ by Month 6 | Functions invoked in last 24h |
| Team satisfaction | 90%+ by Month 6 | Quarterly survey (NPS) across Notebooks vNext, Streamlit vNext, ML, OpenFlow |

### 10.2 Performance Metrics

| Metric | Target | Measurement |
|--------|--------|-------------|
| Deployment time | <1 minute (P95) | Time from deploy() to ready |
| Cold start time | <5 seconds (P95) | First invocation latency |
| Warm start time | <100ms (P95) | Subsequent invocations |
| Availability | 99.9% | Uptime SLA |

### 10.3 Efficiency Metrics

| Metric | Target | Measurement |
|--------|--------|-------------|
| Code reduction | 90% | Lines of infra code before/after |
| Engineering time saved | 320 hours/month | Time tracking by 4 teams (4 teams × 2 eng × 40% time) |
| Feature velocity | 67% increase | Features shipped per quarter (across all teams) |
| Cost savings | $1.3M/year | 6,400 hours/year × $200/hour fully loaded cost |

### 10.4 Financial Metrics

| Metric | Target | Measurement |
|--------|--------|-------------|
| Gross margin (SPCS) | 10% | By design (COGS + 10%) |
| Gross margin (Partners) | 60-70% | Partner revenue - SPCS costs |
| Customer acquisition | 50+ by Month 8 | Paying customers |
| Revenue (Year 1) | $1M run rate | Annualized from Month 12 |

---

## 11. Dependencies

### 11.1 Internal Dependencies

| Team | Dependency | Impact | Mitigation |
|------|-----------|--------|-----------|
| **SPCS Core** | ServiceSpec API, compute pools, image registry | Critical - blocks all work | Early engagement, dedicated liaison |
| **Platform Eng** | Token management, auth, secrets | Critical - blocks Phase 1 | Reuse existing patterns, automatic refresh |
| **OpenFlow** | Workflow integration API | Medium - blocks Phase 3 only | Can proceed with other teams, adds value in Phase 3 |
| **Notebooks vNext** | Testing feedback, migration support | High - primary alpha user | Weekly syncs, dedicated support |
| **Streamlit vNext** | Feature flag deprecation, bootstrap simplification | High - beta user | Collaborate on managed pattern |
| **ML Platform** | Image build optimization, resource estimation | High - GPU workflows critical | Joint optimization efforts |
| **Snowsight** | Cost dashboard, logs UI | Low - nice to have | Use existing Snowsight components |

### 11.2 External Dependencies

| Vendor | Dependency | Impact | Mitigation |
|--------|-----------|--------|-----------|
| **AWS** | Firecracker for microVMs | Low - Phase 5 only | Can use gVisor alternative |
| **Cloud providers** | COGS pricing stability | Medium - affects pricing model | Lock in contracts, pass increases to customers |

---

## 12. Risks & Mitigations

### 12.1 Technical Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|-----------|
| **Cold starts remain slow** | Medium | High | Warm pools (Phase 1-3), microVM migration (Phase 5) |
| **Backend abstraction overhead** | Low | Medium | Simple interface, proven pattern (Modal.ai, AWS Lambda) |
| **SPCS performance limits** | Low | High | Early load testing, direct engagement with SPCS team |
| **Token refresh complexity** | Low | Medium | Reuse existing patterns, automatic renewal |

### 12.2 Adoption Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|-----------|
| **Teams resist change** | Medium | High | Migration tools, docs, PM engagement, phased rollout |
| **Missing features** | Medium | Medium | Prioritize top 3 blockers from each team |
| **Performance regression** | Low | High | Benchmarking, SLAs, automatic rollback |

### 12.3 Financial Risks

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|-----------|
| **Cost overruns** | Medium | Medium | Aggressive auto-suspend, quotas, monitoring |
| **Warm pool waste** | Medium | Low | Dynamic sizing, usage-based adjustments |
| **Partner pricing too high** | Low | Medium | Pricing governance, competitive benchmarking |

---

## 13. OpenFlow Integration

### 13.1 Overview

OpenFlow is Snowflake's workflow orchestration framework. With serverless, each workflow step becomes a simple Python function instead of full SPCS configuration.

### 13.2 API Design

```python
import snowflake.serverless as sf
from snowflake.openflow import Workflow

# Define workflow tasks as serverless functions
@sf.function(cpu=2, memory="4GB")
def extract_data(params):
    session = sf.get_session()
    df = session.table("RAW_DATA")
    return df.filter(params["filter"]).to_pandas()

@sf.function(cpu=4, memory="16GB")
def train_model(data):
    import sklearn
    model = sklearn.train(data)
    return model.serialize()

@sf.function(cpu=2, memory="8GB", gpu="A10G")
def deploy_model(model_bytes):
    # Deploy for inference
    return {"endpoint": "https://..."}

# Create workflow
workflow = Workflow("ml_pipeline")
workflow.add_task("extract", extract_data)
workflow.add_task("train", train_model, depends_on=["extract"])
workflow.add_task("deploy", deploy_model, depends_on=["train"])

# Deploy workflow (creates all functions + orchestration)
workflow.deploy()

# Trigger execution
workflow.run({"filter": "date > '2026-01-01'"})
```

### 13.3 Benefits

| Benefit | Current State (SPCS) | Future State (Serverless) |
|---------|---------------------|--------------------------|
| Code per workflow | 200+ lines (3 ServiceSpecs) | 50 lines (3 functions) |
| Deployment time | 90+ minutes (3 services) | <3 minutes (parallel deploy) |
| Cost | Always-on pools | Pay per execution |
| Debugging | Check 3 service logs | Single unified trace |

---

## 14. Appendix

### 14.1 Glossary

| Term | Definition |
|------|------------|
| **SPCS** | Snowpark Container Services - Snowflake's Kubernetes-based container platform |
| **ServiceSpec** | YAML specification for SPCS services |
| **Compute Pool** | Group of compute nodes for running containers |
| **Warm Pool** | Pre-provisioned capacity for fast cold starts |
| **microVM** | Lightweight virtual machine (Firecracker, gVisor) |
| **Backend Abstraction** | Layer decoupling API from execution infrastructure |
| **OpenFlow** | Snowflake's workflow orchestration framework |

### 14.2 Competitive Analysis

| Provider | Cold Start | GPU Support | Pricing (CPU) | Pricing (GPU) | Snowflake Integration |
|----------|-----------|-------------|---------------|---------------|----------------------|
| **AWS Lambda** | 1-3s | ❌ No | $0.41/1M req | N/A | ❌ External |
| **Modal.ai** | 1-2s | ✅ A10G, H100 | $10/1M req | $74/10h A10G | ❌ External |
| **GCP Cloud Run** | 2-5s | ❌ No | $5.60/1M req | N/A | ❌ External |
| **Azure ACI** | 5-10s | ✅ Limited | $2.40/1M req | Expensive | ❌ External |
| **Snowflake Serverless** | <5s (Phase 1-3) | ✅ A10G, H100 | $9.90/1M req | $62/10h A10G | ✅ Native |

### 14.3 References

- [AWS Lambda Pricing](https://aws.amazon.com/lambda/pricing/)
- [Modal.ai Documentation](https://modal.com/docs)
- [Firecracker microVM](https://firecracker-vm.github.io/)
- [SPCS Documentation](https://docs.snowflake.com/en/developer-guide/snowpark-container-services)

---

**Document Version:** 2.0
**Last Updated:** February 21, 2026
**Next Review:** March 2026
