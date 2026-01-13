# Step 4: Kubernetes Resource Management

## Objective

Build comprehensive Kubernetes resource management capabilities including CRUD operations, templating, validation, and deployment pipelines.

---

## Prerequisites

- Completed [Step 3: Pingora Proxy](step-3-pingora-proxy.md)
- Working k8s-client with API connectivity
- Understanding of Kubernetes resource model

---

## Part A: Kubernetes Resource Model

### Core Resources to Support

| Category | Resources | Priority |
|----------|-----------|----------|
| Workloads | Pod, Deployment, ReplicaSet, StatefulSet, DaemonSet, Job, CronJob | High |
| Networking | Service, Ingress, NetworkPolicy | High |
| Configuration | ConfigMap, Secret | High |
| Storage | PersistentVolume, PersistentVolumeClaim, StorageClass | Medium |
| RBAC | Role, ClusterRole, RoleBinding, ClusterRoleBinding, ServiceAccount | Medium |
| Metadata | Namespace, LimitRange, ResourceQuota | Medium |
| Custom | CustomResourceDefinition, Custom Resources | High |

### Resource Operations Matrix

```text
            │ List │ Get │ Create │ Update │ Patch │ Delete │ Watch │
────────────┼──────┼─────┼────────┼────────┼───────┼────────┼───────┤
Pod         │  ✓   │  ✓  │   ✓    │   ✓    │   ✓   │   ✓    │   ✓   │
Deployment  │  ✓   │  ✓  │   ✓    │   ✓    │   ✓   │   ✓    │   ✓   │
Service     │  ✓   │  ✓  │   ✓    │   ✓    │   ✓   │   ✓    │   ✓   │
ConfigMap   │  ✓   │  ✓  │   ✓    │   ✓    │   ✓   │   ✓    │   ✓   │
Secret      │  ✓   │  ✓  │   ✓    │   ✓    │   ✓   │   ✓    │   ✓   │
Ingress     │  ✓   │  ✓  │   ✓    │   ✓    │   ✓   │   ✓    │   ✓   │
Namespace   │  ✓   │  ✓  │   ✓    │   ─    │   ─   │   ✓    │   ✓   │
CRD         │  ✓   │  ✓  │   ✓    │   ✓    │   ✓   │   ✓    │   ✓   │
```

---

## Part B: Resource Manager Design

### Module Structure

```text
crates/k8s-client/src/
├── resources/
│   ├── mod.rs              # Trait definitions, exports
│   ├── manager.rs          # ResourceManager orchestrator
│   ├── workloads/
│   │   ├── mod.rs
│   │   ├── pod.rs
│   │   ├── deployment.rs
│   │   ├── statefulset.rs
│   │   ├── daemonset.rs
│   │   └── job.rs
│   ├── networking/
│   │   ├── mod.rs
│   │   ├── service.rs
│   │   ├── ingress.rs
│   │   └── network_policy.rs
│   ├── config/
│   │   ├── mod.rs
│   │   ├── configmap.rs
│   │   └── secret.rs
│   └── storage/
│       ├── mod.rs
│       ├── pv.rs
│       └── pvc.rs
└── templates/
    ├── mod.rs
    ├── engine.rs           # Template rendering
    └── builtin/            # Built-in templates
        ├── deployment.yaml
        └── service.yaml
```

### Resource Trait

Define a common interface for all resources:

```text
trait K8sResource {
    type Spec;
    type Status;

    fn api_version() -> &'static str;
    fn kind() -> &'static str;
    fn group() -> &'static str;
    fn plural() -> &'static str;

    fn validate(&self) -> Result<(), ValidationError>;
    fn default_labels(&self) -> BTreeMap<String, String>;
}
```

### ResourceManager API

```text
ResourceManager
├── new(client: K8sClient) -> Self
│
├── Workloads
│   ├── deployments() -> DeploymentApi
│   ├── pods() -> PodApi
│   ├── statefulsets() -> StatefulSetApi
│   └── ...
│
├── Networking
│   ├── services() -> ServiceApi
│   ├── ingresses() -> IngressApi
│   └── ...
│
├── Config
│   ├── configmaps() -> ConfigMapApi
│   └── secrets() -> SecretApi
│
├── Bulk Operations
│   ├── apply_manifest(yaml: &str) -> Result<Vec<ApplyResult>>
│   ├── delete_manifest(yaml: &str) -> Result<Vec<DeleteResult>>
│   └── diff_manifest(yaml: &str) -> Result<DiffReport>
│
└── Discovery
    ├── api_resources() -> Vec<ApiResource>
    └── supports_resource(gvk: &Gvk) -> bool
```

---

## Part C: CRUD Operations

### Create Operation

**Flow:**
```text
1. Validate resource spec
2. Apply default labels/annotations
3. Check if resource exists (optional - for idempotent create)
4. Send create request
5. Wait for creation (optional)
6. Return created resource or error
```

**Validation checks:**
- Required fields present
- Name conforms to DNS-1123 (lowercase, alphanumeric, max 63 chars)
- Labels/annotations valid
- Resource-specific validation (ports, volumes, etc.)

### Update Operation

**Strategies:**

| Strategy | Description | Use Case |
|----------|-------------|----------|
| Replace | Full resource replacement | When you have complete spec |
| Patch (Strategic Merge) | Merge changes into existing | Partial updates |
| Patch (JSON Merge) | RFC 7386 merge | Simple field updates |
| Patch (JSON Patch) | RFC 6902 operations | Precise modifications |

**Conflict handling:**
```text
1. Get current resource with resourceVersion
2. Apply modifications
3. Attempt update with resourceVersion
4. On conflict (409):
   a. Retry with fresh resourceVersion (max 3 times)
   b. Or report conflict to caller
```

### Delete Operation

**Options:**
- `propagationPolicy`: Orphan, Background, Foreground
- `gracePeriodSeconds`: Override pod termination grace period
- `preconditions`: resourceVersion, uid for safety

**Delete patterns:**
```text
Simple delete:
  DELETE /api/v1/namespaces/{ns}/pods/{name}

Delete with cascade:
  DELETE with propagationPolicy=Foreground
  → Wait for dependents to delete first

Delete orphaning dependents:
  DELETE with propagationPolicy=Orphan
  → Dependents remain, owner reference cleared
```

### List & Watch Operations

**List with field selectors:**
```text
List all running pods:
  fieldSelector: status.phase=Running

List pods on specific node:
  fieldSelector: spec.nodeName=worker-1
```

**Watch for changes:**
```text
Watch returns stream of events:
├── Added    - New resource created
├── Modified - Resource updated
├── Deleted  - Resource removed
├── Bookmark - Checkpoint for resume
└── Error    - Watch error (restart watch)
```

**Watch implementation considerations:**
- Handle watch timeout/reconnection
- Track resourceVersion for resume
- Buffer events during processing
- Implement backoff on repeated failures

---

## Part D: Resource Templating

### Template Engine

Build a simple, type-safe templating system:

```text
Template flow:
1. Load template (YAML with placeholders)
2. Validate template structure
3. Apply values (type-safe substitution)
4. Validate rendered output
5. Return K8s resource object
```

### Template Format

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ name }}
  namespace: {{ namespace | default: "default" }}
  labels:
    app.kubernetes.io/name: {{ name }}
    app.kubernetes.io/version: {{ version }}
    app.kubernetes.io/managed-by: k8s-rs
spec:
  replicas: {{ replicas | default: 1 }}
  selector:
    matchLabels:
      app.kubernetes.io/name: {{ name }}
  template:
    metadata:
      labels:
        app.kubernetes.io/name: {{ name }}
    spec:
      containers:
        - name: {{ name }}
          image: {{ image }}:{{ version }}
          ports:
            {{#each ports}}
            - containerPort: {{ this.port }}
              name: {{ this.name | default: "http" }}
            {{/each}}
          {{#if resources}}
          resources:
            requests:
              memory: {{ resources.memory | default: "128Mi" }}
              cpu: {{ resources.cpu | default: "100m" }}
            limits:
              memory: {{ resources.maxMemory | default: resources.memory | default: "256Mi" }}
              cpu: {{ resources.maxCpu | default: resources.cpu | default: "200m" }}
          {{/if}}
```

### Template Values

```yaml
# values.yaml
name: my-app
namespace: production
version: "1.2.3"
image: myregistry.io/my-app
replicas: 3
ports:
  - port: 8080
    name: http
  - port: 9090
    name: metrics
resources:
  memory: "256Mi"
  cpu: "200m"
  maxMemory: "512Mi"
  maxCpu: "500m"
```

### Built-in Templates

Provide ready-to-use templates for common patterns:

| Template | Description |
|----------|-------------|
| `deployment-basic` | Simple deployment with single container |
| `deployment-with-pvc` | Deployment with persistent storage |
| `statefulset-basic` | StatefulSet with headless service |
| `service-clusterip` | Internal ClusterIP service |
| `service-loadbalancer` | External LoadBalancer service |
| `service-nodeport` | NodePort for external access |
| `ingress-basic` | Simple Ingress with TLS |
| `configmap-from-files` | ConfigMap from file contents |
| `secret-tls` | TLS secret from cert/key |

---

## Part E: Resource Validation

### Validation Layers

```text
Layer 1: Structural Validation
├── YAML/JSON syntax valid
├── Required fields present
├── Field types correct
└── API version supported

Layer 2: Semantic Validation
├── Names are DNS-compliant
├── Labels are valid
├── Selectors match
├── Ports are valid (1-65535)
└── Resource quantities parse correctly

Layer 3: Referential Validation
├── ConfigMap/Secret references exist
├── ServiceAccount exists
├── PVC references valid PV
├── Ingress references valid Service
└── NetworkPolicy selectors match pods

Layer 4: Policy Validation (optional)
├── Resource limits set
├── Security context defined
├── No privileged containers
├── Image from allowed registry
└── Custom policy rules
```

### Validation Error Reporting

```text
ValidationResult:
├── valid: bool
├── errors: Vec<ValidationError>
│   ├── path: "spec.containers[0].image"
│   ├── code: "REQUIRED_FIELD"
│   ├── message: "Container image is required"
│   └── suggestion: "Add 'image: <registry>/<name>:<tag>'"
└── warnings: Vec<ValidationWarning>
    ├── path: "spec.containers[0].resources"
    ├── code: "MISSING_LIMITS"
    └── message: "No resource limits set; may cause scheduling issues"
```

---

## Part F: Deployment Pipelines

### Pipeline Stages

```text
Deploy Pipeline:
│
├── 1. Validate
│   ├── Parse manifests
│   ├── Run all validation layers
│   └── Fail fast on errors
│
├── 2. Plan
│   ├── Diff against current state
│   ├── Identify: create, update, delete
│   └── Generate change report
│
├── 3. Pre-deploy Checks
│   ├── Namespace exists (create if not)
│   ├── Secrets/ConfigMaps exist
│   └── RBAC permissions sufficient
│
├── 4. Apply
│   ├── Apply in dependency order
│   ├── Wait for readiness (configurable)
│   └── Rollback on failure (optional)
│
└── 5. Verify
    ├── All resources exist
    ├── Deployments available
    ├── Services have endpoints
    └── Report final status
```

### Dependency Ordering

Apply resources in correct order:

```text
Order 1: Cluster-scoped
├── Namespace
├── ClusterRole
├── ClusterRoleBinding
└── CustomResourceDefinition

Order 2: Configuration
├── ConfigMap
├── Secret
└── ServiceAccount

Order 3: RBAC (namespaced)
├── Role
└── RoleBinding

Order 4: Storage
├── PersistentVolumeClaim
└── StorageClass

Order 5: Networking
├── Service
├── NetworkPolicy
└── Ingress

Order 6: Workloads
├── Deployment
├── StatefulSet
├── DaemonSet
├── Job
└── CronJob
```

### Rollback Strategy

```text
On deployment failure:
│
├── Identify failed resource
├── Check rollback policy:
│   ├── none: Leave as-is, report error
│   ├── failed-only: Rollback failed resource
│   └── all: Rollback all changes in pipeline
│
├── Execute rollback:
│   ├── For create: Delete resource
│   ├── For update: Restore previous version
│   └── For delete: Re-create resource
│
└── Report rollback status
```

---

## Part G: Testing Strategy

### Unit Tests

- Template rendering with various values
- Validation rules for each resource type
- Dependency ordering algorithm
- Diff calculation

### Integration Tests

```text
Test scenarios:
├── Create deployment, verify pods running
├── Update deployment, verify rolling update
├── Delete deployment, verify cleanup
├── Apply multi-resource manifest
├── Validate reference checking (missing ConfigMap)
├── Test rollback on partial failure
└── Watch events during deployment
```

### Test Fixtures

```text
crates/k8s-client/tests/fixtures/
├── valid/
│   ├── deployment-basic.yaml
│   ├── deployment-with-pvc.yaml
│   └── service-complete.yaml
├── invalid/
│   ├── deployment-missing-image.yaml
│   ├── service-invalid-port.yaml
│   └── pod-privileged.yaml
└── multi-resource/
    ├── app-complete.yaml      # ConfigMap + Secret + Deployment + Service
    └── expected-order.json    # Expected apply order
```

---

## Verification Checklist

Before proceeding to Step 5, verify:

| Check | Command | Expected |
|-------|---------|----------|
| Resource APIs work | `cargo test -p k8s-client resources` | All pass |
| Templates render | `cargo run --example render_template` | Valid YAML |
| Validation catches errors | `cargo test validation` | Invalid cases rejected |
| Apply works | `cargo run --example apply_deployment` | Deployment created |
| Watch works | `cargo run --example watch_pods` | Events stream |

---

## Next Steps

Proceed to **[Step 5: Custom Controllers & Operators](step-5-controllers.md)** to:
1. Understand the controller pattern
2. Build reconciliation loops
3. Implement custom controllers
4. Create operators with CRDs

---

## References

- [Kubernetes API Concepts](https://kubernetes.io/docs/reference/using-api/api-concepts/)
- [Kubernetes Object Management](https://kubernetes.io/docs/concepts/overview/working-with-objects/)
- [kube-rs Resource Examples](https://github.com/kube-rs/kube/tree/main/examples)
- [Server-Side Apply](https://kubernetes.io/docs/reference/using-api/server-side-apply/)
