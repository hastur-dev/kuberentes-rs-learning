# Step 2: Setting Up Kubernetes Client with kube-rs

## Objective

Set up the Cargo workspace, create the `k8s-client` crate, and establish connectivity to a Kubernetes cluster using kube-rs.

---

## Prerequisites

- Completed [Step 1: Foundation](step-1-foundation.md)
- Local Kubernetes cluster running (kind or minikube)
- Valid kubeconfig at `~/.kube/config`

---

## Part A: Workspace Setup

### 1. Convert to Cargo Workspace

Transform the root `Cargo.toml` into a workspace manifest:

```toml
# Cargo.toml (workspace root)
[workspace]
resolver = "2"
members = [
    "crates/shared",
    "crates/k8s-client",
    "crates/pingora-proxy",
    "crates/k8s-controller",
    "crates/k8s-operator",
    "services/proxy-service",
    "services/controller-service",
]

[workspace.package]
version = "0.1.0"
edition = "2024"
rust-version = "1.81"
license = "MIT OR Apache-2.0"
repository = "https://github.com/hastur-dev/kuberentes-rs-learning"

[workspace.dependencies]
# Async runtime
tokio = { version = "1.43", features = ["full"] }

# Kubernetes
kube = { version = "0.98", features = ["runtime", "client", "derive"] }
k8s-openapi = { version = "0.23", features = ["latest"] }

# Serialization
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
serde_yaml = "0.9"

# Error handling
thiserror = "2.0"
anyhow = "1.0"

# Logging & tracing
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter", "json"] }

# Configuration
config = "0.14"

# Testing
tokio-test = "0.4"

# Internal crates
shared = { path = "crates/shared" }
k8s-client = { path = "crates/k8s-client" }
pingora-proxy = { path = "crates/pingora-proxy" }
k8s-controller = { path = "crates/k8s-controller" }
k8s-operator = { path = "crates/k8s-operator" }

[workspace.lints.rust]
unsafe_code = "deny"
missing_docs = "warn"

[workspace.lints.clippy]
all = "warn"
pedantic = "warn"
nursery = "warn"
```

### 2. Create Directory Structure

```bash
# POSIX
mkdir -p crates/{shared,k8s-client,pingora-proxy,k8s-controller,k8s-operator}/src
mkdir -p services/{proxy-service,controller-service}/src
mkdir -p scripts

# Windows PowerShell
New-Item -ItemType Directory -Force -Path crates/shared/src, crates/k8s-client/src, crates/pingora-proxy/src, crates/k8s-controller/src, crates/k8s-operator/src
New-Item -ItemType Directory -Force -Path services/proxy-service/src, services/controller-service/src
New-Item -ItemType Directory -Force -Path scripts
```

---

## Part B: Shared Crate

### 1. Shared Crate Manifest

```toml
# crates/shared/Cargo.toml
[package]
name = "shared"
version.workspace = true
edition.workspace = true
rust-version.workspace = true

[dependencies]
serde = { workspace = true }
serde_yaml = { workspace = true }
thiserror = { workspace = true }
tracing = { workspace = true }

[dev-dependencies]
tokio-test = { workspace = true }
```

### 2. Shared Types

The shared crate should contain:

- **Configuration structures** - Typed config loaded from `config.yaml`
- **Common error types** - Unified error handling across crates
- **Utility functions** - Validation, parsing helpers
- **Constants** - Labels, annotations, defaults

**Key types to implement:**

```text
crates/shared/src/
├── lib.rs           # Re-exports
├── config.rs        # AppConfig struct with validation
├── error.rs         # SharedError enum with thiserror
└── constants.rs     # K8s labels, defaults
```

**Configuration validation requirements:**
- All paths must exist if specified
- Ports must be in valid range (1-65535)
- Log level must be valid variant
- Fail fast with clear error messages

---

## Part C: Kubernetes Client Crate

### 1. k8s-client Manifest

```toml
# crates/k8s-client/Cargo.toml
[package]
name = "k8s-client"
version.workspace = true
edition.workspace = true
rust-version.workspace = true

[dependencies]
shared = { workspace = true }
kube = { workspace = true }
k8s-openapi = { workspace = true }
tokio = { workspace = true }
serde = { workspace = true }
serde_json = { workspace = true }
thiserror = { workspace = true }
tracing = { workspace = true }

[dev-dependencies]
tokio-test = { workspace = true }
```

### 2. Client Module Structure

```text
crates/k8s-client/src/
├── lib.rs           # Public API re-exports
├── client.rs        # K8sClient wrapper struct
├── error.rs         # ClientError enum
├── resources/
│   ├── mod.rs       # Resource module exports
│   ├── pods.rs      # Pod operations
│   ├── services.rs  # Service operations
│   ├── deployments.rs
│   ├── namespaces.rs
│   └── configmaps.rs
└── discovery.rs     # API discovery utilities
```

### 3. K8sClient Design

The client wrapper should provide:

**Initialization:**
- `K8sClient::new()` - Use default kubeconfig discovery
- `K8sClient::from_kubeconfig(path)` - Explicit kubeconfig path
- `K8sClient::in_cluster()` - In-cluster ServiceAccount auth

**Core operations per resource type:**
- `list(namespace, label_selector)` - List with optional filtering
- `get(namespace, name)` - Get single resource
- `create(namespace, resource)` - Create new resource
- `update(namespace, resource)` - Update existing resource
- `delete(namespace, name)` - Delete resource
- `watch(namespace, label_selector)` - Watch for changes (returns Stream)

**Health & discovery:**
- `health_check()` - Verify API server connectivity
- `api_versions()` - List supported API versions
- `api_resources(group)` - List resources in API group

---

## Part D: Implementing the Client

### 1. Error Handling Strategy

```text
ClientError variants:
├── ConfigError        - Kubeconfig loading failed
├── ConnectionError    - Cannot reach API server
├── AuthError          - Authentication/authorization failed
├── NotFound           - Resource doesn't exist
├── AlreadyExists      - Resource already exists (on create)
├── Conflict           - Version conflict (on update)
├── ValidationError    - Invalid resource spec
├── Timeout            - Operation timed out
└── Internal           - Unexpected kube-rs error
```

### 2. Connection Patterns

**Pattern: Lazy Connection with Health Check**

```text
1. Create client (stores config, doesn't connect)
2. On first operation OR explicit health_check():
   - Attempt connection
   - Cache connection state
   - Return health status
3. On connection failure:
   - Log error with context
   - Return typed error
   - Allow retry
```

**Pattern: Retry with Exponential Backoff**

For transient failures, implement retry:
- Initial delay: 100ms
- Max delay: 30s
- Max attempts: 5
- Jitter: +/- 10%

### 3. Testing Strategy

**Unit tests (mock kube client):**
- Resource serialization/deserialization
- Error mapping
- Label selector building

**Integration tests (requires cluster):**
- Create namespace for tests
- CRUD operations on pods, services
- Watch functionality
- Clean up on test completion

**Test fixtures:**
```text
crates/k8s-client/tests/
├── integration/
│   ├── mod.rs
│   ├── pods_test.rs
│   └── services_test.rs
└── fixtures/
    ├── pod.yaml
    └── service.yaml
```

---

## Part E: Local Cluster Setup

### 1. Create kind Cluster

```yaml
# kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
name: k8s-rs-dev
nodes:
  - role: control-plane
    kubeadmConfigPatches:
      - |
        kind: InitConfiguration
        nodeRegistration:
          kubeletExtraArgs:
            node-labels: "ingress-ready=true"
    extraPortMappings:
      - containerPort: 80
        hostPort: 80
        protocol: TCP
      - containerPort: 443
        hostPort: 443
        protocol: TCP
      - containerPort: 8080
        hostPort: 8080
        protocol: TCP
  - role: worker
  - role: worker
```

```bash
# Create cluster
kind create cluster --config kind-config.yaml

# Verify
kubectl cluster-info
kubectl get nodes
```

### 2. Verify Connectivity

After implementing the k8s-client, run:

```bash
# Build and run example
cargo run --example health_check

# Expected output:
# INFO  k8s_client > Connecting to Kubernetes API server...
# INFO  k8s_client > Connected successfully
# INFO  k8s_client > Server version: v1.31.0
# INFO  k8s_client > Available API groups: 25
```

---

## Verification Checklist

Before proceeding to Step 3, verify:

| Check | Command | Expected |
|-------|---------|----------|
| Workspace builds | `cargo build` | Success, no errors |
| Tests pass | `cargo test` | All tests green |
| Clippy clean | `cargo clippy -- -D warnings` | No warnings |
| Format check | `cargo fmt --check` | No changes needed |
| Kind cluster up | `kubectl get nodes` | 3 nodes Ready |
| Client connects | `cargo run --example health_check` | Connected successfully |

---

## Common Issues

### Issue: "error: no matching package named `k8s-openapi`"

**Cause:** Feature flag mismatch with Kubernetes version.

**Solution:** Ensure k8s-openapi feature matches your cluster version:
```toml
# For K8s 1.31.x
k8s-openapi = { version = "0.23", features = ["v1_31"] }
```

### Issue: "connection refused" on API server

**Cause:** Kind cluster not running or kubeconfig not set.

**Solution:**
```bash
# Check cluster status
kind get clusters
docker ps | grep kind

# Recreate if needed
kind delete cluster --name k8s-rs-dev
kind create cluster --config kind-config.yaml
```

### Issue: "Forbidden" errors on API calls

**Cause:** RBAC permissions insufficient.

**Solution:** For local dev, ensure your kubeconfig has cluster-admin. For in-cluster, see Step 5 for RBAC setup.

---

## Next Steps

Proceed to **[Step 3: Building the Pingora Proxy Layer](step-3-pingora-proxy.md)** to:
1. Add Pingora dependencies
2. Create the pingora-proxy crate
3. Implement basic HTTP proxying
4. Connect to Kubernetes for upstream discovery

---

## References

- [kube-rs Getting Started](https://kube.rs/getting-started/)
- [kube-rs API Documentation](https://docs.rs/kube/latest/kube/)
- [k8s-openapi Docs](https://docs.rs/k8s-openapi/latest/k8s_openapi/)
- [kind Quick Start](https://kind.sigs.k8s.io/docs/user/quick-start/)
