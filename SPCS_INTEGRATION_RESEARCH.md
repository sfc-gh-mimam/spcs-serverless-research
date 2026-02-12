# SPCS Integration Research: How Internal Snowflake Services Access SnowServices

## Executive Summary

This document analyzes how internal Snowflake services (Notebooks vNext, Streamlit/SiS vNext, Cortex Code Agent, and Snowflake Intelligence Agent) integrate with Snowpark Container Services (SPCS). The goal is to inform the design of a **Modal.ai-style serverless interface** that lets developers write code in any language and get endpoints automatically—with the platform handling all infrastructure behind the scenes.

---

## 1. Current Integration Patterns Overview

### 1.1 Service Categories

| Service | Integration Type | Complexity | Key Interface |
|---------|-----------------|------------|---------------|
| Notebooks vNext | Direct SPCS (ServiceSpec) | High | SYSTEM$NOTEBOOKS_VNEXT_CREATE_* |
| Streamlit/SiS vNext | Direct SPCS (ServiceSpec) | High | SYSTEM$STREAMLIT_BOOTSTRAP |
| Cortex Search | Managed Service (SQL DDL) | Medium | CREATE CORTEX SEARCH SERVICE |
| Cortex Agent | Versioned Stage + Orchestrator | Medium | CREATE AGENT + gRPC |
| Intelligence Agent | Versioned Stage + Mapping | Medium | CREATE SNOWFLAKE INTELLIGENCE |

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

## 3. Streamlit/SiS vNext Integration

### 3.1 Service Bootstrap

**File:** `/Users/mimam/snowflake-research/snowflake-main/Snowfort/tests/t_streamlit/test_streamlit_on_spcs_v2.py`

```python
@CreateAccount.with_args(parameters={
    "ENABLE_SNOWSERVICES_USER_FACING_FEATURES": True,
    "ENABLE_STREAMLIT_SPCS_RUNTIME_V2": True,
})
def test_streamlit_spcs_v2():
    hybrid_executor.execute_query(
        f"SELECT SYSTEM$STREAMLIT_BOOTSTRAP('{streamlit.name}', 'session-uuid', false)"
    )
```

### 3.2 Token Authentication Pattern

**File:** `/Users/mimam/snowflake-research/streamlit-container-runtime/internal/snowflake/config.go`

```go
const TokenFilePath = "/snowflake/session/token"  // OAuth token location

// Token refresh logic - valid for 60 minutes
func getSession() *Session {
    if session.connection.is_valid() {
        return session
    }
    return createNewSession()
}
```

### 3.3 Key Friction Points

1. **Feature Flags**: Multiple flags required (`ENABLE_STREAMLIT_SPCS_RUNTIME_V2`)
2. **Token Expiration**: 60-minute token requires explicit refresh handling
3. **Cold Start**: Service startup time affects user experience
4. **Resource Sharing**: No built-in multi-tenant support

---

## 4. Cortex Services Integration

### 4.1 Cortex Search Service

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

### 4.2 Cortex Agent Orchestration

**File:** `/Users/mimam/snowflake-research/snowflake-main/GlobalServices/modules/cortex/cortex-protos/src/main/protobuf/cortex_chat_service.proto`

```protobuf
service Orchestrator {
  rpc Chat (ChatRequest) returns (stream ChatResponse)
  rpc Agent (AgentRequest) returns (stream AgentResponse)
  rpc QuerySemanticModel(SemanticModelQueryRequest) returns (stream SemanticModelQueryResponse)
}

message AgentRequest {
  string model = 1;
  string messages = 4;
  string tools = 5;
  map<string, SearchService> search_services = 8;
  AnalystParams analyst_params = 12;
}
```

---

## 5. Snowflake Intelligence Agent Integration

### 5.1 Entity Mapping Architecture

**File:** `/Users/mimam/snowflake-research/snowflake-main/GlobalServices/src/main/java/com/snowflake/metadata/dictionary/snowflakeintelligence/SnowflakeIntelligenceEntityMapping.java`

```java
public class SnowflakeIntelligenceEntityMapping {
    public static SnowflakeIntelligenceEntityMapping create(
        SnowflakeIntelligence si, CortexAgent agent) {
        return new SnowflakeIntelligenceEntityMapping(
            si.getAccountId(), si.getId(), Domain.CORTEX_AGENT, agent.getId());
    }

    public static void attachAgentToSnowflakeIntelligenceInTransaction(
        DPOTransaction tx, SnowflakeIntelligence si, CortexAgent agent) {
        SnowflakeIntelligenceEntityMapping mapping = create(si, agent);
        mapping.persist(tx);
    }
}
```

---

## 6. ServiceSpec Deep Dive

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

## 7. Compute Pool Management

### 7.1 Instance Families

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

## 8. Friction Points Summary

### 8.1 High Severity (Must Solve)

| Friction | Current State | Impact |
|----------|--------------|--------|
| **ServiceSpec Complexity** | 50+ lines YAML | Teams avoid SPCS |
| **Compute Pool Management** | Manual DDL, sizing | Delays deployment |
| **Image Management** | Build/push/version | DevOps overhead |

### 8.2 Medium Severity

| Friction | Current State | Impact |
|----------|--------------|--------|
| **Token Refresh** | 60-min expiry | Session failures |
| **Resource Estimation** | Guesswork | Over/under provisioning |
| **Cold Start** | Minutes | Poor UX |

---

## 9. Vision: Modal.ai-Style Serverless Interface

### 9.1 What is Modal.ai?

Modal.ai provides a developer experience where you:
1. **Write code in any language** (Python, etc.)
2. **Decorate functions** with resource requirements
3. **Deploy instantly** - platform handles everything
4. **Get endpoints automatically** - no infrastructure config

### 9.2 Proposed Snowflake Serverless Interface

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

### 9.3 Multi-Language Support

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

### 9.4 Resource Tiers (Simplified)

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

## 10. Architecture: What Platform Handles

### 10.1 Developer Provides

| Input | Required | Example |
|-------|----------|---------|
| **Code** | Yes | Python function, container, stage path |
| **Resources** | Yes | Tier or cpu/memory/gpu |
| **Secrets** | Optional | `["api_key", "db_password"]` |
| **Schedule** | Optional | Cron expression for jobs |

### 10.2 Platform Handles (Invisible to Developer)

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

### 10.3 Behind the Scenes

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

## 11. API Design

### 11.1 Core Decorators

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

### 11.2 Deployment Methods

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

### 11.3 Endpoint Operations

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

### 11.4 Resource Specification Options

```python
# Option 1: Explicit resources
@sf.function(cpu=2, memory="4GB", gpu="A10G")

# Option 2: Tier-based
@sf.function(tier="gpu-small")

# Option 3: Auto (platform decides based on code analysis)
@sf.function(tier="auto")
```

---

## 12. Comparison: Current vs Proposed

### 12.1 Current SPCS (50+ lines)

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

### 12.2 Proposed Serverless (5 lines)

```python
import snowflake.serverless as sf

@sf.function(cpu=2, memory="4GB")
def my_function(data):
    return process(data)

endpoint = sf.deploy(my_function)
# Done. endpoint.url is ready to use.
```

### 12.3 Reduction

| Metric | Current | Proposed | Reduction |
|--------|---------|----------|-----------|
| Lines of code | 50+ | 5 | 90% |
| Concepts to learn | 8+ | 2 | 75% |
| Time to deploy | 30+ min | < 1 min | 97% |
| Infrastructure knowledge | High | None | 100% |

---

## 13. Implementation Considerations

### 13.1 For Internal Teams (Notebooks, Streamlit, Cortex)

The serverless interface can be adopted incrementally:

1. **Phase 1**: New projects use `sf.deploy()`
2. **Phase 2**: Existing services migrate to decorators
3. **Phase 3**: Deprecate direct ServiceSpec usage

### 13.2 Backward Compatibility

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

### 13.3 Snowflake Integration

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

## 14. Key Files Reference

### 14.1 Core SPCS Infrastructure

| File | Purpose |
|------|---------|
| `Snowfort/snowfort/testlib/snowservices/service_spec.py` | ServiceSpec class definition |
| `Snowfort/tests/snowservices/lib/compute_pool.py` | Compute pool management |
| `GlobalServices/modules/snowapi/snowapi-codegen/.../compute-pool.yaml` | REST API spec |

### 14.2 Notebooks vNext

| File | Purpose |
|------|---------|
| `Snowfort/tests/notebooks_vnext/api/notebook_service_api.py` | Service creation API |
| `Snowfort/tests/notebooks_vnext/test_notebook_managed_service.py` | Integration tests |

### 14.3 Streamlit/SiS

| File | Purpose |
|------|---------|
| `streamlit-container-runtime/internal/snowflake/config.go` | Token auth |
| `streamlit-container-runtime/runtime-container/scripts/streamlit-runner.py` | Session management |
| `Snowfort/tests/t_streamlit/test_streamlit_on_spcs_v2.py` | SPCS v2 tests |

### 14.4 Cortex Services

| File | Purpose |
|------|---------|
| `GlobalServices/modules/cortex/cortex-protos/.../cortex_chat_service.proto` | Orchestrator |
| `Snowfort/snowfort/testlib/native_apps/api/cortex_search_service_api.py` | Search API |

### 14.5 Intelligence Agent

| File | Purpose |
|------|---------|
| `GlobalServices/.../ExecCreateSnowflakeIntelligence.java` | SI creation |
| `GlobalServices/.../SnowflakeIntelligenceEntityMapping.java` | SI-Agent mapping |

---

## 15. PRD Recommendations

### 15.1 Design Principles

1. **Code-First**: Developers write code, not infrastructure
2. **Any Language**: Python native, containers for others
3. **Instant Deploy**: < 1 minute from code to endpoint
4. **Zero Infrastructure**: No pools, specs, images to manage
5. **Snowflake Native**: Automatic session, data access, secrets

### 15.2 MVP Scope

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

### 15.3 Success Metrics

| Metric | Target |
|--------|--------|
| Lines of code reduction | 90% |
| Time to first deploy | < 1 minute |
| Concepts to learn | 2 (decorator + deploy) |
| Internal team adoption | 80% within 6 months |

---

## 16. Conclusion

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
