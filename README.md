# kubernetes-rs-learning

A comprehensive Rust-based Kubernetes deployment and management platform, featuring Cloudflare's Pingora as a high-performance replacement for Nginx.

## Overview

This project provides a complete learning path and implementation guide for building production-grade Kubernetes tooling in Rust:

- **kube-rs** - Native Rust Kubernetes client
- **Pingora** - High-performance HTTP proxy (Nginx replacement)
- **Custom Controllers** - Kubernetes controller/operator pattern
- **Ingress Controller** - Full Ingress implementation with Pingora

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                      kubernetes-rs-learning                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌──────────────┐    ┌──────────────┐    ┌──────────────┐      │
│   │pingora-proxy │    │k8s-controller│    │ k8s-operator │      │
│   │(Load Balancer)    │ (Core Mgmt)  │    │(Custom CRDs) │      │
│   └──────┬───────┘    └──────┬───────┘    └──────┬───────┘      │
│          │                   │                   │               │
│          └─────────────┬─────┴───────────────────┘               │
│                        │                                         │
│              ┌─────────▼─────────┐                               │
│              │    k8s-client     │                               │
│              │    (kube-rs)      │                               │
│              └─────────┬─────────┘                               │
│                        │                                         │
│              ┌─────────▼─────────┐                               │
│              │  Kubernetes API   │                               │
│              └───────────────────┘                               │
└─────────────────────────────────────────────────────────────────┘
```

## Learning Path

This repository is structured as an 8-step learning guide. Each step builds upon the previous:

| Step | Topic | Description |
|------|-------|-------------|
| [Step 1](docs/step-1-foundation.md) | Foundation | Project architecture & prerequisites |
| [Step 2](docs/step-2-k8s-client.md) | K8s Client | Setting up kube-rs and workspace |
| [Step 3](docs/step-3-pingora-proxy.md) | Pingora Proxy | Building the HTTP proxy layer |
| [Step 4](docs/step-4-resource-mgmt.md) | Resource Management | CRUD operations & templating |
| [Step 5](docs/step-5-controllers.md) | Controllers | Custom controllers & operators |
| [Step 6](docs/step-6-ingress.md) | Ingress Controller | Pingora-based Ingress implementation |
| [Step 7](docs/step-7-observability.md) | Observability | Metrics, tracing, logging |
| [Step 8](docs/step-8-deployment.md) | Production | Helm charts, CI/CD, security |

## Why Pingora over Nginx?

| Feature | Nginx | Pingora |
|---------|-------|---------|
| Language | C | Rust |
| Memory Safety | Manual | Guaranteed |
| Configuration | Static files + Lua | Native Rust code |
| HTTP/3 (QUIC) | Limited | Native support |
| Programmability | Lua scripting | Full Rust ecosystem |
| Performance | Excellent | Comparable/Better |

Pingora is battle-tested at Cloudflare, handling millions of requests per second.

## Project Structure (Target)

```
kubernetes-rs-learning/
├── Cargo.toml                    # Workspace root
├── config.yaml                   # Runtime configuration
├── docs/                         # Step-by-step guides
│   ├── step-1-foundation.md
│   ├── step-2-k8s-client.md
│   ├── step-3-pingora-proxy.md
│   ├── step-4-resource-mgmt.md
│   ├── step-5-controllers.md
│   ├── step-6-ingress.md
│   ├── step-7-observability.md
│   └── step-8-deployment.md
├── crates/
│   ├── shared/                   # Common types & utilities
│   ├── k8s-client/               # Kubernetes API wrapper
│   ├── pingora-proxy/            # Pingora proxy implementation
│   ├── k8s-controller/           # Controller runtime
│   └── k8s-operator/             # CRD & operator logic
├── services/
│   ├── proxy-service/            # Deployable proxy
│   └── controller-service/       # Deployable controller
├── charts/
│   └── k8s-rs/                   # Helm chart
└── scripts/
    ├── run_all.sh
    └── run_all.ps1
```

## Prerequisites

| Tool | Minimum Version | Purpose |
|------|-----------------|---------|
| Rust | 1.81+ | Language toolchain |
| Docker | 24.0+ | Container builds |
| kubectl | 1.28+ | Kubernetes CLI |
| kind | Latest | Local K8s cluster |
| Helm | 3.14+ | Package management |

## Quick Start

```bash
# Clone the repository
git clone https://github.com/hastur-dev/kuberentes-rs-learning.git
cd kuberentes-rs-learning

# Create local Kubernetes cluster
kind create cluster --name k8s-rs-dev

# Build the project
cargo build

# Run tests
cargo test

# Start reading the guides
# Begin with docs/step-1-foundation.md
```

## Key Technologies

### kube-rs
- Native async/await with Tokio
- Type-safe Kubernetes API interactions
- Controller runtime for building operators
- Active maintenance and strong community

### Pingora
- Cloudflare's production-proven proxy framework
- Memory-safe, high-performance HTTP proxy
- Native HTTP/2 and HTTP/3 support
- Programmable via Rust (not config files)

## Features (Planned)

- [x] Documentation & learning guides
- [ ] Kubernetes client wrapper
- [ ] Pingora proxy service
- [ ] Resource management & templating
- [ ] Custom Resource Definitions
- [ ] Ingress controller
- [ ] Prometheus metrics
- [ ] OpenTelemetry tracing
- [ ] Helm chart
- [ ] CI/CD pipelines

## Contributing

Contributions are welcome! Please read through the step-by-step guides first to understand the architecture.

## License

MIT OR Apache-2.0

## References

- [kube-rs Documentation](https://kube.rs/)
- [Pingora GitHub](https://github.com/cloudflare/pingora)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Cloudflare Pingora Blog Post](https://blog.cloudflare.com/pingora-open-source/)
