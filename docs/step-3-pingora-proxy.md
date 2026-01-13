# Step 3: Building the Pingora Proxy Layer

## Objective

Replace Nginx with Cloudflare's Pingora framework to create a high-performance, programmable proxy layer that integrates with Kubernetes for dynamic upstream discovery.

---

## Prerequisites

- Completed [Step 2: Kubernetes Client](step-2-k8s-client.md)
- Working k8s-client crate
- Understanding of HTTP proxy concepts

---

## Part A: Understanding Pingora

### Why Pingora?

| Feature | Nginx | Pingora |
|---------|-------|---------|
| Language | C | Rust |
| Memory Safety | Manual | Guaranteed |
| Configuration | Static files + Lua | Rust code |
| HTTP/3 (QUIC) | Limited | Native |
| Connection Pooling | Basic | Advanced multiplexing |
| Programmability | Lua scripting | Full Rust ecosystem |
| Performance | Excellent | Comparable/Better |
| CVE History | Many buffer overflows | Memory-safe by design |

### Pingora Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      Pingora Server                          │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐   │
│  │   Listener   │───▶│   Service    │───▶│   Upstream   │   │
│  │  (TCP/TLS)   │    │  (HTTP Proxy)│    │  (Backend)   │   │
│  └──────────────┘    └──────────────┘    └──────────────┘   │
│                              │                               │
│                      ┌───────┴───────┐                      │
│                      │               │                      │
│               ┌──────▼──────┐ ┌──────▼──────┐              │
│               │   Filters   │ │Load Balancer│              │
│               │  (Request/  │ │ (RoundRobin │              │
│               │  Response)  │ │  Weighted)  │              │
│               └─────────────┘ └─────────────┘              │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

---

## Part B: Pingora Dependencies

### 1. Add to Workspace Dependencies

```toml
# Cargo.toml (workspace root) - add to [workspace.dependencies]
[workspace.dependencies]
# ... existing deps ...

# Pingora
pingora = "0.4"
pingora-core = "0.4"
pingora-proxy = "0.4"
pingora-load-balancing = "0.4"
pingora-cache = "0.4"
pingora-timeout = "0.4"

# Async utilities
async-trait = "0.1"
futures = "0.3"

# HTTP types
http = "1.1"
bytes = "1.7"
```

### 2. pingora-proxy Crate Manifest

```toml
# crates/pingora-proxy/Cargo.toml
[package]
name = "pingora-proxy"
version.workspace = true
edition.workspace = true
rust-version.workspace = true

[dependencies]
shared = { workspace = true }
k8s-client = { workspace = true }

pingora = { workspace = true }
pingora-core = { workspace = true }
pingora-proxy = { workspace = true }
pingora-load-balancing = { workspace = true }

tokio = { workspace = true }
async-trait = { workspace = true }
futures = { workspace = true }
http = { workspace = true }
bytes = { workspace = true }

serde = { workspace = true }
thiserror = { workspace = true }
tracing = { workspace = true }

[dev-dependencies]
tokio-test = { workspace = true }
```

---

## Part C: Proxy Module Structure

```text
crates/pingora-proxy/src/
├── lib.rs              # Public API
├── server.rs           # ProxyServer builder & lifecycle
├── service.rs          # HTTP proxy service implementation
├── upstream/
│   ├── mod.rs          # Upstream management exports
│   ├── discovery.rs    # K8s-based service discovery
│   ├── health.rs       # Health checking
│   └── pool.rs         # Connection pooling config
├── lb/
│   ├── mod.rs          # Load balancing exports
│   ├── round_robin.rs  # Round-robin selection
│   ├── weighted.rs     # Weighted selection
│   └── least_conn.rs   # Least connections
├── filters/
│   ├── mod.rs          # Filter chain exports
│   ├── request.rs      # Request modification filters
│   ├── response.rs     # Response modification filters
│   ├── auth.rs         # Authentication filter
│   └── rate_limit.rs   # Rate limiting filter
└── error.rs            # ProxyError enum
```

---

## Part D: Core Concepts

### 1. ProxyServer Lifecycle

```text
ProxyServer::builder()
    │
    ├── .with_config(config)           # Load from config.yaml
    ├── .with_k8s_client(client)       # For upstream discovery
    ├── .with_listen_addr(addr)        # Bind address
    ├── .with_tls(cert, key)           # Optional TLS
    ├── .with_filters(vec![...])       # Request/response filters
    └── .build()
           │
           ▼
    ProxyServer
           │
    .start()  ──────────────────────▶  Running
           │                              │
           │                              ├── Accept connections
           │                              ├── Route to upstreams
           │                              ├── Apply filters
           │                              └── Health check upstreams
           │
    .shutdown()  ───────────────────▶  Graceful shutdown
```

### 2. Request Flow

```text
Client Request
      │
      ▼
┌─────────────────┐
│ Request Filters │──▶ Auth, Rate Limit, Headers
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Upstream Select │──▶ Load Balancer chooses backend
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│ Forward Request │──▶ Connection from pool
└────────┬────────┘
         │
         ▼
┌──────────────────┐
│ Response Filters │──▶ Headers, Compression
└────────┬─────────┘
         │
         ▼
   Client Response
```

### 3. Upstream Discovery from Kubernetes

```text
K8s Service (type: ClusterIP/LoadBalancer)
         │
         │  k8s-client watches Services + Endpoints
         ▼
┌─────────────────────┐
│  Service Discovery  │
│                     │
│  - Service name     │
│  - Port mappings    │
│  - Endpoint IPs     │
│  - Health status    │
└────────┬────────────┘
         │
         │  Updates upstream pool
         ▼
┌─────────────────────┐
│   Upstream Pool     │
│                     │
│  backend-1:8080 ✓   │
│  backend-2:8080 ✓   │
│  backend-3:8080 ✗   │ (unhealthy, excluded)
└─────────────────────┘
```

---

## Part E: Implementation Guide

### 1. HTTP Proxy Service Trait

Pingora uses a trait-based design. Implement `ProxyHttp`:

**Key methods to implement:**

| Method | Purpose |
|--------|---------|
| `upstream_peer()` | Select backend for request |
| `request_filter()` | Modify/reject incoming request |
| `upstream_request_filter()` | Modify request to backend |
| `response_filter()` | Modify response to client |
| `logging()` | Custom access logging |
| `fail_to_connect()` | Handle connection failures |

### 2. Upstream Selection Logic

**Selection flow:**
1. Extract routing info from request (Host header, path)
2. Look up matching upstream pool
3. Apply load balancing algorithm
4. Return selected backend address
5. Handle selection failure (no healthy upstreams)

**Routing rules (priority order):**
1. Exact host + path match
2. Host + path prefix match
3. Host wildcard match
4. Default backend

### 3. Health Checking Strategy

**Active health checks:**
- Periodic HTTP GET to `/healthz` or custom path
- Configurable interval (default: 10s)
- Configurable threshold (default: 3 failures = unhealthy)
- Configurable timeout (default: 5s)

**Passive health checks:**
- Mark unhealthy on connection failure
- Mark unhealthy on 5xx responses (configurable)
- Automatic recovery after success threshold

### 4. Connection Pooling

**Pool configuration:**
```text
UpstreamPoolConfig:
  - max_connections_per_upstream: 100
  - idle_timeout: 60s
  - connection_timeout: 10s
  - max_retries: 3
  - retry_on: [connection_error, 502, 503, 504]
```

---

## Part F: Testing Strategy

### 1. Unit Tests

- Filter chain execution order
- Load balancer selection distribution
- Health check state transitions
- Routing rule matching

### 2. Integration Tests

**Test server setup:**
```text
1. Start mock HTTP backend(s)
2. Start Pingora proxy pointing to backends
3. Send requests through proxy
4. Verify correct routing, filtering
5. Verify metrics/logging
```

**Test scenarios:**
- Basic proxy pass-through
- Load balancing distribution
- Health check removing unhealthy backend
- Request/response filter modification
- Connection pooling reuse
- Graceful shutdown with in-flight requests

### 3. Performance Tests

**Benchmarks to include:**
- Requests/second at various concurrency levels
- Latency percentiles (p50, p95, p99)
- Memory usage under load
- Connection pool efficiency

Use `criterion` for benchmarks:
```text
crates/pingora-proxy/benches/
├── proxy_benchmark.rs
└── lb_benchmark.rs
```

---

## Part G: Configuration

### Proxy Configuration Section

```yaml
# config.yaml - pingora section
pingora:
  # Main proxy listener
  listen:
    address: "0.0.0.0"
    port: 8080
    # TLS configuration
    tls:
      enabled: false
      cert_path: "/etc/certs/tls.crt"
      key_path: "/etc/certs/tls.key"

  # Admin API (for health checks, metrics)
  admin:
    address: "127.0.0.1"
    port: 9090

  # Upstream discovery
  upstream:
    # Discover from Kubernetes Services
    kubernetes:
      enabled: true
      # Only discover services with this label
      label_selector: "proxy.k8s-rs/enabled=true"
      # Namespace to watch (empty = all)
      namespace: ""
      # Sync interval
      sync_interval_secs: 30

    # Static upstreams (fallback/override)
    static:
      - name: "default-backend"
        endpoints:
          - "10.0.0.1:8080"
          - "10.0.0.2:8080"

  # Connection pool settings
  pool:
    max_connections: 1000
    idle_timeout_secs: 60
    connection_timeout_secs: 10

  # Health checking
  health_check:
    enabled: true
    interval_secs: 10
    timeout_secs: 5
    path: "/healthz"
    unhealthy_threshold: 3
    healthy_threshold: 2

  # Load balancing
  load_balancing:
    algorithm: "round_robin"  # round_robin, weighted, least_connections

  # Request limits
  limits:
    max_request_body_bytes: 10485760  # 10MB
    request_timeout_secs: 60
```

---

## Verification Checklist

Before proceeding to Step 4, verify:

| Check | Command | Expected |
|-------|---------|----------|
| Crate builds | `cargo build -p pingora-proxy` | Success |
| Tests pass | `cargo test -p pingora-proxy` | All green |
| Example runs | `cargo run --example basic_proxy` | Starts on 8080 |
| Proxy forwards | `curl http://localhost:8080/` | Backend response |
| Health endpoint | `curl http://localhost:9090/health` | 200 OK |

---

## Common Issues

### Issue: "error linking with `cc`" on Windows

**Cause:** Pingora has Unix-specific dependencies.

**Solution:** Use WSL2 or Docker for development:
```powershell
# Run in Docker
docker run -it --rm -v ${PWD}:/app -w /app rust:latest cargo build
```

### Issue: "Address already in use"

**Cause:** Previous proxy instance still running.

**Solution:**
```bash
# Find and kill process
lsof -i :8080
kill -9 <PID>
```

### Issue: Upstream connection timeout

**Cause:** Backend not reachable from proxy.

**Solution:**
1. Verify backend is running: `curl http://backend:port/`
2. Check network connectivity
3. Ensure correct port mapping in config

---

## Next Steps

Proceed to **[Step 4: Kubernetes Resource Management](step-4-resource-mgmt.md)** to:
1. Implement CRUD operations for common K8s resources
2. Build resource templating system
3. Add resource validation
4. Create deployment pipelines

---

## References

- [Pingora GitHub](https://github.com/cloudflare/pingora)
- [Pingora Quick Start](https://github.com/cloudflare/pingora/blob/main/docs/quick_start.md)
- [Pingora User Guide](https://github.com/cloudflare/pingora/blob/main/docs/user_guide/index.md)
- [Cloudflare Blog: Open Sourcing Pingora](https://blog.cloudflare.com/pingora-open-source/)
