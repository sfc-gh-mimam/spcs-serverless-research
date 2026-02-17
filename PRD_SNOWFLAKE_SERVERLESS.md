# PRD: Snowflake Serverless Compute Platform

**Author:** Muzzammil Imam
**Status:** Draft
**Last Updated:** February 2026
**Stakeholders:** SPCS, Notebooks, Streamlit, Cortex, Platform Engineering

---

## 1. Executive Summary

### 1.1 Problem Statement

Internal Snowflake teams (Notebooks vNext, Streamlit/SiS, Cortex) spend significant engineering effort managing SPCS infrastructure instead of building product features. Currently, deploying code to SPCS requires:

- Writing 50+ lines of ServiceSpec YAML
- Creating and managing compute pools
- Building, pushing, and versioning container images
- Implementing token refresh logic (60-min OAuth expiry)
- Estimating CPU/memory/GPU requirements

**This infrastructure burden slows down development and creates inconsistent patterns across teams.**

Additionally, **SPCS locks teams into the current container-based architecture**. There is no easy way to experiment with alternative execution models (microVMs, Firecracker, WebAssembly, etc.) without requiring every consumer team to rewrite their integration code. This architectural rigidity prevents the platform from evolving to adopt better technologies as they emerge.

### 1.2 Proposed Solution

Build a **Modal.ai-style serverless interface** that lets developers:

1. Write code in any language
2. Decorate functions with resource requirements
3. Deploy instantly with a single command
4. Get endpoints automatically
5. **Remain agnostic to the underlying execution model**

The platform handles all infrastructure behind the scenes. Critically, **the unified interface decouples developer code from the execution backend**, allowing Snowflake to experiment with and migrate between different compute architectures (containers on Kubernetes today, microVMs tomorrow, Firecracker, WebAssembly, etc.) without requiring any changes to consumer code.

### 1.3 Success Metrics

| Metric | Current | Target |
|--------|---------|--------|
| Lines of code to deploy | 50+ | 5 |
| Time to first deployment | 30+ minutes | < 1 minute |
| Infrastructure concepts to learn | 8+ | 2 |
| Internal team adoption | 0% | 80% in 6 months |
| Time to migrate to new backend | Weeks (full rewrite) | Zero (transparent) |

---

## 2. Background & Motivation

### 2.1 Current State

Teams integrating with SPCS must manage:

```
┌─────────────────────────────────────────────────────────────┐
│                    DEVELOPER RESPONSIBILITIES                │
├─────────────────────────────────────────────────────────────┤
│  1. Compute Pool         CREATE COMPUTE POOL, sizing, DDL   │
│  2. Container Image      Dockerfile, build, push, version   │
│  3. ServiceSpec YAML     containers, endpoints, volumes...  │
│  4. Service Creation     CREATE SERVICE SQL                 │
│  5. Token Management     60-min refresh logic               │
│  6. Endpoint Config      public/private, CORS, auth         │
│  7. Scaling              min/max nodes, auto-suspend        │
│  8. Monitoring           logs, metrics, health checks       │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 Pain Points from Research

| Pain Point | Severity | Evidence |
|------------|----------|----------|
| ServiceSpec complexity | High | 50+ lines for simple service |
| Compute pool management | High | Manual DDL, sizing guesswork |
| Image management | High | DevOps overhead for every change |
| Token refresh | Medium | Session failures in production |
| Resource estimation | Medium | Over/under provisioning |
| Cold start | Medium | Minutes to start, poor UX |
| **Architectural lock-in** | **High** | **Cannot experiment with microVMs, Firecracker, or alternative runtimes without rewriting all consumer integrations** |

### 2.3 What Works Well

**Cortex Search** uses a simplified pattern that abstracts infrastructure:

```sql
CREATE CORTEX SEARCH SERVICE my_search
  ON text_column
  WAREHOUSE = my_warehouse
  AS SELECT * FROM my_table;
```

No compute pools. No ServiceSpec. No images. **This is the model to follow.**

---

## 3. Goals & Non-Goals

### 3.1 Goals

| Goal | Description |
|------|-------------|
| **G1: Zero Infrastructure** | Developers never see compute pools, ServiceSpec, or images |
| **G2: Code-First** | Write code → deploy → get endpoint. That's it. |
| **G3: Any Language** | Python native, containers for Go/Rust/Java/etc. |
| **G4: Instant Deploy** | < 1 minute from code change to running endpoint |
| **G5: Snowflake Native** | Automatic session, data access, secrets, auth |
| **G6: Production Ready** | Auto-scaling, monitoring, cost attribution built-in |
| **G7: Backend Agnostic** | Unified interface allows switching execution models (Kubernetes → microVMs → Firecracker) without code changes |

### 3.2 Non-Goals (V1)

| Non-Goal | Rationale |
|----------|-----------|
| Replace SPCS entirely | Power users may still need direct SPCS access |
| Support all SPCS features | Focus on 80% use case first |
| Multi-cloud parity | Start with AWS, expand later |
| Custom networking | Use standard Snowflake networking |

---

## 4. User Personas

### 4.1 Primary: Internal Platform Teams

**Notebooks vNext Team**
- Needs: Deploy notebook runtimes, GPU support, long sessions
- Pain: ServiceSpec complexity, token refresh, per-user isolation

**Streamlit/SiS Team**
- Needs: Deploy Streamlit apps, fast cold start, concurrent users
- Pain: Feature flags, compute pool management, scaling

**Cortex Team**
- Needs: Deploy ML models, inference endpoints, batch jobs
- Pain: GPU provisioning, resource estimation, image management

### 4.2 Secondary: Snowflake Customers (Future)

- Data scientists deploying ML models
- App developers building internal tools
- Analysts creating dashboards

---

## 5. Proposed Solution

### 5.1 Core Concept

```
┌─────────────────────────────────────────────────────────────┐
│                         DEVELOPER                            │
│                                                              │
│   @sf.function(cpu=2, memory="4GB")                         │
│   def my_function(data):                                     │
│       return process(data)                                   │
│                                                              │
│   endpoint = sf.deploy(my_function)                          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    PLATFORM (INVISIBLE)                      │
├─────────────────────────────────────────────────────────────┤
│  • Analyze code, detect dependencies                         │
│  • Build container image automatically                       │
│  • Generate ServiceSpec from decorator                       │
│  • Select/create compute pool                                │
│  • Deploy service, wait for ready                            │
│  • Generate endpoint URL with auth                           │
│  • Handle token refresh transparently                        │
│  • Auto-scale based on load                                  │
│  • Track costs per function/user                             │
└─────────────────────────────────────────────────────────────┘
```

### 5.2 API Design

#### 5.2.1 Core Decorators

```python
import snowflake.serverless as sf

# Serverless function endpoint
@sf.function(
    cpu=2,              # vCPUs (0.5 to 16)
    memory="4GB",       # Memory (1GB to 64GB)
    gpu="A10G",         # Optional: A10G, H100, None
    timeout=300,        # Max execution seconds
    secrets=["api_key"] # Snowflake secrets to inject
)
def my_function(data: dict) -> dict:
    return process(data)

# Streamlit app
@sf.app(
    cpu=1,
    memory="2GB",
    concurrent_users=10
)
def my_streamlit_app():
    import streamlit as st
    st.title("My App")

# Scheduled batch job
@sf.job(
    cpu=8,
    memory="32GB",
    gpu="H100",
    schedule="0 0 * * *"  # Cron: daily at midnight
)
def nightly_training():
    train_model()

# Long-running service
@sf.service(
    cpu=2,
    memory="4GB",
    min_instances=1,
    max_instances=10
)
def my_api_server():
    from flask import Flask
    app = Flask(__name__)
    # ...
```

#### 5.2.2 Resource Tiers (Alternative to Explicit Resources)

```python
# Instead of specifying exact resources, use tiers
@sf.function(tier="medium")
def my_function(data):
    return process(data)

@sf.function(tier="gpu-small")
def ml_inference(data):
    return model.predict(data)
```

| Tier | CPU | Memory | GPU | Monthly Cost Est. |
|------|-----|--------|-----|-------------------|
| `xs` | 0.5 | 1 GB | - | $5 |
| `small` | 1 | 2 GB | - | $10 |
| `medium` | 2 | 4 GB | - | $20 |
| `large` | 4 | 16 GB | - | $50 |
| `xl` | 8 | 32 GB | - | $100 |
| `gpu-small` | 4 | 16 GB | A10G | $150 |
| `gpu-large` | 8 | 32 GB | A10G x4 | $400 |
| `gpu-xl` | 16 | 64 GB | H100 | $800 |

#### 5.2.3 Deployment Methods

```python
# Deploy decorated function
endpoint = sf.deploy(my_function)

# Deploy from container image (any language)
endpoint = sf.container(
    image="my-go-service:latest",
    port=8080,
    tier="medium"
)

# Deploy from stage
endpoint = sf.from_stage(
    stage="@my_code/app",
    entrypoint="main.py",
    tier="large"
)

# Deploy notebook
notebook = sf.notebook(
    source="@notebooks/analysis.ipynb",
    tier="gpu-small"
)
```

#### 5.2.4 Endpoint Operations

```python
# Get endpoint info
print(endpoint.url)          # https://xxx.snowflakecomputing.app
print(endpoint.status)       # READY, STARTING, FAILED
print(endpoint.logs())       # Container logs

# Call endpoint
result = endpoint.call({"input": "data"})

# Async call
future = endpoint.call_async({"input": "data"})
result = future.result()

# Manage lifecycle
endpoint.scale(min_instances=2, max_instances=10)
endpoint.stop()
endpoint.start()
endpoint.delete()

# Get metrics
print(endpoint.metrics())    # Invocations, latency, errors, cost
```

### 5.3 Snowflake Integration

```python
@sf.function(tier="medium")
def query_and_process():
    # Snowflake session is automatically available
    df = session.sql("SELECT * FROM my_table").to_pandas()
    return process(df)

@sf.function(tier="medium", secrets=["openai_key"])
def call_llm():
    import os
    # Secrets automatically injected as env vars
    api_key = os.environ["OPENAI_KEY"]
    return call_openai(api_key)
```

### 5.4 Multi-Language Support

#### Python (Native - Zero Config)
```python
@sf.function(tier="medium")
def python_function(data):
    import pandas as pd
    return pd.DataFrame(data).to_dict()
```

#### Container (Any Language)
```python
# Go, Rust, Java, Node.js, etc.
endpoint = sf.container(
    image="my-rust-service:latest",
    port=8080,
    tier="medium",
    env={"LOG_LEVEL": "debug"}
)
```

#### Dockerfile (Build from Source)
```python
endpoint = sf.container(
    dockerfile="./Dockerfile",
    context="./src",
    tier="medium"
)
```

---

## 6. Technical Architecture

### 6.1 System Components

```
┌─────────────────────────────────────────────────────────────┐
│                      SNOWFLAKE SERVERLESS                    │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌───────────────┐    ┌───────────────┐    ┌──────────────┐ │
│  │   SDK/CLI     │───▶│  Control Plane │───▶│   SPCS       │ │
│  │               │    │               │    │   (Backend)  │ │
│  │ • Decorators  │    │ • Deploy API  │    │              │ │
│  │ • sf.deploy() │    │ • Image Build │    │ • Services   │ │
│  │ • Endpoint    │    │ • Spec Gen    │    │ • Pools      │ │
│  │   client      │    │ • Pool Mgmt   │    │ • Endpoints  │ │
│  └───────────────┘    └───────────────┘    └──────────────┘ │
│                              │                               │
│                              ▼                               │
│                    ┌───────────────┐                        │
│                    │  Shared Infra  │                        │
│                    │               │                        │
│                    │ • Image Reg   │                        │
│                    │ • Pool Fleet  │                        │
│                    │ • Warm Pools  │                        │
│                    │ • Metrics     │                        │
│                    └───────────────┘                        │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 6.2 Deployment Flow

```
1. Developer calls sf.deploy(my_function)
   │
   ▼
2. SDK analyzes function
   • Parse decorator parameters
   • Detect Python dependencies (imports)
   • Hash code for caching
   │
   ▼
3. Control Plane receives deploy request
   • Validate parameters
   • Check quotas/limits
   │
   ▼
4. Image Builder (if needed)
   • Generate Dockerfile from dependencies
   • Build container image
   • Push to internal registry
   • Cache for future deploys
   │
   ▼
5. ServiceSpec Generator
   • Create ContainerSpec from image + resources
   • Create EndpointSpec for HTTP
   • Add secrets, volumes as needed
   │
   ▼
6. Pool Manager
   • Find existing pool with capacity
   • Or allocate from warm pool fleet
   • Or create new pool (fallback)
   │
   ▼
7. Service Deployer
   • Call SYSTEM$CREATE_SERVICE internally
   • Wait for READY state
   • Handle retries/failures
   │
   ▼
8. Endpoint Manager
   • Generate unique URL
   • Configure Snowflake OAuth
   • Register in service directory
   │
   ▼
9. Return endpoint to developer
   • endpoint.url ready to use
   • Monitoring automatically enabled
```

### 6.3 Warm Pool Fleet

To minimize cold start time, maintain pre-provisioned pools:

```
┌─────────────────────────────────────────────────────────────┐
│                      WARM POOL FLEET                         │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  Tier: small (CPU_X64_XS)     │  Pool Count: 10             │
│  ──────────────────────────   │  Idle Nodes: 15             │
│                               │  Utilization: 60%           │
│                                                              │
│  Tier: medium (CPU_X64_S)     │  Pool Count: 8              │
│  ──────────────────────────   │  Idle Nodes: 12             │
│                               │  Utilization: 70%           │
│                                                              │
│  Tier: gpu-small (GPU_NV_S)   │  Pool Count: 4              │
│  ──────────────────────────   │  Idle Nodes: 4              │
│                               │  Utilization: 50%           │
│                                                              │
└─────────────────────────────────────────────────────────────┘

Auto-scaling rules:
• If utilization > 80%: Add pools
• If utilization < 30%: Remove pools
• Always maintain minimum idle capacity per tier
```

### 6.4 Token Management

Platform handles OAuth token refresh transparently:

```python
# Internal token manager (invisible to developer)
class TokenManager:
    def __init__(self, service_id):
        self.service_id = service_id
        self.token_path = "/snowflake/session/token"
        self.refresh_interval = 50 * 60  # 50 min (before 60 min expiry)

    def start_refresh_loop(self):
        """Background thread refreshes token before expiry."""
        while True:
            time.sleep(self.refresh_interval)
            self._refresh_token()

    def _refresh_token(self):
        """Get new token and update file."""
        new_token = self._request_new_token()
        with open(self.token_path, 'w') as f:
            f.write(new_token)
```

### 6.5 Execution Backend Abstraction Layer

**Critical architectural principle:** The serverless interface provides a stable abstraction that decouples developer code from the underlying execution model. This enables Snowflake to experiment with and migrate between different compute architectures without breaking consumer code.

```
┌─────────────────────────────────────────────────────────────────┐
│                  EXECUTION BACKEND FLEXIBILITY                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Developer Interface (Stable)                                   │
│  ────────────────────────────────                               │
│  @sf.function(tier="medium")                                   │
│  def my_function(data): ...                                     │
│                                                                 │
│                        │                                        │
│                        ▼                                        │
│                                                                 │
│  Serverless Control Plane (Abstraction Layer)                  │
│  ─────────────────────────────────────────────                 │
│  • Parse decorator metadata                                     │
│  • Select optimal execution backend                             │
│  • Generate backend-specific config                             │
│  • Manage lifecycle                                             │
│                                                                 │
│                        │                                        │
│            ┌───────────┼───────────┐                           │
│            ▼           ▼           ▼                           │
│                                                                 │
│  Execution Backends (Pluggable)                                │
│  ───────────────────────────────                               │
│                                                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐            │
│  │  SPCS       │  │  microVMs   │  │ Firecracker │            │
│  │  (Today)    │  │  (Future)   │  │  (Future)   │            │
│  │             │  │             │  │             │            │
│  │ Kubernetes  │  │ Cloud       │  │ Ultra-fast  │            │
│  │ Containers  │  │ Hypervisor  │  │ Isolation   │            │
│  └─────────────┘  └─────────────┘  └─────────────┘            │
│                                                                 │
│  ┌─────────────┐  ┌─────────────┐                             │
│  │ WebAssembly │  │ Bare Metal  │                             │
│  │  (Future)   │  │  (Future)   │                             │
│  │             │  │             │                             │
│  │ Near-native │  │ Max Perf    │                             │
│  │ Speed       │  │ GPU/TPU     │                             │
│  └─────────────┘  └─────────────┘                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

#### Backend Selection Criteria

The control plane automatically selects the optimal backend based on workload characteristics:

| Workload Type | Current Backend | Potential Future Backend | Why Migrate? |
|---------------|----------------|-------------------------|--------------|
| **Lightweight functions** | SPCS containers | microVMs or Firecracker | 10x faster cold start (100ms vs 10s) |
| **GPU workloads** | SPCS GPU pools | Bare metal GPU | Lower overhead, better utilization |
| **Untrusted code** | SPCS containers | WebAssembly sandbox | Stronger isolation, near-native speed |
| **Stateful services** | SPCS persistent | VM-based with local storage | Better state management |
| **High throughput** | SPCS auto-scale | Purpose-built runtime | Optimized for specific workload |

#### Migration Example: SPCS to microVMs

**Scenario:** Platform team discovers that microVMs (e.g., AWS Firecracker, Google gVisor) provide 10x faster cold starts and better isolation than Kubernetes containers.

**Without unified interface:**
```
1. Platform builds microVM backend
2. Every team (Notebooks, Streamlit, Cortex) must:
   - Learn new microVM APIs
   - Rewrite service creation code
   - Migrate existing deployments
   - Update documentation
   - Retrain engineers

Timeline: 6-12 months, high risk
```

**With unified interface:**
```
1. Platform builds microVM backend adapter
2. Control plane routes new deployments to microVMs
3. Gradually migrate existing functions (transparent)
4. Zero code changes for consumer teams

Timeline: 2-4 weeks, low risk
```

#### Implementation Strategy

```python
# Control plane backend router
class ExecutionBackend:
    """Abstract interface for execution backends."""

    def deploy(self, function_spec) -> Endpoint:
        """Deploy function, return endpoint."""
        raise NotImplementedError

    def scale(self, endpoint_id, min_instances, max_instances):
        """Scale function instances."""
        raise NotImplementedError

    def get_logs(self, endpoint_id) -> List[str]:
        """Retrieve function logs."""
        raise NotImplementedError

class SPCSBackend(ExecutionBackend):
    """Current SPCS/Kubernetes backend."""

    def deploy(self, function_spec):
        # Generate ServiceSpec, create compute pool, deploy to SPCS
        service_spec = generate_service_spec(function_spec)
        pool = select_compute_pool(function_spec.tier)
        service_id = create_spcs_service(service_spec, pool)
        return Endpoint(service_id, "spcs")

class MicroVMBackend(ExecutionBackend):
    """Future microVM backend (Firecracker/gVisor)."""

    def deploy(self, function_spec):
        # Generate VM config, provision microVM, deploy
        vm_config = generate_vm_config(function_spec)
        vm_id = provision_microvm(vm_config)
        return Endpoint(vm_id, "microvm")

class WebAssemblyBackend(ExecutionBackend):
    """Future WebAssembly backend for untrusted code."""

    def deploy(self, function_spec):
        # Compile to WASM, deploy to runtime
        wasm_module = compile_to_wasm(function_spec.code)
        module_id = deploy_wasm(wasm_module)
        return Endpoint(module_id, "wasm")

# Control plane selects backend
class ControlPlane:
    def deploy_function(self, function_spec):
        # Select optimal backend based on workload
        backend = self.select_backend(function_spec)
        return backend.deploy(function_spec)

    def select_backend(self, spec):
        if spec.cold_start_sensitive:
            return MicroVMBackend()  # Fast cold starts
        elif spec.untrusted_code:
            return WebAssemblyBackend()  # Strong isolation
        elif spec.gpu_required:
            return SPCSBackend()  # GPU support
        else:
            return SPCSBackend()  # Default
```

#### Benefits of Backend Abstraction

| Benefit | Description | Business Value |
|---------|-------------|----------------|
| **Future-proof** | Platform can adopt new technologies without breaking consumers | Low migration risk |
| **Experimentation** | A/B test different backends on production traffic | Data-driven decisions |
| **Cost optimization** | Route workloads to most cost-effective backend | Lower compute costs |
| **Performance tuning** | Use specialized backends for specific workload types | Better UX |
| **Vendor flexibility** | Not locked into single cloud provider's container tech | Negotiating leverage |

#### Real-World Example: Notebooks Migration

**Today's problem:** If Snowflake decides to replace Kubernetes with a better orchestrator, the Notebooks team must:
- Rewrite `SYSTEM$NOTEBOOKS_VNEXT_CREATE_*` integration
- Update all ServiceSpec generation logic
- Migrate running notebooks
- Test extensively

**With serverless interface:**
```python
# Notebooks team code (unchanged)
@sf.notebook(source="@stage/nb.ipynb", tier="gpu-small")
def user_notebook():
    pass

# Platform team (transparent migration)
# Week 1: Add new backend adapter
# Week 2: Route 10% traffic to new backend
# Week 3: Route 50% traffic
# Week 4: Route 100% traffic, decommission old backend

# Notebooks team: Zero code changes, zero downtime
```

---

## 7. Detailed Requirements

### 7.1 Functional Requirements

#### FR1: Function Deployment
| ID | Requirement | Priority |
|----|-------------|----------|
| FR1.1 | Deploy Python function with `@sf.function` decorator | P0 |
| FR1.2 | Auto-detect and install Python dependencies | P0 |
| FR1.3 | Support CPU and memory specification | P0 |
| FR1.4 | Support GPU specification (A10G, H100) | P0 |
| FR1.5 | Support secrets injection from Snowflake | P0 |
| FR1.6 | Support timeout configuration | P1 |
| FR1.7 | Support environment variables | P1 |

#### FR2: App Deployment (Streamlit)
| ID | Requirement | Priority |
|----|-------------|----------|
| FR2.1 | Deploy Streamlit app with `@sf.app` decorator | P0 |
| FR2.2 | Support concurrent user limit | P0 |
| FR2.3 | Auto-scaling based on user count | P1 |
| FR2.4 | Session affinity for stateful apps | P2 |

#### FR3: Notebook Deployment
| ID | Requirement | Priority |
|----|-------------|----------|
| FR3.1 | Deploy notebook from stage with `sf.notebook()` | P0 |
| FR3.2 | Support GPU for ML notebooks | P0 |
| FR3.3 | Persistent storage for notebook state | P1 |
| FR3.4 | Collaborative editing (multiple users) | P2 |

#### FR4: Job Deployment
| ID | Requirement | Priority |
|----|-------------|----------|
| FR4.1 | Deploy scheduled job with `@sf.job` decorator | P1 |
| FR4.2 | Support cron schedule syntax | P1 |
| FR4.3 | Job history and logs | P1 |
| FR4.4 | Manual job trigger | P1 |

#### FR5: Container Deployment
| ID | Requirement | Priority |
|----|-------------|----------|
| FR5.1 | Deploy from container image with `sf.container()` | P1 |
| FR5.2 | Deploy from Dockerfile | P2 |
| FR5.3 | Support any language (Go, Rust, Java, etc.) | P1 |

#### FR6: Endpoint Management
| ID | Requirement | Priority |
|----|-------------|----------|
| FR6.1 | Generate unique HTTPS endpoint URL | P0 |
| FR6.2 | Snowflake OAuth authentication | P0 |
| FR6.3 | Endpoint status (READY, STARTING, FAILED) | P0 |
| FR6.4 | Container logs access | P0 |
| FR6.5 | Stop/start/delete operations | P0 |
| FR6.6 | Manual scaling (min/max instances) | P1 |

#### FR7: Snowflake Integration
| ID | Requirement | Priority |
|----|-------------|----------|
| FR7.1 | Automatic Snowflake session in functions | P0 |
| FR7.2 | Access to tables, stages, warehouses | P0 |
| FR7.3 | Secrets from Snowflake secret manager | P0 |
| FR7.4 | Inherit caller's role and permissions | P0 |

### 7.2 Non-Functional Requirements

#### NFR1: Performance
| ID | Requirement | Target |
|----|-------------|--------|
| NFR1.1 | Cold start time (warm pool) | < 10 seconds |
| NFR1.2 | Cold start time (new pool) | < 60 seconds |
| NFR1.3 | Deploy time (cached image) | < 30 seconds |
| NFR1.4 | Deploy time (new image) | < 2 minutes |
| NFR1.5 | Endpoint latency overhead | < 50ms |

#### NFR2: Reliability
| ID | Requirement | Target |
|----|-------------|--------|
| NFR2.1 | Service availability | 99.9% |
| NFR2.2 | Deployment success rate | 99% |
| NFR2.3 | Auto-recovery from failures | < 30 seconds |

#### NFR3: Scalability
| ID | Requirement | Target |
|----|-------------|--------|
| NFR3.1 | Max concurrent functions per account | 1000 |
| NFR3.2 | Max instances per function | 100 |
| NFR3.3 | Auto-scale response time | < 30 seconds |

#### NFR4: Security
| ID | Requirement | Priority |
|----|-------------|----------|
| NFR4.1 | All endpoints require Snowflake auth | P0 |
| NFR4.2 | Network isolation between functions | P0 |
| NFR4.3 | Secrets encrypted at rest and in transit | P0 |
| NFR4.4 | Audit logging for all operations | P0 |

---

## 8. User Stories

### 8.1 Notebooks Team

**US1: Deploy Notebook Runtime**
```
As a Notebooks vNext developer,
I want to deploy a notebook runtime with GPU support,
So that users can run ML workloads without infrastructure setup.

Acceptance Criteria:
- Deploy with sf.notebook(source="@stage/nb.ipynb", tier="gpu-small")
- Get working URL in < 1 minute
- GPU is available in notebook environment
- Snowflake session works automatically
```

### 8.2 Streamlit Team

**US2: Deploy Streamlit App**
```
As a Streamlit developer,
I want to deploy an app that scales with user count,
So that I don't have to manage compute pools manually.

Acceptance Criteria:
- Deploy with @sf.app(concurrent_users=50)
- App auto-scales from 1 to N instances
- Cold start < 10 seconds for new users
- Token refresh is automatic (no session timeouts)
```

### 8.3 Cortex Team

**US3: Deploy ML Inference Endpoint**
```
As a Cortex developer,
I want to deploy an ML model as a serverless endpoint,
So that I can serve predictions without managing infrastructure.

Acceptance Criteria:
- Deploy with @sf.function(tier="gpu-small")
- Endpoint handles concurrent requests
- Auto-scales based on load
- Cost tracked per invocation
```

### 8.4 General

**US4: Deploy Any Container**
```
As a developer using Go/Rust/Java,
I want to deploy my container without learning SPCS,
So that I can use my preferred language.

Acceptance Criteria:
- Deploy with sf.container(image="my-go:latest", port=8080)
- Works with any HTTP server
- Same endpoint/scaling features as Python
```

---

## 9. Rollout Plan

### 9.1 Phase 1: Internal Alpha (Month 1-2)

**Scope:**
- `@sf.function` decorator (Python only)
- Basic tiers (small, medium, large)
- Manual deploy via SDK
- Single region (AWS us-west-2)

**Users:**
- SPCS team (dogfooding)
- 2-3 volunteer teams

**Success Criteria:**
- 10+ functions deployed
- < 2 minute deploy time
- No major bugs

### 9.2 Phase 2: Internal Beta (Month 3-4)

**Scope:**
- `@sf.app` for Streamlit
- `sf.notebook()` for notebooks
- GPU tiers (gpu-small, gpu-large)
- Auto-scaling
- Multi-region

**Users:**
- Notebooks vNext team
- Streamlit team
- Cortex team

**Success Criteria:**
- 3+ teams actively using
- 50% reduction in SPCS boilerplate
- P0 features working reliably

### 9.3 Phase 3: Internal GA (Month 5-6)

**Scope:**
- `@sf.job` for scheduled jobs
- `sf.container()` for any language
- Cost attribution dashboard
- Full monitoring/alerting

**Users:**
- All internal teams

**Success Criteria:**
- 80% of new deployments use serverless
- < 1 minute average deploy time
- 90% developer satisfaction

### 9.4 Phase 4: Customer Preview (Month 7+)

**Scope:**
- Public documentation
- Snowsight integration
- Billing integration
- Support playbooks

**Users:**
- Select customer beta

### 9.5 Phase 5: Alternative Execution Backends (Month 9-12)

**Scope:**
- Execution backend abstraction layer
- microVM backend (Firecracker or gVisor)
- A/B testing framework for backend selection
- Automatic backend migration for improved cold starts

**Backend Architecture:**
- Pluggable backend interface
- SPCS remains default for all workloads
- microVMs opt-in for cold-start-sensitive workloads
- Transparent migration with automatic fallback

**Success Criteria:**
- 50% reduction in cold start times for eligible workloads
- Zero code changes required for backend migration
- < 1% error rate during backend transitions
- Cost-neutral or cost-positive (microVMs may be cheaper)

**Key Milestone:**
This phase proves the value of the unified interface - teams can benefit from next-generation execution models without rewriting any code.

---

## 10. Open Questions & Options

### Q1: Should we build on existing SPCS or create new backend?

| Option | Description | Pros | Cons |
|--------|-------------|------|------|
| **A: Layer on SPCS** | Serverless SDK generates ServiceSpec, uses existing SPCS APIs | Faster to build, leverages existing infra, lower risk | Inherits SPCS limitations, may hit performance ceiling |
| **B: New Backend** | Purpose-built serverless runtime, bypass SPCS | Optimized for use case, no legacy constraints | 6+ months longer, duplicate infrastructure, higher risk |
| **C: Hybrid** | New control plane, reuse SPCS compute/networking | Best of both, can migrate incrementally | Complexity of two systems, unclear ownership |
| **D: Abstraction Layer** | Build pluggable backend interface, start with SPCS, add alternatives over time | Future-proof, enables experimentation, incremental investment | Upfront abstraction overhead |

**Updated Context:** The unified interface should include a **backend abstraction layer** from day one. This allows:
- V1: Launch with SPCS backend (proven, low-risk)
- V2: Add microVM backend for faster cold starts
- V3+: Add WebAssembly, bare metal GPU, or other specialized backends

**Recommendation:** _Option D (Abstraction Layer) - Start with SPCS as the default backend, but architect the control plane to support multiple backends. This provides immediate value (simplified SPCS interface) while enabling future performance improvements (microVMs, etc.) without breaking consumer code._

---

### Q2: How to handle image caching across regions?

| Option | Description | Pros | Cons |
|--------|-------------|------|------|
| **A: Global Registry** | Single registry, replicate to all regions | Simple model, consistent | Cross-region latency, replication costs |
| **B: Regional Registries** | Registry per region, sync on-demand | Fast local pulls, cost efficient | Complexity, cold start on first regional deploy |
| **C: Edge Caching** | CDN-style caching at edge locations | Fastest pulls, automatic | Infrastructure cost, cache invalidation complexity |
| **D: Customer-owned** | Customers use their own registries | No storage cost for us | Poor UX, security concerns, slower deploys |

**Recommendation:** _TBD - needs cost analysis_

---

### Q3: What's the billing model?

| Option | Description | Pros | Cons |
|--------|-------------|------|------|
| **A: Per-invocation** | Charge per function call (like Lambda) | Pay for what you use, simple to understand | Unpredictable costs, gaming concerns |
| **B: Per-hour (compute time)** | Charge for actual compute seconds | Aligns with cost, familiar model | Complex metering, idle time debates |
| **C: Per-hour (wall clock)** | Charge while endpoint exists | Predictable, simple billing | Customers pay for idle, less competitive |
| **D: Tiered subscription** | Monthly tiers with included compute | Predictable revenue, simple | May not fit all use cases |
| **E: Hybrid** | Base fee + per-invocation overage | Predictable base, scales with usage | Complex to explain |

**Pricing Comparison:**
| Provider | Model | Example: 1M invocations/month, 100ms avg |
|----------|-------|------------------------------------------|
| AWS Lambda | Per-invocation + compute | ~$20/month |
| Modal.ai | Per-second compute | ~$30/month |
| Google Cloud Run | Per-request + compute | ~$25/month |
| Snowflake (current SPCS) | Per-hour pool | ~$200/month (always-on) |

**Recommendation:** _TBD - needs competitive analysis_

---

### Q4: How to handle VPC/PrivateLink for enterprise customers?

| Option | Description | Pros | Cons |
|--------|-------------|------|------|
| **A: Public endpoints only (V1)** | All endpoints are public HTTPS | Simple, fast to market | Blocks enterprise customers |
| **B: PrivateLink per function** | Each function gets PrivateLink endpoint | Maximum isolation | Expensive, slow provisioning |
| **C: Shared PrivateLink gateway** | One PrivateLink per account, routes to functions | Cost efficient, enterprise ready | Routing complexity |
| **D: VPC peering** | Direct VPC connection to Snowflake | Best performance | Complex setup, not self-service |

**Recommendation:** _TBD - needs security review_

---

### Q5: Should functions share pools or have dedicated pools?

| Option | Description | Pros | Cons |
|--------|-------------|------|------|
| **A: Shared pools (multi-tenant)** | All functions share pool fleet | Cost efficient, fast cold start | Noisy neighbor, security isolation |
| **B: Dedicated pools (per-account)** | Each account gets own pools | Full isolation, predictable | Expensive, slow cold start |
| **C: Dedicated pools (per-function)** | Each function gets own pool | Maximum isolation | Very expensive, wasteful |
| **D: Hybrid** | Shared by default, dedicated on request | Flexibility | Complexity, two code paths |
| **E: Tiered isolation** | Free=shared, Pro=account-dedicated, Enterprise=function-dedicated | Aligns with pricing | Complex operations |

**Isolation Considerations:**
- CPU/memory: Containers provide isolation
- Network: Need network policies
- Storage: Separate volumes required
- Secrets: Must not leak between tenants

**Recommendation:** _TBD - needs security review_

---

### Q6: How to handle large dependencies (ML libraries)?

| Option | Description | Pros | Cons |
|--------|-------------|------|------|
| **A: Build from scratch** | Install all deps at build time | Clean, reproducible | Slow builds (10+ min for PyTorch) |
| **B: Curated base images** | Pre-built images with common ML libs | Fast builds, optimized | Limited flexibility, maintenance burden |
| **C: Layer caching** | Cache dependency layers, rebuild only app code | Fast incremental builds | Storage costs, cache invalidation |
| **D: Conda environments** | Use existing Snowflake conda integration | Familiar to users | Limited packages, version conflicts |
| **E: Bring your own image** | Customers provide complete image | Full flexibility | Poor UX, security scanning needed |

**Common ML Dependencies:**
| Library | Size | Install Time |
|---------|------|--------------|
| PyTorch | 2+ GB | 3-5 min |
| TensorFlow | 1.5 GB | 2-4 min |
| Transformers | 500 MB | 1-2 min |
| scikit-learn | 100 MB | 30 sec |

**Recommendation:** _TBD - needs benchmarking_

---

### Q7: What's the migration path for existing SPCS services?

| Option | Description | Pros | Cons |
|--------|-------------|------|------|
| **A: No migration** | Existing SPCS stays, serverless is new | No disruption | Two systems forever |
| **B: Manual migration** | Documentation + tools, customers migrate | Low effort for us | Slow adoption, support burden |
| **C: Automated migration** | Tool converts ServiceSpec to decorators | Easy for customers | Edge cases, may not cover all |
| **D: Gradual deprecation** | Serverless is default, SPCS deprecated over 2 years | Clean end state | Forcing customers to change |
| **E: Compatibility layer** | Serverless can run existing ServiceSpecs | No code changes | Complexity, maintaining two APIs |

**Migration Complexity by Service Type:**
| Service Type | Migration Difficulty | Notes |
|--------------|---------------------|-------|
| Simple HTTP endpoint | Easy | Direct mapping to @sf.function |
| Streamlit app | Easy | Direct mapping to @sf.app |
| Multi-container | Hard | Need to split or use sf.container |
| Custom networking | Very Hard | May need manual intervention |
| Stateful services | Very Hard | Need storage migration |

**Recommendation:** _TBD - needs customer research_

---

### Q8: How to expose this to external customers?

| Option | Description | Pros | Cons |
|--------|-------------|------|------|
| **A: Internal only** | Only for Snowflake internal teams | Lower risk, focused scope | Limited impact |
| **B: Native Apps only** | Available to Native App developers | Clear use case, controlled | Limited market |
| **C: All customers (Python SDK)** | pip install snowflake-serverless | Maximum reach | Support burden, abuse risk |
| **D: Snowsight only** | GUI-based deployment, no SDK | Easy to use, controlled | Limited power users |
| **E: Phased rollout** | Internal → Native Apps → All customers | Controlled expansion | Slower time to market |

**Customer Segments:**
| Segment | Need | Fit |
|---------|------|-----|
| Data Scientists | Deploy ML models | High |
| App Developers | Internal tools | High |
| Data Engineers | Batch jobs | Medium |
| Analysts | Dashboards | Medium |
| External Apps | Public APIs | Low (security) |

**Recommendation:** _TBD - needs PM decision_

---

### Q9: What interface style should we prioritize?

| Option | Description | Pros | Cons |
|--------|-------------|------|------|
| **A: Python SDK only** | Decorators + sf.deploy() | Pythonic, Modal-like | Excludes non-Python users |
| **B: SQL DDL only** | CREATE SERVERLESS FUNCTION... | Snowflake-native | Verbose, not code-first |
| **C: CLI only** | snow serverless deploy | DevOps friendly | Learning curve |
| **D: GUI only (Snowsight)** | Point-and-click deployment | Easy onboarding | Limited for power users |
| **E: All of the above** | SDK, SQL, CLI, GUI | Maximum flexibility | 4x development effort |
| **F: SDK + SQL** | Python SDK primary, SQL for admin | Best of both | Still 2x effort |

**User Preference by Persona:**
| Persona | Preferred Interface |
|---------|-------------------|
| Data Scientist | Python SDK |
| ML Engineer | Python SDK + CLI |
| Data Engineer | SQL + CLI |
| App Developer | Python SDK |
| DBA | SQL |
| Business Analyst | GUI |

**Recommendation:** _TBD - needs user research_

---

### Q10: How should auto-scaling work?

| Option | Description | Pros | Cons |
|--------|-------------|------|------|
| **A: Request-based** | Scale based on pending requests | Responsive, matches load | May over-provision |
| **B: CPU/Memory-based** | Scale when resources exhausted | Efficient | Slower response to spikes |
| **C: Predictive** | ML-based prediction of load | Proactive scaling | Complex, may mispredict |
| **D: Manual only** | User sets min/max, no auto | Predictable costs | Poor UX, over/under provision |
| **E: Schedule-based** | Scale based on time of day | Predictable | Doesn't handle unexpected load |
| **F: Hybrid** | Request-based + schedule hints | Flexible | Complex configuration |

**Scaling Parameters:**
```python
@sf.function(
    tier="medium",
    # Scaling options
    min_instances=1,      # Always running (warm)
    max_instances=10,     # Cap for cost control
    scale_to_zero=True,   # Allow 0 instances when idle
    target_concurrency=10, # Requests per instance before scaling
    scale_up_delay=5,     # Seconds before adding instance
    scale_down_delay=300, # Seconds before removing instance
)
```

**Recommendation:** _TBD - needs benchmarking_

---

### Q11: How to handle secrets and credentials?

| Option | Description | Pros | Cons |
|--------|-------------|------|------|
| **A: Snowflake Secrets only** | Use existing secret manager | Integrated, secure | Limited to Snowflake secrets |
| **B: Environment variables** | Pass secrets as env vars | Simple, standard | Less secure, visible in config |
| **C: External vaults** | HashiCorp Vault, AWS Secrets Manager | Enterprise grade | Complexity, external dependency |
| **D: Encrypted config** | Encrypted secrets in code/config | Portable | Key management burden |
| **E: Hybrid** | Snowflake Secrets + external vaults | Flexibility | Complex |

**Secret Injection API:**
```python
# Option A: Snowflake Secrets
@sf.function(secrets=["my_api_key", "db_password"])
def my_function():
    key = os.environ["MY_API_KEY"]  # Injected from Snowflake

# Option B: Inline (less secure)
@sf.function(env={"API_KEY": "sk-xxx"})
def my_function():
    key = os.environ["API_KEY"]

# Option C: External vault
@sf.function(secrets_from="hashicorp://vault/path")
def my_function():
    key = os.environ["API_KEY"]
```

**Recommendation:** _TBD - needs security review_

---

### Q12: What monitoring and observability features are needed?

| Feature | Priority | Options |
|---------|----------|---------|
| **Logs** | P0 | Container stdout/stderr, structured JSON |
| **Metrics** | P0 | Invocations, latency, errors, cost |
| **Traces** | P1 | OpenTelemetry, Snowflake native |
| **Alerts** | P1 | Error rate, latency, cost thresholds |
| **Dashboards** | P2 | Snowsight, Grafana, DataDog |
| **Profiling** | P2 | CPU, memory, GPU utilization |

**Observability API:**
```python
# Get logs
logs = endpoint.logs(since="1h", level="ERROR")

# Get metrics
metrics = endpoint.metrics(
    period="1d",
    metrics=["invocations", "p99_latency", "error_rate", "cost"]
)

# Set alerts
endpoint.alert(
    name="high_error_rate",
    condition="error_rate > 0.05",
    notify=["slack://channel", "email://team@snowflake.com"]
)
```

**Recommendation:** _TBD - needs platform review_

---

### Q13: What execution backends should we support, and when?

| Backend | Cold Start | Isolation | Maturity | Use Case | Timeline |
|---------|------------|-----------|----------|----------|----------|
| **SPCS (Kubernetes)** | 10-60s | Strong (containers) | Production | Default, all workloads | V1 (today) |
| **microVMs (Firecracker/gVisor)** | 100-500ms | Very strong (VM-level) | Production-ready | Cold start sensitive functions | V2 (6-12 mo) |
| **WebAssembly (wasmtime)** | <100ms | Extremely strong (sandbox) | Emerging | Untrusted code, edge compute | V3 (12-18 mo) |
| **Bare Metal GPU** | Instant (persistent) | Moderate | Production | Large GPU workloads | V2 (6-12 mo) |
| **Serverless Containers (Cloud Run style)** | 1-5s | Strong | Production | Alternative to SPCS | Investigate |

**Key Decision: Backend Selection Strategy**

| Approach | Description | Pros | Cons |
|----------|-------------|------|------|
| **A: Manual selection** | Developer specifies backend in decorator | Explicit control | Complexity, wrong choices |
| **B: Automatic (heuristic)** | Platform chooses based on tier/code analysis | Simple UX | May be suboptimal |
| **C: Hybrid** | Auto by default, manual override | Best of both | Extra API surface |
| **D: Progressive rollout** | Start with SPCS, add backends over time | Lower risk | Delayed benefits |

**Migration Scenarios:**

```python
# Scenario 1: Transparent (recommended)
@sf.function(tier="medium")
def my_function(data):
    return process(data)

# Platform migrates from SPCS → microVMs automatically
# Developer sees no change, just faster cold starts

# Scenario 2: Explicit backend selection
@sf.function(tier="medium", backend="microvm")  # Opt-in to new backend
def my_function(data):
    return process(data)

# Scenario 3: A/B testing
@sf.function(tier="medium", experimental=True)
def my_function(data):
    return process(data)

# Platform routes 10% to microVMs, 90% to SPCS
# Collects metrics, gradually shifts traffic
```

**Backend Compatibility Matrix:**

| Feature | SPCS | microVMs | WebAssembly | Bare Metal |
|---------|------|----------|-------------|------------|
| Python support | ✅ | ✅ | Limited | ✅ |
| GPU support | ✅ | Limited | ❌ | ✅ |
| Multi-container | ✅ | ❌ | ❌ | ✅ |
| Persistent storage | ✅ | Limited | ❌ | ✅ |
| Network access | ✅ | ✅ | Restricted | ✅ |
| Cold start | Slow | Fast | Ultra-fast | N/A (persistent) |
| Max memory | 256GB | 32GB | 4GB | 1TB+ |

**Example Use Cases by Backend:**

| Use Case | Recommended Backend | Why |
|----------|-------------------|-----|
| Jupyter notebook (long session) | SPCS or Bare Metal | Need persistent state, GPU |
| Streamlit app (bursty traffic) | microVMs | Fast cold start for new users |
| ML inference (high throughput) | Bare Metal GPU | Minimize latency, maximize GPU utilization |
| Data transformation (lightweight) | microVMs or WebAssembly | Fast cold start, efficient |
| Batch job (hours-long) | SPCS or Bare Metal | Stable runtime, no cold start concern |
| User-provided code (untrusted) | WebAssembly | Strongest isolation |

**Recommendation:** _TBD - needs architecture review and benchmarking_

**Key Questions to Answer:**
1. What's the investment timeline for each backend?
2. What workload percentage benefits from microVMs vs SPCS?
3. Do we build backend adapters or fork to specialized runtimes?
4. How do we handle backend-specific failures (fallback to SPCS)?
5. Can we A/B test backends in production safely?

---

## 11. Risks & Mitigations

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Cold start too slow | Medium | High | Warm pool fleet, image caching, migrate to microVMs |
| Image build failures | Medium | Medium | Curated base images, clear errors |
| Cost overruns (pools) | Medium | Medium | Aggressive auto-suspend, quotas |
| Security vulnerabilities | Low | High | Network isolation, code scanning |
| Adoption resistance | Low | Medium | Migration tools, documentation |
| **Backend migration failures** | **Medium** | **High** | **Thorough testing, gradual rollout, automatic fallback to SPCS** |
| **Wrong backend selection** | **Medium** | **Medium** | **Data-driven heuristics, manual override option, continuous profiling** |

---

## 12. Dependencies

| Dependency | Team | Status |
|------------|------|--------|
| SPCS service creation APIs | SPCS | Available |
| Compute pool management | SPCS | Available |
| Image registry | Platform | Available |
| Snowflake secrets | Security | Available |
| OAuth token refresh | Auth | Needs enhancement |
| Cost attribution APIs | Billing | Needs development |

---

## 13. OpenFlow Integration

### 13.1 Overview

OpenFlow is Snowflake's workflow orchestration framework for coordinating multi-step data pipelines and ML workflows. The serverless platform must integrate seamlessly with OpenFlow to enable declarative workflow definitions without infrastructure complexity.

### 13.2 Current OpenFlow + SPCS Pattern

**Today's approach (complex):**
```yaml
# OpenFlow workflow definition
tasks:
  - id: preprocess_data
    type: spcs_job
    config:
      compute_pool: MY_POOL           # Must pre-create
      image: data-prep:v1             # Must build/push
      command: ["python", "prep.py"]
      # ... 10+ more config lines
```

**Developer must manage:**
- Compute pool creation and sizing
- Container image building and versioning
- ServiceSpec configuration
- Resource estimation

### 13.3 Proposed Serverless + OpenFlow Integration

**New approach (simple):**
```python
import snowflake.serverless as sf

# Define function with decorator
@sf.function(tier="medium")
def preprocess_data(batch):
    return transform(batch)

# OpenFlow workflow references function
workflow = sf.workflow("ml_pipeline")
workflow.add_task("preprocess", preprocess_data)
workflow.add_task("train", train_model, depends_on=["preprocess"])
workflow.deploy()
```

**Platform handles:**
- Automatic function deployment
- Resource allocation
- Image building
- State management

### 13.4 Integration Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│              OPENFLOW + SERVERLESS INTEGRATION                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Developer Code                                                 │
│       │                                                         │
│       ├─▶ @sf.function() decorators                            │
│       └─▶ workflow.add_task()                                  │
│                                                                 │
│                        │                                        │
│                        ▼                                        │
│                                                                 │
│  Serverless Platform                                           │
│       │                                                         │
│       ├─▶ sf.deploy() → creates endpoints                      │
│       └─▶ Returns function IDs                                 │
│                                                                 │
│                        │                                        │
│                        ▼                                        │
│                                                                 │
│  OpenFlow Orchestrator                                         │
│       │                                                         │
│       ├─▶ Schedules task execution                             │
│       ├─▶ Manages dependencies                                 │
│       ├─▶ Handles retries/failures                             │
│       └─▶ Tracks state                                         │
│                                                                 │
│                        │                                        │
│                        ▼                                        │
│                                                                 │
│  SPCS Backend (invisible)                                      │
│       │                                                         │
│       ├─▶ Compute pools                                        │
│       ├─▶ Container execution                                  │
│       └─▶ Resource management                                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 13.5 API Design for Workflows

```python
import snowflake.serverless as sf

# Define workflow tasks as functions
@sf.function(tier="medium")
def extract_features(input_path: str) -> str:
    """Extract features from raw data."""
    # Process data
    return output_path

@sf.function(tier="gpu-large")
def train_model(features_path: str) -> str:
    """Train ML model on features."""
    # Train model
    return model_path

@sf.function(tier="medium")
def evaluate_model(model_path: str) -> dict:
    """Evaluate model performance."""
    # Run evaluation
    return metrics

# Create workflow
workflow = sf.Workflow("ml_training_pipeline")

# Add tasks with dependencies
workflow.add_task(
    "extract",
    extract_features,
    args={"input_path": "@raw_data/inputs"}
)

workflow.add_task(
    "train",
    train_model,
    depends_on=["extract"],
    retry_policy={"max_attempts": 3, "backoff_seconds": [60, 300, 900]}
)

workflow.add_task(
    "evaluate",
    evaluate_model,
    depends_on=["train"]
)

# Deploy workflow (creates all functions + orchestration)
workflow.deploy()

# Trigger workflow execution
run = workflow.run()
print(f"Workflow run ID: {run.id}")

# Monitor progress
status = run.status()  # RUNNING, SUCCEEDED, FAILED
logs = run.logs()
```

### 13.6 Advanced Workflow Patterns

#### Pattern 1: Parallel Fan-Out
```python
@sf.function(tier="medium")
def process_partition(partition_id: int):
    return process(partition_id)

workflow = sf.Workflow("parallel_processing")

# Process 100 partitions in parallel
for i in range(100):
    workflow.add_task(f"partition_{i}", process_partition, args={"partition_id": i})

# Merge results
workflow.add_task("merge", merge_results, depends_on=[f"partition_{i}" for i in range(100)])
```

#### Pattern 2: Conditional Branching
```python
workflow = sf.Workflow("conditional_pipeline")

workflow.add_task("check_quality", check_data_quality)

# Only run cleaning if quality is poor
workflow.add_task(
    "clean_data",
    clean_data,
    depends_on=["check_quality"],
    condition="check_quality.result.score < 0.95"
)

# Training runs either way
workflow.add_task(
    "train",
    train_model,
    depends_on=["check_quality", "clean_data"],
    condition="check_quality.result.score >= 0.95 OR clean_data.status == 'SUCCEEDED'"
)
```

#### Pattern 3: Scheduled Workflows
```python
workflow = sf.Workflow("nightly_pipeline")

# Schedule: daily at midnight
workflow.schedule("0 0 * * *")

workflow.add_task("extract", extract_data)
workflow.add_task("transform", transform_data, depends_on=["extract"])
workflow.add_task("load", load_data, depends_on=["transform"])

workflow.deploy()
```

### 13.7 Data Passing Between Tasks

```python
# Pattern 1: Return values (automatic stage storage)
@sf.function(tier="medium")
def task_a():
    return {"status": "success", "count": 1000}  # Auto-stored in stage

@sf.function(tier="medium")
def task_b(input_data: dict):
    print(input_data["count"])  # Receives output from task_a

workflow.add_task("a", task_a)
workflow.add_task("b", task_b, inputs={"input_data": "a.result"})  # Reference previous task

# Pattern 2: Explicit stage paths
@sf.function(tier="medium")
def task_c():
    df.to_parquet("@pipeline_stage/output.parquet")
    return "@pipeline_stage/output.parquet"

@sf.function(tier="medium")
def task_d(input_path: str):
    df = pd.read_parquet(input_path)
```

### 13.8 Monitoring and Observability

```python
# Get workflow status
run = workflow.get_run(run_id)

# Task-level metrics
for task in run.tasks:
    print(f"{task.name}: {task.status}")
    print(f"  Duration: {task.duration_seconds}s")
    print(f"  Cost: ${task.cost}")
    print(f"  Logs: {task.logs()}")

# Workflow-level metrics
print(f"Total duration: {run.duration_seconds}s")
print(f"Total cost: ${run.cost}")
print(f"Task success rate: {run.success_rate}%")

# Set up alerts
workflow.alert(
    name="failure_alert",
    condition="status == 'FAILED'",
    notify=["slack://data-team", "email://oncall@company.com"]
)
```

### 13.9 Integration with Existing OpenFlow Workflows

**Migration path for existing OpenFlow workflows:**

1. **Phase 1: Hybrid Mode** (backward compatible)
```yaml
# Existing SPCS tasks continue to work
tasks:
  - id: legacy_task
    type: spcs_job
    config:
      compute_pool: MY_POOL
      image: legacy:v1

  # New serverless tasks coexist
  - id: new_task
    type: serverless_function
    function_ref: "my_function"  # References sf.function()
```

2. **Phase 2: Full Migration**
```python
# Convert entire workflow to serverless
workflow = sf.Workflow.from_yaml("legacy_workflow.yaml")
workflow.convert_to_serverless()  # Automatic migration
workflow.deploy()
```

### 13.10 Benefits of Serverless + OpenFlow

| Capability | Current (OpenFlow + SPCS) | With Serverless | Improvement |
|------------|--------------------------|-----------------|-------------|
| **Task Definition** | 20+ lines YAML | 5 lines Python | 75% reduction |
| **Compute Management** | Manual pool creation | Automatic | Zero overhead |
| **Image Management** | Manual build/push | Automatic | Zero overhead |
| **Cold Start** | 1-5 minutes | < 30 seconds (SPCS) → < 1s (microVMs) | 80-99% faster |
| **Resource Tuning** | Manual trial-and-error | Auto-optimization | Intelligent |
| **Cost Tracking** | Job-level only | Function + workflow level | Granular |
| **Developer Friction** | High | Low | Smooth |
| **Backend Migration** | Weeks of rewriting | Transparent (zero code changes) | **Future-proof** |

### 13.11 Requirements for Integration

#### FR13: OpenFlow Integration
| ID | Requirement | Priority |
|----|-------------|----------|
| FR13.1 | Support `sf.Workflow()` API for workflow definition | P0 |
| FR13.2 | Automatic function deployment when workflow is deployed | P0 |
| FR13.3 | Task dependency management (depends_on) | P0 |
| FR13.4 | Pass data between tasks via return values | P0 |
| FR13.5 | Workflow scheduling (cron expressions) | P1 |
| FR13.6 | Conditional task execution | P1 |
| FR13.7 | Retry policies per task | P1 |
| FR13.8 | Workflow-level monitoring and alerts | P1 |
| FR13.9 | Parallel task execution (fan-out/fan-in) | P1 |
| FR13.10 | Migration tool from YAML workflows to serverless | P2 |
| FR13.11 | Hybrid mode (mix SPCS + serverless tasks) | P2 |

### 13.12 Implementation Notes

**Backend Integration Points:**

1. **Function Registry**: Serverless functions register with OpenFlow's task registry
2. **State Management**: OpenFlow tracks serverless function execution state
3. **Resource Allocation**: Serverless platform communicates available capacity to OpenFlow
4. **Failure Handling**: OpenFlow's retry logic works transparently with serverless functions
5. **Data Lineage**: Track data flow through workflow stages automatically

**Technical Considerations:**

- Serverless functions must support long-running tasks (hours) for ML training
- Checkpointing for resumable execution after failures
- Output caching to avoid re-running expensive tasks
- Cross-workflow function sharing (same function used in multiple workflows)

---

## 14. Appendix

### A. Current vs Proposed Comparison

**Current (50+ lines):**
```python
# 1. Create compute pool
# CREATE COMPUTE POOL my_pool INSTANCE_FAMILY=CPU_X64_S...

# 2. Build image
# docker build && docker push

# 3. Write ServiceSpec
service_spec = ServiceSpec()
service_spec.add_container(ContainerSpec(
    name="main",
    image="registry/my-image:latest",
    resources=ResourceSpec(
        requests={"memory": "4Gi", "cpu": "2"},
        limits={"memory": "4Gi", "cpu": "2"}
    ),
    # ... 30 more lines
))
service_spec.add_endpoint(EndpointSpec(...))
service_spec.add_volume(VolumeSpec(...))

# 4. Create service
cursor.execute(f"CREATE SERVICE ... FROM SPECIFICATION $$...$$")

# 5. Wait for ready
# 6. Get endpoint URL
```

**Proposed (5 lines):**
```python
import snowflake.serverless as sf

@sf.function(cpu=2, memory="4GB")
def my_function(data):
    return process(data)

endpoint = sf.deploy(my_function)
```

### B. Glossary

| Term | Definition |
|------|------------|
| SPCS | Snowpark Container Services - Snowflake's container orchestration platform (Kubernetes-based) |
| ServiceSpec | YAML specification for SPCS services |
| Compute Pool | Group of compute nodes for running containers |
| Warm Pool | Pre-provisioned pool for fast cold starts |
| Tier | Predefined resource configuration (small/medium/large) |
| OpenFlow | Snowflake's workflow orchestration framework for multi-step pipelines |
| DAG | Directed Acyclic Graph - workflow structure with task dependencies |
| Execution Backend | The underlying compute infrastructure (SPCS, microVMs, WebAssembly, etc.) |
| Backend Abstraction | Layer that decouples developer API from execution infrastructure |
| microVM | Lightweight virtual machine (e.g., Firecracker, gVisor) with fast cold start |
| Cold Start | Time to start a new function instance from zero |

---

*Document Version: 1.2*
*Created: February 2026*
*Updated: February 2026 - Added OpenFlow integration (v1.1) and execution backend abstraction (v1.2)*
