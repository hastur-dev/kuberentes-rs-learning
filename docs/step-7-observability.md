# Step 7: Observability & Monitoring

## Objective

Implement comprehensive observability including Prometheus metrics, distributed tracing with OpenTelemetry, and structured logging for all components.

---

## Prerequisites

- Completed [Step 6: Ingress Controller](step-6-ingress.md)
- Working proxy and controller services
- Understanding of observability concepts (metrics, traces, logs)

---

## Part A: Observability Stack Overview

### Three Pillars

```text
┌─────────────────────────────────────────────────────────────────┐
│                    Observability Pillars                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐        │
│   │   METRICS    │   │   TRACES     │   │    LOGS      │        │
│   │              │   │              │   │              │        │
│   │ What happened│   │ Why it       │   │ Detailed     │        │
│   │ (aggregated) │   │ happened     │   │ context      │        │
│   │              │   │ (per request)│   │              │        │
│   │ Prometheus   │   │ OpenTelemetry│   │ Structured   │        │
│   │ + Grafana    │   │ + Jaeger     │   │ JSON logs    │        │
│   └──────────────┘   └──────────────┘   └──────────────┘        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Architecture

```text
┌─────────────────────────────────────────────────────────────────┐
│                    k8s-rs Services                               │
│  ┌─────────────────┐           ┌─────────────────┐              │
│  │  proxy-service  │           │controller-service│              │
│  │                 │           │                  │              │
│  │  ┌───────────┐  │           │  ┌───────────┐  │              │
│  │  │  Metrics  │──┼───────────┼──│  Metrics  │  │              │
│  │  │   :9090   │  │           │  │   :9091   │  │              │
│  │  └───────────┘  │           │  └───────────┘  │              │
│  │                 │           │                  │              │
│  │  ┌───────────┐  │           │  ┌───────────┐  │              │
│  │  │  Tracer   │──┼───────────┼──│  Tracer   │  │              │
│  │  └───────────┘  │           │  └───────────┘  │              │
│  │                 │           │                  │              │
│  │  ┌───────────┐  │           │  ┌───────────┐  │              │
│  │  │  Logger   │──┼───────────┼──│  Logger   │  │              │
│  │  └───────────┘  │           │  └───────────┘  │              │
│  └────────┬────────┘           └────────┬────────┘              │
└───────────┼─────────────────────────────┼───────────────────────┘
            │                             │
            ▼                             ▼
┌───────────────────────┐    ┌───────────────────────┐
│      Prometheus       │    │    OTLP Collector     │
│    (scrape metrics)   │    │   (receive traces)    │
└───────────┬───────────┘    └───────────┬───────────┘
            │                            │
            ▼                            ▼
┌───────────────────────┐    ┌───────────────────────┐
│       Grafana         │    │        Jaeger         │
│    (dashboards)       │    │   (trace viewer)      │
└───────────────────────┘    └───────────────────────┘
```

---

## Part B: Prometheus Metrics

### Dependencies

```toml
# Add to workspace dependencies
[workspace.dependencies]
# Metrics
prometheus = "0.13"
prometheus-client = "0.22"  # Alternative: official Prometheus client
metrics = "0.23"            # Facade for metrics collection
metrics-exporter-prometheus = "0.15"
```

### Metric Types

| Type | Use Case | Example |
|------|----------|---------|
| Counter | Cumulative counts | Total requests, errors |
| Gauge | Current value | Active connections, queue depth |
| Histogram | Distributions | Request latency, response sizes |

### Proxy Metrics

```text
# Request metrics
pingora_http_requests_total{method, host, path, status}
pingora_http_request_duration_seconds{method, host, path}
pingora_http_request_size_bytes{method, host}
pingora_http_response_size_bytes{method, host, status}

# Connection metrics
pingora_connections_active{listener}
pingora_connections_total{listener, state}  # state: accepted, closed, error

# Upstream metrics
pingora_upstream_requests_total{upstream, status}
pingora_upstream_request_duration_seconds{upstream}
pingora_upstream_connections_active{upstream}
pingora_upstream_health{upstream}  # 1 = healthy, 0 = unhealthy

# TLS metrics
pingora_tls_handshakes_total{host, version}
pingora_tls_handshake_duration_seconds{host}
```

### Controller Metrics

```text
# Reconciliation metrics
controller_reconcile_total{controller, result}  # result: success, error, requeue
controller_reconcile_duration_seconds{controller}
controller_reconcile_errors_total{controller, error_type}

# Resource metrics
controller_resources_total{controller, kind}
controller_owned_resources_total{controller, kind}

# Watch metrics
controller_watch_events_total{controller, kind, event}  # event: add, update, delete
controller_watch_errors_total{controller, kind}

# Leader election
controller_leader_election_status{controller}  # 1 = leader, 0 = follower
```

### Metrics Endpoint

```text
GET /metrics HTTP/1.1

# Response (Prometheus text format):
# HELP pingora_http_requests_total Total HTTP requests
# TYPE pingora_http_requests_total counter
pingora_http_requests_total{method="GET",host="api.example.com",status="200"} 15234
pingora_http_requests_total{method="POST",host="api.example.com",status="201"} 423
...
```

---

## Part C: Distributed Tracing

### Dependencies

```toml
# Add to workspace dependencies
[workspace.dependencies]
# OpenTelemetry
opentelemetry = "0.27"
opentelemetry_sdk = { version = "0.27", features = ["rt-tokio"] }
opentelemetry-otlp = { version = "0.27", features = ["http-proto", "reqwest-client"] }
tracing-opentelemetry = "0.28"
```

### Trace Structure

```text
Trace: user-request-abc123
│
├── Span: proxy.receive_request
│   ├── Attributes:
│   │   ├── http.method: GET
│   │   ├── http.url: https://api.example.com/users/123
│   │   ├── http.host: api.example.com
│   │   └── net.peer.ip: 192.168.1.100
│   │
│   └── Child Span: proxy.route_request
│       ├── Attributes:
│       │   ├── route.host: api.example.com
│       │   ├── route.path: /users/123
│       │   └── route.backend: api-service
│       │
│       └── Child Span: proxy.upstream_request
│           ├── Attributes:
│           │   ├── upstream.name: api-service
│           │   ├── upstream.address: 10.244.1.5:8080
│           │   └── http.status_code: 200
│           └── Duration: 45ms
│
└── Total Duration: 52ms
```

### Trace Context Propagation

```text
Incoming request:
  Headers:
    traceparent: 00-abc123...-def456...-01
    tracestate: vendor=value

          │
          ▼

Proxy extracts trace context
          │
          ▼

Upstream request:
  Headers:
    traceparent: 00-abc123...-ghi789...-01  # Same trace, new span
    tracestate: vendor=value
```

Support W3C Trace Context headers:
- `traceparent`: Trace ID, Span ID, flags
- `tracestate`: Vendor-specific data

### Sampling Configuration

```yaml
# config.yaml - observability section
observability:
  tracing:
    enabled: true
    # OTLP endpoint (Jaeger, Tempo, etc.)
    otlp_endpoint: "http://jaeger-collector:4317"
    # Service name in traces
    service_name: "pingora-proxy"
    # Sampling strategy
    sampling:
      # Options: always_on, always_off, trace_id_ratio, parent_based
      strategy: "parent_based"
      # For trace_id_ratio: sample this fraction (0.0 - 1.0)
      ratio: 0.1  # 10% of traces
```

---

## Part D: Structured Logging

### Dependencies

```toml
# Already in workspace, ensure features:
tracing = "0.1"
tracing-subscriber = { version = "0.3", features = ["env-filter", "json", "fmt"] }
```

### Log Levels

| Level | Use Case |
|-------|----------|
| ERROR | Unrecoverable failures, requires attention |
| WARN | Recoverable issues, degraded operation |
| INFO | Normal operation, significant events |
| DEBUG | Detailed operational info |
| TRACE | Very detailed, per-request data |

### Log Format (JSON)

```json
{
  "timestamp": "2024-01-15T10:30:45.123Z",
  "level": "INFO",
  "target": "pingora_proxy::router",
  "message": "Request routed to upstream",
  "span": {
    "request_id": "abc123",
    "trace_id": "def456"
  },
  "fields": {
    "method": "GET",
    "host": "api.example.com",
    "path": "/users/123",
    "upstream": "api-service",
    "upstream_addr": "10.244.1.5:8080",
    "duration_ms": 45
  }
}
```

### Log Configuration

```yaml
# config.yaml
observability:
  logging:
    # Log level (trace, debug, info, warn, error)
    level: "info"
    # Per-module levels
    filters:
      - "pingora_proxy=debug"
      - "kube=warn"
      - "hyper=warn"
    # Format: json (production) or pretty (development)
    format: "json"
    # Include span context in logs
    include_span: true
```

### Log Initialization

```text
Initialization order:
1. Parse config
2. Set up tracing subscriber with:
   ├── EnvFilter (log levels)
   ├── JSON formatter OR pretty formatter
   ├── OpenTelemetry layer (if tracing enabled)
   └── Output to stdout
3. Set as global default
```

---

## Part E: Observability Module Structure

```text
crates/shared/src/
├── observability/
│   ├── mod.rs              # Public API, init functions
│   ├── metrics.rs          # Metric registry, helpers
│   ├── tracing.rs          # Tracer setup, span helpers
│   └── logging.rs          # Log subscriber setup
```

### Initialization API

```text
ObservabilityBuilder::new()
    .with_config(&config.observability)
    .with_service_name("pingora-proxy")
    .with_metrics_endpoint(([0, 0, 0, 0], 9090))
    .build()
    .init()?;

// After init:
// - Global tracing subscriber active
// - Metrics registry ready
// - OTLP exporter running (if enabled)
```

---

## Part F: Dashboard Design

### Grafana Dashboard: Proxy Overview

```text
┌─────────────────────────────────────────────────────────────────┐
│  Pingora Proxy Overview                         [Last 1h ▼]     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌─────────┐ │
│  │   Requests   │ │    Errors    │ │   Latency    │ │  Conns  │ │
│  │    /sec      │ │    Rate %    │ │    p99       │ │ Active  │ │
│  │   12.5k      │ │    0.02%     │ │    45ms      │ │   234   │ │
│  └──────────────┘ └──────────────┘ └──────────────┘ └─────────┘ │
│                                                                  │
│  Request Rate by Status                    Latency Distribution  │
│  ┌─────────────────────────────┐  ┌─────────────────────────────┐│
│  │    ████████████ 2xx         │  │         ▂▄▆█▆▄▂             ││
│  │    ██ 3xx                   │  │     p50  p90 p99            ││
│  │    █ 4xx                    │  │     12ms 32ms 45ms          ││
│  │    ▏ 5xx                    │  │                             ││
│  └─────────────────────────────┘  └─────────────────────────────┘│
│                                                                  │
│  Upstream Health                           Top Routes by Traffic │
│  ┌─────────────────────────────┐  ┌─────────────────────────────┐│
│  │ api-service      ● Healthy  │  │ /api/users        45%      ││
│  │ auth-service     ● Healthy  │  │ /api/orders       30%      ││
│  │ static-service   ○ Degraded │  │ /health           15%      ││
│  │ payment-service  ● Healthy  │  │ /api/products     10%      ││
│  └─────────────────────────────┘  └─────────────────────────────┘│
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Grafana Dashboard: Controller

```text
┌─────────────────────────────────────────────────────────────────┐
│  Controller Overview                            [Last 1h ▼]     │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌─────────┐ │
│  │  Reconciles  │ │   Errors     │ │  Queue Depth │ │ Leader  │ │
│  │    /min      │ │   /hour      │ │              │ │         │ │
│  │    125       │ │     3        │ │     12       │ │   Yes   │ │
│  └──────────────┘ └──────────────┘ └──────────────┘ └─────────┘ │
│                                                                  │
│  Reconcile Duration                    Resources Managed         │
│  ┌─────────────────────────────┐  ┌─────────────────────────────┐│
│  │         ▂▄█▄▂               │  │ ProxyConfig:      15       ││
│  │     p50: 45ms               │  │ BackendPool:      23       ││
│  │     p99: 250ms              │  │ Deployments:      38       ││
│  └─────────────────────────────┘  └─────────────────────────────┘│
│                                                                  │
│  Watch Events                              Error Breakdown       │
│  ┌─────────────────────────────┐  ┌─────────────────────────────┐│
│  │ ███████ Add                 │  │ Timeout:          2        ││
│  │ ████ Update                 │  │ Conflict:         1        ││
│  │ ██ Delete                   │  │ NotFound:         0        ││
│  └─────────────────────────────┘  └─────────────────────────────┘│
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Key Queries

**Request rate:**
```promql
sum(rate(pingora_http_requests_total[5m])) by (status)
```

**Error rate:**
```promql
sum(rate(pingora_http_requests_total{status=~"5.."}[5m]))
/
sum(rate(pingora_http_requests_total[5m]))
```

**Latency percentiles:**
```promql
histogram_quantile(0.99, sum(rate(pingora_http_request_duration_seconds_bucket[5m])) by (le))
```

**Reconcile duration:**
```promql
histogram_quantile(0.99, sum(rate(controller_reconcile_duration_seconds_bucket[5m])) by (le, controller))
```

---

## Part G: Alerting Rules

### Proxy Alerts

```yaml
# prometheus-rules.yaml
groups:
  - name: pingora-proxy
    rules:
      - alert: HighErrorRate
        expr: |
          sum(rate(pingora_http_requests_total{status=~"5.."}[5m]))
          /
          sum(rate(pingora_http_requests_total[5m]))
          > 0.01
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High error rate on Pingora proxy"
          description: "Error rate is {{ $value | humanizePercentage }}"

      - alert: HighLatency
        expr: |
          histogram_quantile(0.99, sum(rate(pingora_http_request_duration_seconds_bucket[5m])) by (le))
          > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High latency on Pingora proxy"
          description: "p99 latency is {{ $value | humanizeDuration }}"

      - alert: UpstreamUnhealthy
        expr: pingora_upstream_health == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Upstream {{ $labels.upstream }} is unhealthy"
```

### Controller Alerts

```yaml
  - name: controller
    rules:
      - alert: ReconcileErrors
        expr: |
          increase(controller_reconcile_errors_total[1h]) > 10
        labels:
          severity: warning
        annotations:
          summary: "Controller reconcile errors"

      - alert: LeaderLost
        expr: controller_leader_election_status == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Controller lost leader election"

      - alert: ReconcileQueueBacklog
        expr: controller_workqueue_depth > 100
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Reconcile queue backlog growing"
```

---

## Part H: Kubernetes Integration

### ServiceMonitor (Prometheus Operator)

```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: pingora-proxy
  namespace: monitoring
spec:
  selector:
    matchLabels:
      app: pingora-proxy
  endpoints:
    - port: metrics
      interval: 15s
      path: /metrics
```

### PodMonitor

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: pingora-controller
spec:
  selector:
    matchLabels:
      app: pingora-controller
  podMetricsEndpoints:
    - port: metrics
      interval: 30s
```

### Deploy Observability Stack

```bash
# Prometheus + Grafana (kube-prometheus-stack)
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm install monitoring prometheus-community/kube-prometheus-stack \
  --namespace monitoring --create-namespace

# Jaeger
helm repo add jaegertracing https://jaegertracing.github.io/helm-charts
helm install jaeger jaegertracing/jaeger \
  --namespace monitoring \
  --set collector.service.otlp.grpc.enabled=true
```

---

## Verification Checklist

Before proceeding to Step 8, verify:

| Check | Command | Expected |
|-------|---------|----------|
| Metrics endpoint works | `curl localhost:9090/metrics` | Prometheus format output |
| Metrics in Prometheus | Query `up{job="pingora"}` | 1 |
| Traces appear | Check Jaeger UI | Traces visible |
| Logs are JSON | Check stdout | Valid JSON lines |
| Grafana dashboard | Import dashboard | Panels render |
| Alerts configured | Check Alertmanager | Rules loaded |

---

## Next Steps

Proceed to **[Step 8: Deployment & Production Readiness](step-8-deployment.md)** to:
1. Build production container images
2. Configure Helm charts
3. Set up CI/CD pipelines
4. Implement security hardening

---

## References

- [Prometheus Best Practices](https://prometheus.io/docs/practices/naming/)
- [OpenTelemetry Rust](https://opentelemetry.io/docs/languages/rust/)
- [Grafana Dashboard Design](https://grafana.com/docs/grafana/latest/dashboards/)
- [tracing crate documentation](https://docs.rs/tracing/latest/tracing/)
