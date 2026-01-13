# Step 5: Custom Controllers & Operators

## Objective

Build Kubernetes controllers and operators using the kube-rs runtime, implementing the reconciliation pattern to manage both built-in and custom resources.

---

## Prerequisites

- Completed [Step 4: Resource Management](step-4-resource-mgmt.md)
- Understanding of Kubernetes controller concepts
- Working knowledge of async Rust

---

## Part A: Controller Fundamentals

### What is a Controller?

A controller is a control loop that watches the shared state of the cluster and makes changes attempting to move the current state towards the desired state.

```text
┌─────────────────────────────────────────────────────────────────┐
│                     Controller Pattern                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌─────────┐         ┌─────────────┐         ┌─────────┐       │
│   │  Watch  │────────▶│  Reconcile  │────────▶│  Write  │       │
│   │ (State) │         │   (Logic)   │         │ (State) │       │
│   └────┬────┘         └──────┬──────┘         └────┬────┘       │
│        │                     │                     │            │
│        │              ┌──────▼──────┐              │            │
│        │              │   Desired   │              │            │
│        └──────────────│     vs      │──────────────┘            │
│                       │   Current   │                           │
│                       └─────────────┘                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Controller vs Operator

| Aspect | Controller | Operator |
|--------|------------|----------|
| Scope | Manages built-in resources | Manages custom resources (CRDs) |
| Complexity | Lower | Higher |
| Domain Knowledge | General | Application-specific |
| Example | Deployment controller | PostgreSQL operator |

### kube-rs Controller Runtime

kube-rs provides `kube-runtime` with:

| Component | Purpose |
|-----------|---------|
| `Controller` | Main orchestrator, manages watchers and reconciliation |
| `watcher` | Watches resources for changes |
| `reflector` | Local cache of watched resources |
| `Store` | In-memory queryable cache |
| `finalizer` | Cleanup before deletion |

---

## Part B: Reconciliation Pattern

### Reconcile Function Signature

```text
async fn reconcile(
    resource: Arc<MyResource>,
    ctx: Arc<Context>
) -> Result<Action, Error>
```

**Return values:**
| Action | Meaning |
|--------|---------|
| `Action::requeue(Duration)` | Recheck after duration |
| `Action::await_change()` | Wait for next watch event |

### Reconciliation Flow

```text
Event received (create/update/delete)
           │
           ▼
   ┌───────────────┐
   │ Get resource  │
   │  from cache   │
   └───────┬───────┘
           │
           ▼
   ┌───────────────┐
   │ Resource      │──────▶ Handle deletion
   │ deleted?      │        (run finalizers)
   └───────┬───────┘
           │ No
           ▼
   ┌───────────────┐
   │  Add/check    │
   │  finalizer    │
   └───────┬───────┘
           │
           ▼
   ┌───────────────┐
   │ Determine     │
   │ desired state │
   └───────┬───────┘
           │
           ▼
   ┌───────────────┐
   │ Get current   │
   │ state         │
   └───────┬───────┘
           │
           ▼
   ┌───────────────┐
   │ Diff desired  │
   │ vs current    │
   └───────┬───────┘
           │
           ▼
   ┌───────────────┐
   │ Apply changes │──────▶ Create/Update/Delete
   │ if needed     │        owned resources
   └───────┬───────┘
           │
           ▼
   ┌───────────────┐
   │ Update status │
   │ on resource   │
   └───────┬───────┘
           │
           ▼
   Return Action::await_change()
   or Action::requeue(duration)
```

### Idempotency Requirements

Reconciliation **must be idempotent**:

```text
✓ DO: Check if action needed before taking it
✓ DO: Use server-side apply for owned resources
✓ DO: Handle partial failures gracefully
✓ DO: Use resource versions for conflict detection

✗ DON'T: Assume reconcile runs exactly once per change
✗ DON'T: Store state outside Kubernetes
✗ DON'T: Perform non-reversible actions without checks
✗ DON'T: Create duplicate resources on retry
```

---

## Part C: k8s-controller Crate Structure

### Module Layout

```text
crates/k8s-controller/src/
├── lib.rs                    # Public API
├── context.rs                # Shared context (client, config, metrics)
├── controller.rs             # Controller builder & runner
├── reconciler.rs             # Base reconciler trait
├── error.rs                  # Controller errors
├── finalizers.rs             # Finalizer utilities
├── conditions.rs             # Status condition helpers
├── controllers/
│   ├── mod.rs
│   ├── deployment_scaler.rs  # Example: auto-scaling controller
│   ├── config_syncer.rs      # Example: ConfigMap sync controller
│   └── pod_labeler.rs        # Example: Pod labeling controller
└── metrics.rs                # Reconciliation metrics
```

### Crate Manifest

```toml
# crates/k8s-controller/Cargo.toml
[package]
name = "k8s-controller"
version.workspace = true
edition.workspace = true
rust-version.workspace = true

[dependencies]
shared = { workspace = true }
k8s-client = { workspace = true }

kube = { workspace = true }
kube-runtime = { version = "0.98", features = ["unstable-runtime"] }
k8s-openapi = { workspace = true }

tokio = { workspace = true }
futures = { workspace = true }
async-trait = { workspace = true }

serde = { workspace = true }
serde_json = { workspace = true }
thiserror = { workspace = true }
tracing = { workspace = true }

# Metrics
prometheus = "0.13"

[dev-dependencies]
tokio-test = { workspace = true }
```

---

## Part D: Building a Controller

### 1. Define Context

```text
Context contains:
├── Kubernetes client
├── Configuration
├── Metrics registry
├── Shared caches (if needed)
└── External service clients
```

### 2. Implement Reconciler

**Steps for reconciler implementation:**

1. **Extract desired state** from the resource spec
2. **Query current state** of owned resources
3. **Calculate diff** between desired and current
4. **Apply changes** in correct order
5. **Update status** with current conditions
6. **Return action** (requeue or await)

### 3. Error Handling

```text
ReconcileError variants:
├── TemporaryError     → Requeue with backoff
│   ├── API timeout
│   ├── Rate limited
│   └── Dependent not ready
│
├── PermanentError     → Update status, don't requeue
│   ├── Invalid spec
│   ├── Missing required field
│   └── Unsupported configuration
│
└── FatalError         → Log, alert, may need manual intervention
    ├── RBAC denied
    ├── CRD not installed
    └── Cluster unreachable
```

### 4. Status Conditions

Use standard condition format:

```text
Status:
  conditions:
    - type: Ready
      status: "True"
      reason: "AllPodsRunning"
      message: "All 3 pods are running and healthy"
      lastTransitionTime: "2024-01-15T10:30:00Z"

    - type: Progressing
      status: "False"
      reason: "NewReplicaSetAvailable"
      message: "ReplicaSet my-app-abc123 has 3 available replicas"
      lastTransitionTime: "2024-01-15T10:29:00Z"
```

---

## Part E: Custom Resource Definitions (CRDs)

### k8s-operator Crate Structure

```text
crates/k8s-operator/src/
├── lib.rs                    # Public API
├── crd.rs                    # CRD types with kube::CustomResource
├── controller.rs             # Operator controller
├── reconciler.rs             # CRD reconciler
├── finalizers.rs             # Cleanup logic
├── status.rs                 # Status types
├── validation.rs             # Admission validation
└── crds/
    ├── mod.rs
    ├── proxy_config.rs       # ProxyConfig CRD
    └── backend_pool.rs       # BackendPool CRD
```

### CRD Definition with kube-rs

**Derive CRD from Rust struct:**

```text
#[derive(CustomResource, ...)]
#[kube(
    group = "k8s-rs.io",
    version = "v1",
    kind = "ProxyConfig",
    namespaced,
    status = "ProxyConfigStatus",
    printcolumn = r#"{"name":"Backends","type":"integer","jsonPath":".status.backendsReady"}"#
)]
struct ProxyConfigSpec {
    // Spec fields
}

struct ProxyConfigStatus {
    // Status fields
    conditions: Vec<Condition>,
    backends_ready: i32,
    backends_total: i32,
}
```

### CRD Lifecycle

```text
1. Define Rust struct with #[derive(CustomResource)]
2. Generate CRD YAML: ProxyConfig::crd()
3. Apply CRD to cluster (before operator starts)
4. Operator watches ProxyConfig resources
5. Reconcile creates/updates owned resources
6. Status reflects current state
```

### Example CRDs for This Project

**ProxyConfig CRD:**
```yaml
apiVersion: k8s-rs.io/v1
kind: ProxyConfig
metadata:
  name: my-proxy
  namespace: default
spec:
  # Proxy settings
  listenPort: 8080
  tls:
    enabled: true
    secretRef: proxy-tls-cert

  # Backend selection
  backendSelector:
    matchLabels:
      app: my-backend

  # Load balancing
  loadBalancing:
    algorithm: roundRobin
    healthCheck:
      path: /healthz
      intervalSeconds: 10
```

**BackendPool CRD:**
```yaml
apiVersion: k8s-rs.io/v1
kind: BackendPool
metadata:
  name: my-backend-pool
  namespace: default
spec:
  # Static backends
  backends:
    - address: 10.0.0.1:8080
      weight: 1
    - address: 10.0.0.2:8080
      weight: 2

  # Or discover from Service
  serviceRef:
    name: my-service
    port: http

  healthCheck:
    enabled: true
    path: /health
    intervalSeconds: 5
    timeoutSeconds: 2
    unhealthyThreshold: 3
```

---

## Part F: Finalizers

### Purpose

Finalizers ensure cleanup before resource deletion:

```text
Without finalizer:
  kubectl delete myresource → Resource deleted immediately
                             → Owned resources orphaned!

With finalizer:
  kubectl delete myresource
       │
       ▼
  Resource marked for deletion (deletionTimestamp set)
       │
       ▼
  Reconciler detects deletion
       │
       ▼
  Cleanup owned resources
       │
       ▼
  Remove finalizer
       │
       ▼
  Resource actually deleted
```

### Finalizer Implementation

```text
Finalizer workflow:

On Reconcile:
  1. Check if resource has deletionTimestamp
     ├── No: Ensure finalizer present, continue reconcile
     └── Yes: Run cleanup

  2. Cleanup steps:
     ├── Delete owned Deployments
     ├── Delete owned Services
     ├── Clean up external resources
     └── Remove finalizer

  3. After finalizer removed:
     └── Kubernetes deletes the resource
```

### Owned Resources

Use owner references for automatic garbage collection:

```text
Parent resource (ProxyConfig)
       │
       │ ownerReferences
       ▼
┌──────────────────┐
│ Owned Deployment │──▶ Automatically deleted
└──────────────────┘    when parent deleted
       │                (with foreground deletion)
       │ ownerReferences
       ▼
┌──────────────────┐
│   Owned Service  │──▶ Automatically deleted
└──────────────────┘
```

---

## Part G: Leader Election

### Why Leader Election?

For high availability, run multiple controller replicas. Only one should be active:

```text
┌─────────────────────────────────────────────────────┐
│                Multiple Replicas                     │
├─────────────────────────────────────────────────────┤
│                                                      │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐          │
│  │ Replica  │  │ Replica  │  │ Replica  │          │
│  │    1     │  │    2     │  │    3     │          │
│  │ (LEADER) │  │(standby) │  │(standby) │          │
│  └────┬─────┘  └──────────┘  └──────────┘          │
│       │                                             │
│       │ Only leader reconciles                      │
│       ▼                                             │
│  ┌──────────────────────────────────────┐          │
│  │        Kubernetes API Server         │          │
│  └──────────────────────────────────────┘          │
│                                                      │
└─────────────────────────────────────────────────────┘
```

### Leader Election Configuration

```yaml
# config.yaml - controller section
controller:
  leader_election:
    enabled: true
    # Lease resource name
    lease_name: "k8s-rs-controller"
    # Namespace for lease
    lease_namespace: "kube-system"
    # Lease duration
    lease_duration_secs: 15
    # Time to renew before expiry
    renew_deadline_secs: 10
    # Retry period for acquiring lease
    retry_period_secs: 2
```

---

## Part H: Testing Controllers

### Unit Testing Reconciler Logic

```text
Test approach:
1. Mock the Kubernetes client
2. Set up initial state (resource + owned resources)
3. Call reconciler
4. Verify actions taken (API calls)
5. Verify returned Action
```

### Integration Testing

```text
Integration test flow:
1. Set up test namespace
2. Apply CRD
3. Create test resource
4. Run controller
5. Wait for reconciliation
6. Verify owned resources created
7. Modify resource
8. Verify owned resources updated
9. Delete resource
10. Verify cleanup
11. Teardown test namespace
```

### Test Utilities

```text
crates/k8s-controller/tests/
├── common/
│   ├── mod.rs
│   ├── test_context.rs     # Test context setup
│   └── assertions.rs       # Custom assertions
├── unit/
│   ├── reconciler_test.rs
│   └── finalizer_test.rs
└── integration/
    ├── controller_test.rs
    └── fixtures/
        └── test_resources.yaml
```

---

## Part I: controller-service

### Service Structure

```text
services/controller-service/
├── Cargo.toml
├── src/
│   └── main.rs           # Entry point
├── Dockerfile
└── k8s/
    ├── deployment.yaml   # Controller deployment
    ├── rbac.yaml         # ServiceAccount, Role, RoleBinding
    ├── configmap.yaml    # Controller configuration
    └── crd.yaml          # Custom Resource Definitions
```

### RBAC Requirements

The controller needs permissions to:

```yaml
# k8s/rbac.yaml (example rules)
rules:
  # Watch and manage our CRDs
  - apiGroups: ["k8s-rs.io"]
    resources: ["proxyconfigs", "backendpools"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: ["k8s-rs.io"]
    resources: ["proxyconfigs/status", "backendpools/status"]
    verbs: ["get", "update", "patch"]

  # Manage owned resources
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  - apiGroups: [""]
    resources: ["services", "configmaps", "secrets"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

  # Leader election
  - apiGroups: ["coordination.k8s.io"]
    resources: ["leases"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]

  # Events
  - apiGroups: [""]
    resources: ["events"]
    verbs: ["create", "patch"]
```

---

## Verification Checklist

Before proceeding to Step 6, verify:

| Check | Command | Expected |
|-------|---------|----------|
| Controller builds | `cargo build -p k8s-controller` | Success |
| Tests pass | `cargo test -p k8s-controller` | All green |
| CRD generates | `cargo run --example gen_crd` | Valid CRD YAML |
| CRD applies | `kubectl apply -f crd.yaml` | CRD created |
| Controller runs | `cargo run -p controller-service` | Watching... |
| Reconciliation works | Create custom resource | Owned resources created |
| Cleanup works | Delete custom resource | Owned resources deleted |

---

## Next Steps

Proceed to **[Step 6: Ingress Controller with Pingora](step-6-ingress.md)** to:
1. Build a full Ingress controller
2. Integrate Pingora proxy with Kubernetes Ingress resources
3. Handle TLS termination
4. Implement path-based routing

---

## References

- [kube-rs Controller Guide](https://kube.rs/controllers/intro/)
- [kube-rs Controller Examples](https://github.com/kube-rs/controller-rs)
- [Kubernetes Controller Concepts](https://kubernetes.io/docs/concepts/architecture/controller/)
- [Kubernetes Operator Pattern](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/)
- [Custom Resources](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/)
