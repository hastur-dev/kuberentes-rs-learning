# Step 1: Project Foundation & Architecture Overview

## Objective

Transform this repository into a comprehensive Rust-based Kubernetes deployment and management platform, replacing Nginx with Cloudflare's Pingora for the proxy/load-balancing layer.

---

## Architecture Overview

```
+-----------------------------------------------------------------------------------+
|                              kubernetes-rs-learning                                |
+-----------------------------------------------------------------------------------+
|                                                                                   |
|  +------------------+    +------------------+    +------------------+              |
|  |   pingora-proxy  |    |   k8s-controller |    |   k8s-operator   |              |
|  |   (Load Balancer)|    |   (Core Mgmt)    |    |   (Custom CRDs)  |              |
|  +--------+---------+    +--------+---------+    +--------+---------+              |
|           |                       |                       |                        |
|           +----------+------------+-----------+-----------+                        |
|                      |                        |                                    |
|           +----------v------------+  +--------v---------+                          |
|           |     k8s-client        |  |   config-store   |                          |
|           |     (kube-rs)         |  |   (YAML/TOML)    |                          |
|           +----------+------------+  +------------------+                          |
|                      |                                                             |
|           +----------v------------+                                                |
|           |   Kubernetes API      |                                                |
|           |   Server (cluster)    |                                                |
|           +-----------------------+                                                |
+-----------------------------------------------------------------------------------+
```

---

## Project Structure (Target)

```
kubernetes-rs-learning/
├── Cargo.toml                    # Workspace root
├── config.yaml                   # Runtime configuration
├── config.yaml.example           # Template with placeholders
├── docs/
│   ├── step-1-foundation.md      # This file
│   ├── step-2-k8s-client.md
│   ├── step-3-pingora-proxy.md
│   ├── step-4-resource-mgmt.md
│   ├── step-5-controllers.md
│   ├── step-6-ingress.md
│   ├── step-7-observability.md
│   └── step-8-deployment.md
├── crates/
│   ├── k8s-client/               # Kubernetes API client wrapper
│   │   ├── Cargo.toml
│   │   ├── src/lib.rs
│   │   └── tests/
│   ├── pingora-proxy/            # Pingora-based proxy/LB
│   │   ├── Cargo.toml
│   │   ├── src/lib.rs
│   │   └── tests/
│   ├── k8s-controller/           # Core Kubernetes controller
│   │   ├── Cargo.toml
│   │   ├── src/lib.rs
│   │   └── tests/
│   ├── k8s-operator/             # Custom Resource Definitions
│   │   ├── Cargo.toml
│   │   ├── src/lib.rs
│   │   └── tests/
│   └── shared/                   # Shared types & utilities
│       ├── Cargo.toml
│       ├── src/lib.rs
│       └── tests/
├── services/
│   ├── proxy-service/            # Deployable Pingora service
│   │   ├── Cargo.toml
│   │   ├── src/main.rs
│   │   ├── Dockerfile
│   │   └── k8s/
│   │       ├── deployment.yaml
│   │       └── service.yaml
│   └── controller-service/       # Deployable controller
│       ├── Cargo.toml
│       ├── src/main.rs
│       ├── Dockerfile
│       └── k8s/
│           ├── deployment.yaml
│           ├── rbac.yaml
│           └── service.yaml
├── scripts/
│   ├── run_all.sh
│   └── run_all.ps1
└── README.md
```

---

## Key Technologies

### 1. kube-rs (Kubernetes Client)

The primary Rust client for interacting with Kubernetes APIs.

**Why kube-rs?**
- Native async/await support with Tokio
- Type-safe API interactions
- Built-in support for Custom Resource Definitions (CRDs)
- Controller runtime for building operators
- Active maintenance and community

**Key crates:**
- `kube` - Core client library
- `kube-runtime` - Controller runtime, watchers, reflectors
- `k8s-openapi` - Generated Kubernetes API types

### 2. Pingora (Proxy/Load Balancer)

Cloudflare's battle-tested Rust framework replacing Nginx.

**Why Pingora over Nginx?**
- Written in Rust (memory safety, no C vulnerabilities)
- Native async I/O with Tokio
- Lower memory footprint
- Programmable via Rust (not Lua/config files)
- HTTP/2 and HTTP/3 (QUIC) support
- Connection pooling and multiplexing
- Handles millions of requests/second at Cloudflare

**Key components:**
- `pingora-core` - Core proxy framework
- `pingora-proxy` - HTTP proxy implementation
- `pingora-load-balancing` - Load balancing algorithms
- `pingora-cache` - HTTP caching layer

---

## Prerequisites

Before proceeding to Step 2, ensure you have:

### Development Environment

| Tool | Minimum Version | Purpose |
|------|-----------------|---------|
| Rust | 1.81+ | Language toolchain |
| Docker | 24.0+ | Container builds |
| kubectl | 1.28+ | Kubernetes CLI |
| kind/minikube | Latest | Local K8s cluster |
| openssl | 3.0+ | TLS certificate generation |

### Kubernetes Cluster Access

You'll need access to a Kubernetes cluster. Options:
1. **kind** - Recommended for local development
2. **minikube** - Alternative local cluster
3. **Cloud provider** - GKE/EKS/AKS for production testing

### Installation Commands

**Windows (PowerShell):**
```powershell
# Rust
winget install Rustlang.Rustup
rustup default stable
rustup update

# Docker Desktop (includes kubectl)
winget install Docker.DockerDesktop

# kind
winget install Kubernetes.kind

# Verify
rustc --version
cargo --version
docker --version
kubectl version --client
kind --version
```

**Linux/macOS (bash):**
```bash
# Rust
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
rustup default stable

# Docker (Linux)
curl -fsSL https://get.docker.com | sh

# kind
go install sigs.k8s.io/kind@latest
# or
brew install kind  # macOS

# Verify
rustc --version
cargo --version
docker --version
kubectl version --client
kind --version
```

---

## Configuration Strategy

All secrets and configuration will be loaded from `config.yaml`:

```yaml
# config.yaml.example
kubernetes:
  # Leave empty to use in-cluster config or ~/.kube/config
  kubeconfig_path: ""
  # Namespace to operate in (empty = all namespaces)
  namespace: "default"
  # Context to use (empty = current context)
  context: ""

pingora:
  # Proxy listen address
  listen_addr: "0.0.0.0:8080"
  # Admin API address
  admin_addr: "127.0.0.1:9090"
  # TLS settings
  tls:
    enabled: false
    cert_path: ""
    key_path: ""
  # Upstream discovery from Kubernetes Services
  upstream_discovery:
    enabled: true
    label_selector: "app.kubernetes.io/managed-by=k8s-rs"

controller:
  # Reconciliation interval
  reconcile_interval_secs: 30
  # Leader election (for HA)
  leader_election:
    enabled: true
    lease_name: "k8s-rs-controller"
    lease_namespace: "kube-system"

observability:
  # Metrics endpoint
  metrics_addr: "0.0.0.0:9091"
  # Tracing (OpenTelemetry)
  tracing:
    enabled: false
    otlp_endpoint: ""
  # Log level: trace, debug, info, warn, error
  log_level: "info"
```

---

## Next Steps

Proceed to **[Step 2: Setting Up Kubernetes Client](step-2-k8s-client.md)** to:
1. Set up the workspace structure
2. Add kube-rs dependencies
3. Implement basic Kubernetes API connectivity
4. Create the k8s-client crate

---

## References

- [kube-rs Documentation](https://kube.rs/)
- [kube-rs GitHub](https://github.com/kube-rs/kube)
- [Pingora GitHub](https://github.com/cloudflare/pingora)
- [Pingora Blog Post](https://blog.cloudflare.com/pingora-open-source/)
- [Kubernetes API Reference](https://kubernetes.io/docs/reference/kubernetes-api/)
