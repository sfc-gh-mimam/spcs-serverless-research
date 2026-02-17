# SPCS Integration Research: How Internal Snowflake Services Access SnowServices

## Executive Summary

This document analyzes how internal Snowflake services (Notebooks vNext, Streamlit/SiS vNext, Cortex Code Agent, and Snowflake Intelligence Agent) integrate with Snowpark Container Services (SPCS). The goal is to inform the design of a **Modal.ai-style serverless interface** that lets developers write code in any language and get endpoints automatically—with the platform handling all infrastructure behind the scenes.

---

## 1. Current Integration Patterns Overview

### 1.1 All Products Using SPCS/SnowServices

| Product | Integration Type | Complexity | Key Interface |
|---------|-----------------|------------|---------------|
| **Notebooks vNext** | Direct SPCS (ServiceSpec) | High | SYSTEM$NOTEBOOKS_VNEXT_CREATE_* |
| **Streamlit/SiS vNext** | Managed SPCS | High | SYSTEM$STREAMLIT_BOOTSTRAP |
| **ML Model Serving** | Direct SPCS | High | SYSTEM$DEPLOY_MODEL |
| **ML Training Jobs** | SPCS Jobs | High | SYSTEM$EXECUTE_ML_JOB |
| **Cortex Search** | Managed Service | Medium | CREATE CORTEX SEARCH SERVICE |
| **Cortex Agent** | Versioned Stage + gRPC | Medium | CREATE AGENT |
| **Intelligence Agent** | Versioned Stage + Mapping | Medium | CREATE SNOWFLAKE INTELLIGENCE |
| **Native Apps** | SPCS for App Services | High | CREATE SERVICE in App |
| **Spark Applications** | SPCS Clusters | High | Spark-on-SPCS module |
| **OpenFlow** | SPCS Integration | Medium | OpenFlow-SPCS API |
| **Data Sharing Apps** | SPCS Services | Medium | Service in shared context |

### 1.2 Product Categories

```
┌─────────────────────────────────────────────────────────────────┐
│                    SPCS CONSUMER PRODUCTS                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  INTERACTIVE COMPUTE          │  AI/ML SERVICES                │
│  ─────────────────────        │  ──────────────                │
│  • Notebooks vNext            │  • ML Model Serving            │
│  • Streamlit/SiS vNext        │  • ML Training Jobs            │
│  • Spark Applications         │  • Cortex Search               │
│                               │  • Cortex Agent                │
│                               │  • Intelligence Agent          │
│                               │                                │
│  PLATFORM SERVICES            │  EXTENSIBILITY                 │
│  ─────────────────            │  ─────────────                 │
│  • Native Apps (NASPCS)       │  • OpenFlow                    │
│  • Data Sharing Apps          │  • Custom Services             │
│                               │                                │
└─────────────────────────────────────────────────────────────────┘
```

### 1.2 The Problem

Teams currently must understand and manage:
- ServiceSpec YAML (containers, endpoints, volumes, secrets)
- Compute pool creation, sizing, and lifecycle
- Container image building, registry, versioning
- Token refresh logic (60-minute OAuth tokens)
- Resource estimation (CPU, memory, GPU)

**This is too much infrastructure burden for teams who just want to run code.**

---

## 2. Notebooks vNext Integration

### 2.1 Service Creation Flow

**File:** `/Users/mimam/snowflake-research/snowflake-main/Snowfort/tests/notebooks_vnext/api/notebook_service_api.py`

```python
class NotebookServiceAPI:
    def create(self, schema_name, service_name, compute_pool_name, service_spec, is_interactive=True):
        if is_interactive:
            return self.hybrid_executor.execute_query(
                f"select SYSTEM$NOTEBOOKS_VNEXT_CREATE_INTERACTIVE('{schema_name}', '{service_name}', '{compute_pool_name}', ${service_spec.to_yaml()}$)"
            )
        else:
            return self.hybrid_executor.execute_query(
                f"select SYSTEM$NOTEBOOKS_VNEXT_CREATE_NON_INTERACTIVE('{schema_name}', '{service_name}', '{compute_pool_name}', ${service_spec.to_yaml()}$)"
            )
```

### 2.2 Current Requirements

| Requirement | Current Implementation | Pain Point |
|-------------|----------------------|------------|
| Container Image | Custom notebook container | Must build/maintain images |
| Compute Pool | Explicit pool creation | Teams must manage pools |
| ServiceSpec YAML | Full spec required | Complex configuration |
| Session Duration | Long-running (hours) | Token refresh needed |
| User Isolation | Per-user services | Expensive resource overhead |
| GPU Support | Instance family selection | Manual configuration |

### 2.3 Key Friction Points

1. **ServiceSpec Complexity**: Teams must understand containers, endpoints, volumes, secrets
2. **Compute Pool Management**: Pre-creation, sizing, auto-suspend configuration
3. **Image Management**: Building, pushing to repository, version management
4. **Token Refresh**: 60-minute OAuth token requires session management logic

---

## 3. Streamlit/SiS vNext Integration (Deep Dive)

### 3.1 Architecture Overview

Streamlit on SPCS uses a **managed service pattern** where the platform handles service creation automatically during bootstrap.

```
┌─────────────────────────────────────────────────────────────────┐
│                    STREAMLIT ON SPCS V2                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  User Request                                                   │
│       │                                                         │
│       ▼                                                         │
│  SYSTEM$STREAMLIT_BOOTSTRAP(streamlit_name, session_id)        │
│       │                                                         │
│       ▼                                                         │
│  StreamlitBootstrapHandler.getBootstrapAndRunActions()         │
│       │                                                         │
│       ├── Check if SPCS v2 runtime enabled                     │
│       │                                                         │
│       ▼                                                         │
│  SpcsInstanceHelpers.createServiceIfNeeded()                   │
│       │                                                         │
│       ├── Check for existing StRunningService                  │
│       ├── Verify service maps to active instance               │
│       │                                                         │
│       ▼                                                         │
│  createServiceViaJob() → SPCS Service Created                  │
│       │                                                         │
│       ▼                                                         │
│  Return bootstrap JSON with endpoint URL                       │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 Service Creation Flow

**File:** `/Users/mimam/snowflake-research/snowflake-main/GlobalServices/src/main/java/com/snowflake/app/stplatform/SpcsInstanceHelpers.java`

```java
// createServiceIfNeeded() - Lines 421-469
// Ensures a valid and active StRunningService exists for a Streamlit
public static void createServiceIfNeeded(Streamlit streamlit, Session session, boolean isV2) {
    // 1. Check for existing StRunningService
    // 2. Verify service maps to active instance
    // 3. Create new service via job if no valid service exists
}

// createServiceViaJob() - Lines 496-612
// Creates job to ensure service and waits for completion
// Uses STREAMLIT_SPCS_V2_SERVICE_BOOT_TIMEOUT_MS for timeout
```

### 3.3 Service Bootstrap System Function

**File:** `/Users/mimam/snowflake-research/snowflake-main/GlobalServices/src/main/java/com/snowflake/sql/compiler/semantic/functions/StreamlitFuncs.java`

```java
// SYSTEM$STREAMLIT_BOOTSTRAP - Line 1479
// Returns bootstrap JSON with everything needed by Snowsight to launch Streamlit
// Parameters: streamlit_name, app_view_request_id, is_private_link_request, query_id

// SYSTEM$STREAMLIT_BOOTSTRAP_STARTUP_STATUS - Line 1543
// Returns warehouse startup status for UI feedback

// SYSTEM$GET_STREAMLIT_API_TOKEN - Line 2187
// Returns API token for Streamlit access
```

### 3.4 Runtime Configuration

**File:** `/Users/mimam/snowflake-research/snowflake-main/Snowfort/snowfort/testlib/streamlit/constants.py`

```python
# Runtime options
WAREHOUSE_RUNTIME = "SYSTEM$WAREHOUSE_RUNTIME"      # Traditional warehouse-backed
SPCS_RUNTIME = "SYSTEM$BASIC_RUNTIME"               # SPCS v1
SPCS_V2_RUNTIME = "SYSTEM$ST_CONTAINER_RUNTIME_PY3_11"  # SPCS v2 (managed)
```

### 3.5 Feature Flags Required

| Parameter | Purpose |
|-----------|---------|
| `ENABLE_STREAMLIT_SPCS_RUNTIME_V2` | Enable SPCS v2 managed runtime |
| `ENABLE_STREAMLIT_SPCS_RUNTIME` | Enable SPCS v1 runtime |
| `ENABLE_SNOWSERVICES_USER_FACING_FEATURES` | Enable SPCS features |
| `ENABLE_STREAMLIT_PASS_ALTER_PROPERTIES_TO_SPCS_SERVICE` | Allow ALTER STREAMLIT to modify service |

### 3.6 Service-Streamlit Mapping

**File:** `/Users/mimam/snowflake-research/snowflake-main/GlobalServices/src/main/java/com/snowflake/metadata/dictionary/snowservices/StAppSnowservicesManagedObjectProvider.java`

```java
// SnowserviceSpec is used to manage service specifications
// Services created with managing_object_name referencing the Streamlit
// Service origin tracked via SnowserviceOrigin.STREAMLIT

// Query pattern to find service for a Streamlit:
// SELECT * FROM SHOW SERVICES WHERE managing_object_name = 'db.schema.streamlit_name'
```

### 3.7 Key Friction Points for Streamlit Team

| Friction | Severity | Details |
|----------|----------|---------|
| **Multiple Feature Flags** | High | 4+ flags needed to enable SPCS v2 |
| **Bootstrap Complexity** | High | Multi-step job creation, timeout handling |
| **Service Lifecycle** | Medium | Managing service state tied to Streamlit object |
| **Token Management** | Medium | 60-min OAuth token requires refresh |
| **Cold Start** | Medium | Service creation adds latency to first load |

---

## 4. ML Platform Integration (Deep Dive)

### 4.1 Architecture Overview

ML Platform uses SPCS for **model deployment** (inference services) and **training jobs**.

```
┌─────────────────────────────────────────────────────────────────┐
│                    ML PLATFORM ON SPCS                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  MODEL DEPLOYMENT (Inference)      TRAINING JOBS               │
│  ─────────────────────────────     ──────────────              │
│                                                                 │
│  Model Registry                    Training Script              │
│       │                                 │                       │
│       ▼                                 ▼                       │
│  SYSTEM$DEPLOY_MODEL()             SYSTEM$EXECUTE_ML_JOB()     │
│       │                                 │                       │
│       ▼                                 ▼                       │
│  Build Container Image             Create SPCS Job              │
│       │                                 │                       │
│       ▼                                 ▼                       │
│  Create SPCS Service               Execute in Container         │
│       │                                 │                       │
│       ▼                                 ▼                       │
│  Service Functions:                Output to Stage              │
│  • PREDICT()                            │                       │
│  • PREDICT_PROBA()                      ▼                       │
│  • DECISION_FUNCTION()             Job History Tracking         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 Model Deployment

**File:** `/Users/mimam/snowflake-research/snowflake-main/Snowfort/tests/ml_platform/t_model_deployment/src/model_deploy.py`

```python
class ModelDeploy:
    def verify_deploy_model(self):
        """Deploy model to SPCS using SYSTEM$DEPLOY_MODEL()"""
        # Creates inference service with:
        # - Container image built from model
        # - Service functions for inference
        # - Compute pool allocation

    def verify_inference(self):
        """Run inference queries on deployed models"""
        # Calls service functions: PREDICT, PREDICT_PROBA, etc.

    def verify_ingress_inference(self):
        """Handle ingress-based model serving (HTTP endpoints)"""
```

### 4.3 Deployment Specification

**File:** `/Users/mimam/snowflake-research/snowflake-main/Snowfort/tests/ml_platform/t_model_deployment/data/spec/deploy.yml.tmpl`

```yaml
models:
  - name: "{{ model_name }}"
    version: "{{ model_version }}"

image_build:
  compute_pool: "{{ compute_pool }}"
  image_repo: "{{ image_repo }}:443/test"
  force_rebuild: true

service:
  name: "{{ service_name }}"
  compute_pool: "{{ compute_pool }}"
  ingress_enabled: false
  max_instances: 1
  cpu: "1"
  num_workers: 1
  max_batch_rows: {{ max_batch_rows }}
  autocapture: {{ autocapture }}  # OTLP metrics
```

### 4.4 Inference Function Pattern

**File:** `/Users/mimam/snowflake-research/snowflake-main/Snowfort/tests/ml_platform/t_model_deployment/data/model/functions/predict.py`

```python
@vectorized(input=pd.DataFrame, max_batch_size=MAX_BATCH_SIZE)
def infer(df: pd.DataFrame) -> dict:
    """Vectorized inference function for batch predictions"""
    df.columns = input_cols
    input_df = df.astype(dtype=dtype_map)
    predictions_df = runner(input_df[input_cols])
    return predictions_df.to_dict("records")
```

### 4.5 ML Training Jobs

**File:** `/Users/mimam/snowflake-research/snowflake-main/Snowfort/tests/t_mljob/test_execute_ml_job.py`

```python
@CreateAccount.with_args(parameters={
    "enable_snowservices": True,
    "enable_snowservices_user_facing_features": True,
    "enable_execute_ml_job_function": True,
})
class TestExecuteMLJob:
    def test_execute_ml_job(self, hybrid_executor):
        spec_options = {
            "STAGE_PATH": "@payload_stage",
            "ARGS": [f"@payload_stage/app/{test_py_file}"],
            "ENTRYPOINT": ["/usr/local/bin/_entrypoint.sh"],
            "RUNTIME": "ml_runtime_version",
            "ENV_VARS": {"KEY": "value"},
            "SPEC_OVERRIDES": {},  # Custom container specs
            "ENABLE_METRICS": True,
        }

        job_options = {
            "QUERY_WAREHOUSE": "compute_wh",
            "EXTERNAL_ACCESS_INTEGRATIONS": [],
            "TARGET_INSTANCES": 1,
            "MIN_INSTANCES": 1,
            "ASYNC": False,
        }

        hybrid_executor.execute_query(
            "CALL SYSTEM$EXECUTE_ML_JOB(%s, %s, %s, %s)",
            bind_values=["", "CPU_POOL", json.dumps(spec_options), json.dumps(job_options)]
        )
```

### 4.6 Batch Inference

**File:** `/Users/mimam/snowflake-research/snowflake-main/Snowfort/tests/ml_platform/t_model_deployment/src/batch_inference.py`

```python
class BatchInference:
    # Configuration
    INPUT_STAGE_NAME = "input_stage"
    OUTPUT_STAGE_NAME = "output_stage"
    MAX_BATCH_ROWS = 1024
    COMPLETION_FILENAME = "_SUCCESS"

    # Features:
    # - Input/output stage management
    # - Batch row configuration
    # - Job status monitoring
    # - Completion markers
```

### 4.7 Model Serving Observability (OTLP)

**File:** `/Users/mimam/snowflake-research/snowflake-main/Snowfort/tests/ml_platform/t_model_deployment/test_inference_otlp_enabled.py`

```python
# Autocapture metrics automatically recorded:
record_attributes = {
    "snow.model_serving.function.name": "predict",
    "snow.model_serving.request.timestamp": "...",
    "snow.model_serving.response.timestamp": "...",
    "snow.model_serving.response.code": 200,
    # Input features
    "snow.model_serving.request.data.input_feature_0": "...",
    "snow.model_serving.request.data.input_feature_1": "...",
    # Output features
    "snow.model_serving.response.data.output_feature_0": "...",
}
```

### 4.8 ML Platform System Functions

| Function | Purpose |
|----------|---------|
| `SYSTEM$DEPLOY_MODEL()` | Deploy model as inference service |
| `SYSTEM$EXECUTE_ML_JOB()` | Execute training job on SPCS |
| `SYSTEM$INFERENCE_SERVICES_FOR_MODEL()` | List services for a model |
| `SYSTEM$GET_DEPLOY_MODEL_RESOURCE_ESTIMATE()` | Estimate compute resources |
| `SYSTEM$ML_PLATFORM_CAPABILITIES()` | Get platform capabilities |

### 4.9 ML Platform Capabilities

```json
{
  "ENABLE_INLINE_DEPLOYMENT_SPEC_FROM_CLIENT_VERSION": "1.8.6",
  "FEATURE_MODEL_INFERENCE_AUTOCAPTURE": "ENABLED",
  "IMAGE_BUILD_DEFAULT_IMAGE_REPO": "snowflake.default_image_store.ml_repo",
  "SPCS_MODEL_ENABLE_JOB_INFERENCE": "true",
  "SPCS_MODEL_ENABLE_INFERENCE_PROXY_CONTAINER": "true"
}
```

### 4.10 Key Friction Points for ML Team

| Friction | Severity | Details |
|----------|----------|---------|
| **Image Build Time** | High | PyTorch/TensorFlow take 3-5+ minutes |
| **Deploy Spec Complexity** | High | YAML with models, image_build, service configs |
| **Resource Estimation** | Medium | Guessing CPU/memory/GPU needs |
| **Batch Configuration** | Medium | max_batch_rows, workers tuning |
| **Multiple System Functions** | Medium | Different functions for deploy vs train |

---

## 5. Native Apps (NASPCS) Integration

### 5.1 Architecture

Native Apps can include SPCS services that run in consumer accounts.

**File:** `/Users/mimam/snowflake-research/snowflake-main/Snowfort/tests/native_apps/snowservices/`

```
┌─────────────────────────────────────────────────────────────────┐
│                    NATIVE APPS + SPCS                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Provider Account              Consumer Account                 │
│  ─────────────────             ────────────────                │
│                                                                 │
│  Native App Package            Installed App                    │
│       │                             │                           │
│       ├── setup.sql                 ├── CREATE SERVICE          │
│       ├── service_spec.yaml         ├── Compute Pool (consumer) │
│       └── container images          └── Running Service         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 5.2 Key Test Files

- `test_naspcs_restrictions.py` - Security restrictions for app services
- `test_naspcs_nested_service.py` - Nested service patterns
- `test_naspcs_query_warehouse_reference_service.py` - Warehouse integration
- `test_naspcs_eai_reference_service.py` - External access
- `test_naspcs_image_version_pinning.py` - Image version control
- `test_naspcs_bg_suspend_resume.py` - Background service lifecycle

---

## 6. Cortex Search Integration

### 6.1 Simplified Pattern (Model to Follow)

**File:** `/Users/mimam/snowflake-research/snowflake-main/Snowfort/snowfort/testlib/native_apps/api/cortex_search_service_api.py`

```python
def create(self, name, schema, search_column, warehouse, source_query, target_lag="1 minute"):
    sql = (
        f"CREATE CORTEX SEARCH SERVICE {schema.fqn}.{name} "
        f"ON {search_column} "
        f"WAREHOUSE = {warehouse} "
        f"TARGET_LAG = '{target_lag}' "
        f"AS {source_query}"
    )
    return self._execute_sql(sql)
```

**Why This Works:**
- Single SQL statement
- No compute pool management
- No container images
- No ServiceSpec
- Platform handles everything

---

## 6. OpenFlow & SPCS Integration (Deep Dive)

### 6.1 What is OpenFlow?

OpenFlow is Snowflake's **orchestration framework for serverless workflows** - designed to coordinate distributed compute tasks across Snowflake services. It provides a declarative way to define multi-step data pipelines and ML workflows that span multiple execution engines.

```
┌─────────────────────────────────────────────────────────────────┐
│                        OPENFLOW ARCHITECTURE                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  User defines workflow (DAG)                                    │
│       │                                                         │
│       ▼                                                         │
│  OpenFlow Orchestrator                                         │
│       │                                                         │
│       ├─▶ Task 1: Warehouse Query                             │
│       ├─▶ Task 2: SPCS Container Job                          │
│       ├─▶ Task 3: Python UDF                                  │
│       ├─▶ Task 4: External API Call                           │
│       └─▶ Task 5: SPCS Model Inference                        │
│                                                                 │
│  Handles: Scheduling, Dependencies, Retries, State             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 6.2 OpenFlow's Role in the Serverless Ecosystem

| Component | Role | Relationship to SPCS |
|-----------|------|---------------------|
| **OpenFlow** | Workflow orchestration | Schedules and coordinates SPCS jobs |
| **SPCS** | Container execution runtime | Executes OpenFlow tasks in containers |
| **Warehouses** | SQL query processing | Alternative execution for SQL-heavy tasks |
| **Cortex** | AI/ML inference | Can be orchestrated by OpenFlow via SPCS |

### 6.3 How OpenFlow Interacts with SPCS

OpenFlow uses SPCS as an **execution backend** for containerized tasks. The integration pattern:

```
┌─────────────────────────────────────────────────────────────────┐
│              OPENFLOW → SPCS INTEGRATION FLOW                   │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. OpenFlow Workflow Definition                               │
│     workflow:                                                   │
│       - task_id: "preprocess_data"                             │
│         type: "spcs_job"                                        │
│         compute_pool: "ML_POOL"                                │
│         container_image: "data-prep:v1"                        │
│                                                                 │
│  2. OpenFlow Scheduler triggers task                           │
│       │                                                         │
│       ▼                                                         │
│  3. Calls SYSTEM$CREATE_JOB (SPCS Job API)                     │
│       │                                                         │
│       ├── Compute Pool: ML_POOL                                │
│       ├── Image: data-prep:v1                                  │
│       ├── Command: python preprocess.py --input @stage/data    │
│       └── Environment: {TASK_ID: "task_123", ...}              │
│                                                                 │
│  4. SPCS creates job in container                              │
│       │                                                         │
│       ▼                                                         │
│  5. OpenFlow monitors job status                               │
│       │                                                         │
│       ├── RUNNING → continues                                  │
│       ├── SUCCEEDED → trigger next task                        │
│       └── FAILED → retry or fail workflow                      │
│                                                                 │
│  6. Job completes, writes output to stage                      │
│       │                                                         │
│       ▼                                                         │
│  7. OpenFlow reads output, proceeds to next task               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 6.4 OpenFlow Task Types for SPCS

| Task Type | SPCS Component | Use Case |
|-----------|---------------|----------|
| **`spcs_job`** | SPCS Jobs (batch) | One-time containerized tasks |
| **`spcs_service`** | SPCS Services | Long-running API servers |
| **`spcs_inference`** | Model Serving | ML model predictions |
| **`spcs_training`** | ML Training Jobs | Model training workloads |

### 6.5 Example: ML Pipeline with OpenFlow + SPCS

```yaml
# OpenFlow workflow definition
name: ml_training_pipeline
version: 1.0

tasks:
  # Step 1: Feature engineering (SPCS Job)
  - id: feature_engineering
    type: spcs_job
    config:
      compute_pool: CPU_POOL
      image: feature-eng:v2
      command: ["python", "features.py"]
      input_stage: "@raw_data/inputs"
      output_stage: "@processed/features"
      env:
        BATCH_SIZE: "10000"

  # Step 2: Model training (SPCS Training Job)
  - id: train_model
    type: spcs_training
    depends_on: [feature_engineering]
    config:
      compute_pool: GPU_POOL
      image: pytorch-trainer:v1
      gpu: "A10G"
      command: ["python", "train.py"]
      input_stage: "@processed/features"
      output_stage: "@models/checkpoints"
      hyperparameters:
        learning_rate: 0.001
        epochs: 100

  # Step 3: Model evaluation (SPCS Job)
  - id: evaluate
    type: spcs_job
    depends_on: [train_model]
    config:
      compute_pool: CPU_POOL
      image: model-eval:v1
      command: ["python", "evaluate.py"]
      input_stage: "@models/checkpoints"
      output_stage: "@metrics/results"

  # Step 4: Deploy model (SPCS Service)
  - id: deploy_model
    type: spcs_service
    depends_on: [evaluate]
    condition: "metrics.accuracy > 0.95"
    config:
      compute_pool: GPU_INFERENCE_POOL
      image: model-server:v1
      service_name: "prod_model_v2"
      endpoint_public: true
      min_instances: 2
      max_instances: 10
```

### 6.6 OpenFlow-SPCS State Management

OpenFlow needs to track the state of SPCS jobs across the workflow:

```
┌─────────────────────────────────────────────────────────────────┐
│                 OPENFLOW STATE TRACKING                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  OpenFlow Metadata Store (FDB)                                 │
│       │                                                         │
│       ├── Workflow Instance: wf_12345                          │
│       │       ├── Status: RUNNING                              │
│       │       ├── Started: 2026-02-17T10:00:00Z               │
│       │       └── Tasks:                                       │
│       │               ├── feature_engineering                  │
│       │               │     ├── SPCS Job ID: job_abc123       │
│       │               │     ├── Status: SUCCEEDED             │
│       │               │     └── Output: @processed/features   │
│       │               ├── train_model                          │
│       │               │     ├── SPCS Job ID: job_def456       │
│       │               │     ├── Status: RUNNING               │
│       │               │     └── Progress: 45%                 │
│       │               └── evaluate                             │
│       │                     └── Status: PENDING               │
│                                                                 │
│  Integration Points:                                            │
│  • SHOW SERVICES/JOBS in SPCS → poll for status               │
│  • SYSTEM$GET_JOB_STATUS(job_id) → detailed state             │
│  • SYSTEM$CANCEL_JOB(job_id) → failure handling               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 6.7 Key SPCS APIs Used by OpenFlow

| API / System Function | Purpose | OpenFlow Usage |
|----------------------|---------|----------------|
| `SYSTEM$CREATE_JOB()` | Create batch job | Submit containerized tasks |
| `SYSTEM$GET_JOB_STATUS()` | Check job state | Poll for completion |
| `SYSTEM$GET_JOB_LOGS()` | Retrieve logs | Debugging failed tasks |
| `SYSTEM$CANCEL_JOB()` | Stop job | Workflow cancellation |
| `CREATE SERVICE` | Deploy service | Long-running task endpoints |
| `SHOW JOBS` | List jobs | Monitoring active tasks |
| `DESCRIBE JOB` | Get job details | Inspect configuration |

### 6.8 Data Flow Between OpenFlow and SPCS

```
┌─────────────────────────────────────────────────────────────────┐
│                    DATA PASSING PATTERNS                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Pattern 1: Stage-Based (Recommended)                          │
│  ─────────────────────────────                                 │
│  Task A (SPCS) → writes to @stage/output                       │
│                  │                                              │
│                  ▼                                              │
│  OpenFlow → reads metadata from @stage/output/_metadata.json   │
│                  │                                              │
│                  ▼                                              │
│  Task B (SPCS) → reads from @stage/output                      │
│                                                                 │
│  Pattern 2: Table-Based                                        │
│  ──────────────────────                                        │
│  Task A (SPCS) → writes to TABLE intermediate_results          │
│                  │                                              │
│                  ▼                                              │
│  OpenFlow → SELECT status FROM intermediate_results            │
│                  │                                              │
│                  ▼                                              │
│  Task B (SPCS) → SELECT * FROM intermediate_results            │
│                                                                 │
│  Pattern 3: Service Endpoint                                   │
│  ───────────────────────                                       │
│  Task A (SPCS Service) → exposes /api/status endpoint          │
│                  │                                              │
│                  ▼                                              │
│  OpenFlow → HTTP GET to service endpoint                       │
│                  │                                              │
│                  ▼                                              │
│  Task B (SPCS) → HTTP POST to Task A service                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 6.9 Retry and Failure Handling

OpenFlow manages SPCS job failures through configurable retry policies:

```yaml
# OpenFlow task with retry configuration
- id: flaky_processing
  type: spcs_job
  config:
    compute_pool: CPU_POOL
    image: processor:v1
    command: ["python", "process.py"]
  retry_policy:
    max_attempts: 3
    backoff_seconds: [60, 300, 900]  # 1min, 5min, 15min
    retry_on:
      - "container_exit_code != 0"
      - "spcs_job_status == 'FAILED'"
    fail_fast_on:
      - "container_exit_code == 42"  # Unrecoverable error
```

### 6.10 Monitoring and Observability

OpenFlow aggregates observability data from SPCS jobs:

| Metric | Source | OpenFlow Action |
|--------|--------|----------------|
| **Container Logs** | `SYSTEM$GET_SERVICE_LOGS()` | Store in workflow metadata |
| **Resource Usage** | SPCS metrics API | Track for cost attribution |
| **Task Duration** | Job start/end timestamps | Identify bottlenecks |
| **Error Messages** | Container stderr | Surface to workflow UI |
| **Data Lineage** | Stage reads/writes | Build dependency graph |

### 6.11 Scaling Patterns

```
┌─────────────────────────────────────────────────────────────────┐
│              OPENFLOW SCALING WITH SPCS                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Pattern 1: Parallel Fan-Out                                   │
│  ──────────────────────────                                    │
│  Input Data: 100 files                                         │
│       │                                                         │
│       ├─▶ SPCS Job 1 (files 1-10)                             │
│       ├─▶ SPCS Job 2 (files 11-20)                            │
│       ├─▶ ...                                                  │
│       └─▶ SPCS Job 10 (files 91-100)                          │
│                  │                                              │
│                  ▼                                              │
│       Merge Task → Combine results                             │
│                                                                 │
│  Pattern 2: Dynamic Scaling                                    │
│  ────────────────────────                                      │
│  OpenFlow → Check queue depth                                  │
│       │                                                         │
│       ├─▶ If queue > 1000: Launch 10 SPCS jobs                │
│       ├─▶ If queue < 100: Launch 2 SPCS jobs                  │
│       └─▶ Scale down when queue empty                          │
│                                                                 │
│  Pattern 3: Conditional Execution                              │
│  ──────────────────────────────                                │
│  Task A → Check data quality                                   │
│       │                                                         │
│       ├─▶ If quality > 95%: Skip cleaning (warehouse query)    │
│       └─▶ If quality < 95%: Launch SPCS cleaning job           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 6.12 OpenFlow vs Direct SPCS Usage

| Dimension | OpenFlow + SPCS | Direct SPCS | Winner |
|-----------|----------------|-------------|--------|
| **Single Task** | Overhead | Direct | Direct SPCS |
| **Multi-Step Pipeline** | Orchestration built-in | Manual coordination | OpenFlow |
| **Retry Logic** | Declarative | Manual | OpenFlow |
| **Conditional Branching** | DAG support | Complex code | OpenFlow |
| **Monitoring** | Workflow-level | Job-level | OpenFlow |
| **Scheduling** | Cron + dependencies | Separate scheduler needed | OpenFlow |
| **Cost Attribution** | Per-workflow | Per-job | OpenFlow |

### 6.13 Use Cases: When to Use OpenFlow with SPCS

| Use Case | Why OpenFlow + SPCS? |
|----------|---------------------|
| **ETL Pipelines** | Multi-stage data transformation with dependencies |
| **ML Training** | Data prep → train → evaluate → deploy sequence |
| **Batch Processing** | Daily/hourly jobs with retry logic |
| **Multi-Tenant Workflows** | Per-customer data processing at scale |
| **Hybrid Compute** | Mix warehouse queries + containerized tasks |
| **Long-Running Jobs** | Need checkpointing and failure recovery |

### 6.14 Current Limitations

| Limitation | Impact | Workaround |
|------------|--------|------------|
| **No Live Streaming** | Can't handle real-time events | Use SPCS services directly |
| **Fixed DAG** | Can't dynamically generate tasks | Pre-generate all possible branches |
| **Limited Task Types** | Only supports predefined types | Use generic `spcs_job` |
| **State Storage in FDB** | Large state can bloat metadata | Store data in stages, only refs in OpenFlow |
| **No Cross-Account** | Workflows stay in single account | Manual coordination |

### 6.15 Future Evolution: Serverless Integration

With a Modal.ai-style serverless interface, OpenFlow could simplify:

**Current (OpenFlow + SPCS):**
```yaml
- id: process_data
  type: spcs_job
  config:
    compute_pool: MY_POOL
    image: processor:v1
    command: ["python", "process.py"]
    # ... 15 more config lines
```

**Future (OpenFlow + Serverless SDK):**
```python
import snowflake.serverless as sf

@sf.function(tier="medium")
def process_data(batch):
    return transform(batch)

# OpenFlow automatically calls sf.deploy() and manages lifecycle
workflow.add_task("process_data", process_data)
```

This would eliminate:
- Compute pool management
- Container image versioning
- ServiceSpec complexity
- Manual resource estimation

---

## 7. ServiceSpec Deep Dive

### 6.1 Core Class Structure

**File:** `/Users/mimam/snowflake-research/snowflake-main/Snowfort/snowfort/testlib/snowservices/service_spec.py`

```python
class ServiceSpec:
    def __init__(self):
        self.spec = {
            "containers": [],
            "endpoints": [],
            "volumes": [],
            "logExporters": {},
            "platformMonitor": {}
        }
        self.capabilities: Dict[str, dict] = {}
        self.service_roles: List[Dict[str, Union[str, List[str]]]] = []

class ContainerSpec:
    def __init__(self, name, image, command=None, args=None, env=None,
                 readiness_probe=None, volume_mounts=None, resources=None, secrets=None):
        # Full container configuration

class ResourceSpec:
    def __init__(self, requests=None, limits=None):
        # CPU/memory requests and limits

DEFAULT_RESOURCE_SPEC = ResourceSpec(
    requests={"memory": "450Mi", "cpu": "250m"},
    limits={"memory": "450Mi", "cpu": "250m"}
)
```

### 6.2 Pain Points with ServiceSpec

| Pain Point | Description | Impact |
|------------|-------------|--------|
| **Verbose Configuration** | 50+ lines for simple service | Developer friction |
| **Resource Guessing** | Teams don't know what resources they need | Over/under provisioning |
| **Image Management** | Building, tagging, pushing images | DevOps overhead |
| **Volume Configuration** | Stage mounting, permissions | Security concerns |
| **Endpoint Setup** | Public/private, CORS, auth | Complexity |

---

## 8. Compute Pool Management

### 8.1 Instance Families

| Instance Family | vCPU | Memory | Storage | GPU | Use Case |
|-----------------|------|--------|---------|-----|----------|
| CPU_X64_XS | 1 | 4 GiB | 93 GiB | None | Lightweight tasks |
| CPU_X64_S | 3 | 13 GiB | 93 GiB | None | Standard workloads |
| CPU_X64_M | 6 | 26 GiB | 93 GiB | None | Medium workloads |
| CPU_X64_L | 28 | 116 GiB | 93 GiB | None | Large workloads |
| GPU_NV_S | 6 | 27 GiB | 93 GiB | 1x A10G | ML inference |
| GPU_NV_M | 44 | 178 GiB | 93 GiB | 4x A10G | ML training |
| GPU_NV_XL | 188 | 1843 GiB | 93 GiB | 8x H100 | Large ML |

---

## 9. Friction Points Summary

### 9.1 High Severity (Must Solve)

| Friction | Current State | Impact |
|----------|--------------|--------|
| **ServiceSpec Complexity** | 50+ lines YAML | Teams avoid SPCS |
| **Compute Pool Management** | Manual DDL, sizing | Delays deployment |
| **Image Management** | Build/push/version | DevOps overhead |

### 9.2 Medium Severity

| Friction | Current State | Impact |
|----------|--------------|--------|
| **Token Refresh** | 60-min expiry | Session failures |
| **Resource Estimation** | Guesswork | Over/under provisioning |
| **Cold Start** | Minutes | Poor UX |

---

## 10. Vision: Modal.ai-Style Serverless Interface

### 10.1 What is Modal.ai?

Modal.ai provides a developer experience where you:
1. **Write code in any language** (Python, etc.)
2. **Decorate functions** with resource requirements
3. **Deploy instantly** - platform handles everything
4. **Get endpoints automatically** - no infrastructure config

### 10.2 Proposed Snowflake Serverless Interface

#### Python SDK (Primary Interface)

```python
import snowflake.serverless as sf

# Define a function - deploy as serverless endpoint
@sf.function(
    cpu=2,
    memory="4GB",
    gpu="A10G",  # Optional
    timeout=300,
    secrets=["my_api_key"]
)
def process_data(input_data: dict) -> dict:
    """This function runs on Snowflake's infrastructure."""
    import pandas as pd
    # Your code here - any Python libraries
    result = expensive_ml_operation(input_data)
    return {"result": result}

# Deploy and get endpoint
endpoint = sf.deploy(process_data)
print(f"Endpoint: {endpoint.url}")

# Call it
result = endpoint.call({"data": [1, 2, 3]})
```

#### Streamlit Apps

```python
import snowflake.serverless as sf

@sf.app(
    cpu=1,
    memory="2GB",
    concurrent_users=10
)
def my_streamlit_app():
    import streamlit as st
    st.title("My App")
    # Streamlit code here

# Deploy
app = sf.deploy(my_streamlit_app)
print(f"App URL: {app.url}")
```

#### Notebooks

```python
import snowflake.serverless as sf

# Deploy a notebook as an interactive service
notebook = sf.notebook(
    source="@my_stage/analysis.ipynb",
    cpu=4,
    memory="16GB",
    gpu="A10G"  # For ML workloads
)

print(f"Notebook URL: {notebook.url}")
```

#### Batch Jobs

```python
import snowflake.serverless as sf

@sf.job(
    cpu=8,
    memory="32GB",
    gpu="H100",
    schedule="0 0 * * *"  # Daily at midnight
)
def nightly_training():
    """Runs as a scheduled batch job."""
    train_model()
    save_to_stage("@models/latest")

# Deploy
job = sf.deploy(nightly_training)
```

### 10.3 Multi-Language Support

#### Python (Native)
```python
@sf.function(cpu=2, memory="4GB")
def my_python_function(data):
    return process(data)
```

#### Container (Any Language)
```python
# For non-Python: provide a Dockerfile or image
endpoint = sf.container(
    image="my-go-service:latest",  # Go, Rust, Java, etc.
    port=8080,
    cpu=2,
    memory="4GB"
)
```

#### From Stage (Pre-built)
```python
# Deploy from files in a stage
endpoint = sf.from_stage(
    stage="@my_code/app",
    entrypoint="main.py",
    cpu=2,
    memory="4GB"
)
```

### 10.4 Resource Tiers (Simplified)

Instead of instance families, offer simple tiers:

| Tier | CPU | Memory | GPU | Use Case |
|------|-----|--------|-----|----------|
| `xs` | 0.5 | 1 GB | - | Tiny tasks |
| `small` | 1 | 2 GB | - | Light workloads |
| `medium` | 2 | 4 GB | - | Standard |
| `large` | 4 | 16 GB | - | Heavy compute |
| `xl` | 8 | 32 GB | - | Very heavy |
| `gpu-small` | 4 | 16 GB | A10G | ML inference |
| `gpu-large` | 8 | 32 GB | A10G x4 | ML training |
| `gpu-xl` | 16 | 64 GB | H100 | Large models |

```python
# Use tier instead of specific resources
@sf.function(tier="gpu-small")
def ml_inference(data):
    return model.predict(data)
```

---

## 11. Architecture: What Platform Handles

### 11.1 Developer Provides

| Input | Required | Example |
|-------|----------|---------|
| **Code** | Yes | Python function, container, stage path |
| **Resources** | Yes | Tier or cpu/memory/gpu |
| **Secrets** | Optional | `["api_key", "db_password"]` |
| **Schedule** | Optional | Cron expression for jobs |

### 11.2 Platform Handles (Invisible to Developer)

| Responsibility | How Platform Handles |
|----------------|---------------------|
| **Compute Pool** | Auto-provision from shared pool |
| **Instance Family** | Map tier to optimal instance |
| **ServiceSpec** | Generate from decorator |
| **Container Image** | Build from code automatically |
| **Image Registry** | Managed internal registry |
| **Token Refresh** | Transparent session management |
| **Endpoints** | Auto-generate public/private URLs |
| **Scaling** | Auto-scale based on load |
| **Cold Start** | Keep warm pools for fast start |
| **Monitoring** | Built-in metrics and logs |
| **Cost Attribution** | Track per-function/per-user |

### 11.3 Behind the Scenes

When a developer runs `sf.deploy(my_function)`:

```
1. Code Analysis
   └── Parse function, detect dependencies
   └── Identify resource requirements from decorator

2. Image Building (if needed)
   └── Generate Dockerfile from dependencies
   └── Build container image
   └── Push to internal registry

3. ServiceSpec Generation
   └── Create ContainerSpec with image, resources
   └── Create EndpointSpec for HTTP access
   └── Add secrets, volumes as needed

4. Compute Pool Selection
   └── Find/create pool matching tier
   └── Handle auto-scaling configuration

5. Service Deployment
   └── Call internal SYSTEM$CREATE_SERVICE
   └── Wait for READY state

6. Endpoint Registration
   └── Generate unique URL
   └── Configure auth (Snowflake OAuth)
   └── Return endpoint to developer
```

---

## 12. API Design

### 12.1 Core Decorators

```python
# Function endpoint
@sf.function(cpu=2, memory="4GB", gpu=None, timeout=300, secrets=[])

# Streamlit app
@sf.app(cpu=1, memory="2GB", concurrent_users=10)

# Batch job
@sf.job(cpu=4, memory="8GB", schedule="0 * * * *")

# Long-running service
@sf.service(cpu=2, memory="4GB", min_instances=1, max_instances=10)
```

### 12.2 Deployment Methods

```python
# Deploy and get endpoint
endpoint = sf.deploy(my_function)

# Deploy from container
endpoint = sf.container(image="my-image:latest", port=8080, cpu=2)

# Deploy from stage
endpoint = sf.from_stage(stage="@code/app", entrypoint="main.py")

# Deploy notebook
notebook = sf.notebook(source="@notebooks/analysis.ipynb", cpu=4)
```

### 12.3 Endpoint Operations

```python
# Get endpoint info
print(endpoint.url)
print(endpoint.status)
print(endpoint.logs())

# Call endpoint
result = endpoint.call({"input": "data"})

# Async call
future = endpoint.call_async({"input": "data"})
result = future.result()

# Scale
endpoint.scale(min_instances=2, max_instances=10)

# Stop/Start
endpoint.stop()
endpoint.start()

# Delete
endpoint.delete()
```

### 12.4 Resource Specification Options

```python
# Option 1: Explicit resources
@sf.function(cpu=2, memory="4GB", gpu="A10G")

# Option 2: Tier-based
@sf.function(tier="gpu-small")

# Option 3: Auto (platform decides based on code analysis)
@sf.function(tier="auto")
```

---

## 13. Comparison: Current vs Proposed

### 13.1 Current SPCS (50+ lines)

```python
# 1. Create compute pool (separate DDL)
# CREATE COMPUTE POOL my_pool INSTANCE_FAMILY=CPU_X64_S MIN_NODES=1...

# 2. Build and push image
# docker build -t my-image .
# docker push registry/my-image:latest

# 3. Write ServiceSpec YAML
service_spec = ServiceSpec()
service_spec.add_container(ContainerSpec(
    name="main",
    image="registry/my-image:latest",
    resources=ResourceSpec(
        requests={"memory": "4Gi", "cpu": "2"},
        limits={"memory": "4Gi", "cpu": "2"}
    ),
    env={"VAR": "value"},
    readiness_probe=ReadinessProbe(port=8080, path="/health"),
    volume_mounts=[VolumeMount(name="data", path="/data")],
    secrets=[SecretMount(name="api_key", path="/secrets")]
))
service_spec.add_endpoint(EndpointSpec(
    name="app",
    port=8080,
    is_public=True,
    protocol="HTTPS"
))
service_spec.add_volume(VolumeSpec(name="data", source="@mystage"))

# 4. Create service
cursor.execute(f"""
    CREATE SERVICE my_service
    IN COMPUTE POOL my_pool
    FROM SPECIFICATION $${service_spec.to_yaml()}$$
""")

# 5. Wait for ready
# SELECT SYSTEM$GET_SERVICE_STATUS('my_service')...

# 6. Get endpoint URL
# SHOW ENDPOINTS IN SERVICE my_service
```

### 13.2 Proposed Serverless (5 lines)

```python
import snowflake.serverless as sf

@sf.function(cpu=2, memory="4GB")
def my_function(data):
    return process(data)

endpoint = sf.deploy(my_function)
# Done. endpoint.url is ready to use.
```

### 13.3 Reduction

| Metric | Current | Proposed | Reduction |
|--------|---------|----------|-----------|
| Lines of code | 50+ | 5 | 90% |
| Concepts to learn | 8+ | 2 | 75% |
| Time to deploy | 30+ min | < 1 min | 97% |
| Infrastructure knowledge | High | None | 100% |

---

## 14. Implementation Considerations

### 14.1 For Internal Teams (Notebooks, Streamlit, Cortex)

The serverless interface can be adopted incrementally:

1. **Phase 1**: New projects use `sf.deploy()`
2. **Phase 2**: Existing services migrate to decorators
3. **Phase 3**: Deprecate direct ServiceSpec usage

### 14.2 Backward Compatibility

```python
# Escape hatch: access underlying ServiceSpec if needed
@sf.function(cpu=2, memory="4GB")
def my_function(data):
    return process(data)

# Get generated ServiceSpec for debugging/customization
spec = sf.get_spec(my_function)
print(spec.to_yaml())

# Override specific settings
endpoint = sf.deploy(my_function, spec_overrides={
    "endpoints": [{"cors": {"allowed_origins": ["*"]}}]
})
```

### 14.3 Snowflake Integration

```python
# Access Snowflake data from within functions
@sf.function(cpu=2, memory="4GB")
def query_and_process():
    # session is automatically available
    df = session.sql("SELECT * FROM my_table").to_pandas()
    return process(df)

# Use Snowflake secrets
@sf.function(cpu=2, memory="4GB", secrets=["my_api_key"])
def call_external_api():
    import os
    api_key = os.environ["MY_API_KEY"]  # Automatically injected
    return call_api(api_key)
```

---

## 15. Key Files Reference

### 15.1 Core SPCS Infrastructure

| File | Purpose |
|------|---------|
| `Snowfort/snowfort/testlib/snowservices/service_spec.py` | ServiceSpec class definition |
| `Snowfort/snowfort/testlib/snowservices/service_api.py` | Service API operations |
| `Snowfort/snowfort/testlib/snowservices/compute_pool_api.py` | Compute pool management |
| `Snowfort/tests/snowservices/lib/compute_pool.py` | Compute pool utilities |
| `GlobalServices/modules/snowapi/snowapi-codegen/.../compute-pool.yaml` | REST API spec |
| `GlobalServices/src/main/java/com/snowflake/sql/execution/ExecCreateService.java` | Service creation |

### 15.2 Notebooks vNext

| File | Purpose |
|------|---------|
| `Snowfort/tests/notebooks_vnext/api/notebook_service_api.py` | Service creation API |
| `Snowfort/tests/notebooks_vnext/test_notebook_managed_service.py` | Integration tests |
| `GlobalServices/modules/notebooks-vnext/notebooks-vnext-impl/` | Implementation |
| `Snowfort/tests/t_notebooks/test_notebook_spcs_execute.py` | SPCS execution tests |
| `Snowfort/tests/t_notebooks/test_notebook_spcs_interactive.py` | Interactive mode |

### 15.3 Streamlit/SiS vNext

| File | Purpose |
|------|---------|
| `GlobalServices/src/main/java/com/snowflake/app/stplatform/SpcsInstanceHelpers.java` | Service management |
| `GlobalServices/src/main/java/com/snowflake/app/streamlit/bootstrap/StreamlitBootstrapHandler.java` | Bootstrap handler |
| `GlobalServices/src/main/java/com/snowflake/sql/compiler/semantic/functions/StreamlitFuncs.java` | System functions |
| `Snowfort/tests/t_streamlit/test_streamlit_on_spcs_v2.py` | SPCS v2 tests |
| `Snowfort/tests/t_streamlit/test_streamlit_bootstrap_spcs.py` | Bootstrap tests |
| `streamlit-container-runtime/internal/snowflake/config.go` | Token auth |

### 15.4 ML Platform

| File | Purpose |
|------|---------|
| `Snowfort/tests/ml_platform/t_model_deployment/src/model_deploy.py` | Model deployment |
| `Snowfort/tests/ml_platform/t_model_deployment/data/spec/deploy.yml.tmpl` | Deploy spec |
| `Snowfort/tests/ml_platform/t_model_deployment/src/batch_inference.py` | Batch inference |
| `Snowfort/tests/t_mljob/test_execute_ml_job.py` | Training jobs |
| `GlobalServices/src/main/java/com/snowflake/sql/compiler/semantic/functions/mlruntime/ExecuteMlJob.java` | Job execution |
| `GlobalServices/src/main/java/com/snowflake/sql/compiler/semantic/functions/model/DeployModel.java` | Model deploy |

### 15.5 Cortex Services

| File | Purpose |
|------|---------|
| `GlobalServices/modules/cortex/cortex-protos/.../cortex_chat_service.proto` | Orchestrator |
| `Snowfort/snowfort/testlib/native_apps/api/cortex_search_service_api.py` | Search API |
| `Snowfort/tests/t_cortex_search/test_cortex_search_stage_to_service.py` | Service tests |
| `GlobalServices/src/main/java/com/snowflake/cortex/rest/inference/` | Inference API |

### 15.6 Intelligence Agent

| File | Purpose |
|------|---------|
| `GlobalServices/.../ExecCreateSnowflakeIntelligence.java` | SI creation |
| `GlobalServices/.../SnowflakeIntelligenceEntityMapping.java` | SI-Agent mapping |
| `GlobalServices/.../CortexAgent.java` | Agent implementation |

### 15.7 Native Apps (NASPCS)

| File | Purpose |
|------|---------|
| `Snowfort/tests/native_apps/snowservices/test_naspcs_restrictions.py` | Security restrictions |
| `Snowfort/tests/native_apps/snowservices/test_naspcs_nested_service.py` | Nested services |
| `Tests/system_tests/native_app_v2_tests/tests/lib/snowservices/services.py` | Service utilities |

---

## 16. Complete Friction Analysis by Product

### 16.1 Friction Matrix

| Product | ServiceSpec | Compute Pool | Image Mgmt | Token Refresh | Resource Est. | Cold Start |
|---------|-------------|--------------|------------|---------------|---------------|------------|
| Notebooks vNext | HIGH | HIGH | HIGH | HIGH | MEDIUM | MEDIUM |
| Streamlit vNext | MEDIUM* | MEDIUM* | LOW* | HIGH | LOW | MEDIUM |
| ML Model Serving | HIGH | HIGH | HIGH | LOW | HIGH | LOW |
| ML Training Jobs | MEDIUM | HIGH | MEDIUM | LOW | HIGH | LOW |
| Cortex Search | LOW | LOW | LOW | LOW | LOW | LOW |
| Cortex Agent | MEDIUM | LOW | LOW | LOW | LOW | LOW |
| Intelligence Agent | LOW | LOW | LOW | LOW | LOW | LOW |
| Native Apps | HIGH | HIGH | HIGH | LOW | MEDIUM | MEDIUM |

*Streamlit vNext uses managed services, reducing some friction

### 16.2 Common Pain Points Across All Products

| Pain Point | Products Affected | Severity |
|------------|-------------------|----------|
| **ServiceSpec YAML** | Notebooks, ML, Native Apps | HIGH |
| **Compute Pool DDL** | All except Cortex Search | HIGH |
| **Image Build Time** | ML (3-5 min), Notebooks (2-3 min) | HIGH |
| **Feature Flags** | Streamlit (4+), Notebooks (3+) | MEDIUM |
| **Token 60-min Expiry** | Notebooks, Streamlit | MEDIUM |
| **Resource Guessing** | All | MEDIUM |

---

## 17. Summary: All SPCS System Functions

| Function | Product | Purpose |
|----------|---------|---------|
| `SYSTEM$NOTEBOOKS_VNEXT_CREATE_INTERACTIVE` | Notebooks | Create interactive notebook service |
| `SYSTEM$NOTEBOOKS_VNEXT_CREATE_NON_INTERACTIVE` | Notebooks | Create batch notebook service |
| `SYSTEM$STREAMLIT_BOOTSTRAP` | Streamlit | Bootstrap Streamlit with service |
| `SYSTEM$STREAMLIT_BOOTSTRAP_STARTUP_STATUS` | Streamlit | Check startup status |
| `SYSTEM$GET_STREAMLIT_API_TOKEN` | Streamlit | Get API token |
| `SYSTEM$DEPLOY_MODEL` | ML | Deploy model as service |
| `SYSTEM$EXECUTE_ML_JOB` | ML | Execute training job |
| `SYSTEM$INFERENCE_SERVICES_FOR_MODEL` | ML | List inference services |
| `SYSTEM$GET_DEPLOY_MODEL_RESOURCE_ESTIMATE` | ML | Estimate resources |
| `SYSTEM$ML_PLATFORM_CAPABILITIES` | ML | Get platform capabilities |
| `SYSTEM$CREATE_SERVICE` | Core | Create any SPCS service |
| `SYSTEM$GET_SERVICE_STATUS` | Core | Get service status |
| `SYSTEM$GET_SERVICE_LOGS` | Core | Get container logs |
| `SYSTEM$WAIT_FOR_SERVICES` | Core | Wait for service ready |

---

## 18. PRD Recommendations

### 18.1 Design Principles

1. **Code-First**: Developers write code, not infrastructure
2. **Any Language**: Python native, containers for others
3. **Instant Deploy**: < 1 minute from code to endpoint
4. **Zero Infrastructure**: No pools, specs, images to manage
5. **Snowflake Native**: Automatic session, data access, secrets

### 18.2 MVP Scope

| Feature | Priority | Description |
|---------|----------|-------------|
| `@sf.function` | P0 | Serverless function endpoints |
| `@sf.app` | P0 | Streamlit apps |
| `sf.notebook` | P0 | Interactive notebooks |
| Tier-based resources | P0 | Simple resource selection |
| Auto image building | P0 | From Python code |
| `sf.container` | P1 | Custom container support |
| `@sf.job` | P1 | Scheduled batch jobs |
| Auto-scaling | P1 | Based on load |
| `@sf.service` | P2 | Long-running services |
| Multi-language | P2 | Beyond Python |

### 18.3 Success Metrics

| Metric | Target |
|--------|--------|
| Lines of code reduction | 90% |
| Time to first deploy | < 1 minute |
| Concepts to learn | 2 (decorator + deploy) |
| Internal team adoption | 80% within 6 months |

---

## 19. Conclusion

The current SPCS integration requires teams to manage significant infrastructure complexity. A Modal.ai-style serverless interface would:

- **Let developers write code in any language**
- **Provide automatic endpoints** without infrastructure config
- **Handle everything behind the scenes**: compute pools, images, scaling, tokens
- **Reduce complexity by 90%**: from 50+ lines to 5 lines

This would transform SPCS from an infrastructure platform into a true serverless compute platform, dramatically increasing adoption and developer productivity.

---

## Appendix A: Modal.ai Feature Comparison

| Feature | Modal.ai | Proposed Snowflake Serverless |
|---------|----------|------------------------------|
| Python decorators | ✅ | ✅ |
| GPU support | ✅ | ✅ |
| Auto-scaling | ✅ | ✅ |
| Container support | ✅ | ✅ |
| Secrets management | ✅ | ✅ |
| Scheduled jobs | ✅ | ✅ |
| Snowflake data access | ❌ | ✅ (native) |
| Snowflake auth | ❌ | ✅ (native) |
| Enterprise security | Partial | ✅ (Snowflake native) |

---

## Appendix B: Proto Message Definitions

### UDF Streamlit Request

**File:** `/Users/mimam/snowflake-research/snowflake-main/proto/UDFStreamlitRequest.proto`

```protobuf
message StreamlitRequest {
  int64 engine_id = 1;
  int64 app_session_id = 2;
  int64 message_id = 3;

  oneof request {
    StreamlitUserRequest user_request = 10;
    StreamlitControlPlaneRequest control_plane_request = 11;
    GetFileContentsRequest get_file_contents = 12;
  }
}
```

---

*Research conducted: February 2026*
*Repository versions: snowflake-main (main), streamlit-container-runtime (main), cortex-code-skills (main)*
