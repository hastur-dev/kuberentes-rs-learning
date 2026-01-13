# Step 6: Ingress Controller with Pingora

## Objective

Build a production-ready Kubernetes Ingress controller using Pingora, supporting TLS termination, path-based routing, and dynamic configuration from Ingress resources.

---

## Prerequisites

- Completed [Step 5: Controllers & Operators](step-5-controllers.md)
- Working controller framework
- Understanding of Kubernetes Ingress concepts

---

## Part A: Ingress Controller Architecture

### Overview

```text
┌────────────────────────────────────────────────────────────────────┐
│                    Pingora Ingress Controller                       │
├────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   ┌─────────────────────────────────────────────────────────────┐  │
│   │                    Ingress Controller                        │  │
│   │                                                              │  │
│   │  ┌──────────┐    ┌──────────┐    ┌──────────┐              │  │
│   │  │  Watch   │───▶│  Build   │───▶│  Update  │              │  │
│   │  │ Ingress  │    │  Config  │    │ Pingora  │              │  │
│   │  │Resources │    │          │    │  Config  │              │  │
│   │  └──────────┘    └──────────┘    └──────────┘              │  │
│   │       │                                │                     │  │
│   │       │ Also watches:                  │                     │  │
│   │       ├── Services                     │                     │  │
│   │       ├── Endpoints                    ▼                     │  │
│   │       ├── Secrets (TLS)     ┌──────────────────┐            │  │
│   │       └── IngressClass      │  Pingora Proxy   │            │  │
│   │                             │  (Hot Reload)    │            │  │
│   └─────────────────────────────┴──────────────────┴────────────┘  │
│                                         │                          │
│                                         ▼                          │
│                               External Traffic                     │
│                                                                     │
└────────────────────────────────────────────────────────────────────┘
```

### IngressClass

Register as an IngressClass provider:

```yaml
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: pingora
  annotations:
    ingressclass.kubernetes.io/is-default-class: "true"
spec:
  controller: k8s-rs.io/pingora
```

Controllers only process Ingress resources that:
1. Reference this IngressClass, OR
2. Have no class and this is the default class

---

## Part B: Ingress Resource Processing

### Ingress Structure

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-app
  namespace: default
  annotations:
    # Pingora-specific annotations
    pingora.k8s-rs.io/ssl-redirect: "true"
    pingora.k8s-rs.io/proxy-body-size: "10m"
    pingora.k8s-rs.io/upstream-hash-by: "$request_uri"
spec:
  ingressClassName: pingora
  tls:
    - hosts:
        - myapp.example.com
      secretName: myapp-tls
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 8080
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-service
                port:
                  number: 80
```

### Processing Flow

```text
Ingress Resource
      │
      ▼
┌─────────────────────────┐
│ 1. Validate IngressClass│
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│ 2. Resolve TLS Secrets  │──▶ Load cert/key from Secrets
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│ 3. Resolve Services     │──▶ Get Service ClusterIP/Ports
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│ 4. Resolve Endpoints    │──▶ Get Pod IPs for upstreams
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│ 5. Build Route Config   │──▶ Host → Path → Backend mapping
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│ 6. Update Pingora       │──▶ Hot reload configuration
└─────────────────────────┘
```

---

## Part C: Module Structure

### proxy-service Layout

```text
services/proxy-service/
├── Cargo.toml
├── src/
│   ├── main.rs                 # Entry point
│   ├── server.rs               # Pingora server setup
│   ├── controller/
│   │   ├── mod.rs
│   │   ├── ingress.rs          # Ingress reconciler
│   │   ├── service.rs          # Service watcher
│   │   ├── endpoints.rs        # Endpoints watcher
│   │   └── secret.rs           # TLS secret watcher
│   ├── config/
│   │   ├── mod.rs
│   │   ├── route.rs            # Route configuration
│   │   ├── upstream.rs         # Upstream configuration
│   │   └── tls.rs              # TLS configuration
│   ├── proxy/
│   │   ├── mod.rs
│   │   ├── http_proxy.rs       # Pingora HTTP proxy impl
│   │   ├── router.rs           # Request routing
│   │   └── filters.rs          # Request/response filters
│   └── metrics.rs              # Prometheus metrics
├── Dockerfile
└── k8s/
    ├── deployment.yaml
    ├── service.yaml
    ├── rbac.yaml
    └── ingressclass.yaml
```

---

## Part D: Route Configuration

### Internal Route Model

```text
RouteConfig:
├── hosts: Map<String, HostConfig>
│   └── HostConfig:
│       ├── tls: Option<TlsConfig>
│       │   ├── cert: Vec<u8>
│       │   └── key: Vec<u8>
│       └── paths: Vec<PathConfig>
│           └── PathConfig:
│               ├── path: String
│               ├── path_type: Exact | Prefix | ImplementationSpecific
│               ├── backend: BackendRef
│               │   ├── name: String
│               │   ├── namespace: String
│               │   └── port: u16
│               └── upstream_pool: UpstreamPool
│                   └── endpoints: Vec<Endpoint>
│                       ├── address: IpAddr
│                       ├── port: u16
│                       └── weight: u32
└── default_backend: Option<BackendRef>
```

### Route Matching Algorithm

```text
Request: GET https://myapp.example.com/api/users

1. TLS Handshake
   └── SNI: myapp.example.com → Select cert from hosts["myapp.example.com"].tls

2. Host Matching
   └── Host header: myapp.example.com → hosts["myapp.example.com"]

3. Path Matching (in order):
   ├── Check Exact matches first
   │   └── /api/users == /api/users? No exact match
   │
   ├── Check Prefix matches (longest first)
   │   ├── /api/users starts with /api? YES → Match!
   │   └── /api/users starts with /? (shorter, lower priority)
   │
   └── Select backend: api-service:8080

4. Upstream Selection
   └── Load balance across api-service endpoints
```

### Path Type Semantics

| PathType | Behavior | Example Path | Matches |
|----------|----------|--------------|---------|
| Exact | Exact string match | `/foo` | `/foo` only |
| Prefix | Path prefix match | `/foo` | `/foo`, `/foo/`, `/foo/bar` |
| ImplementationSpecific | Regex (our impl) | `/api/v[0-9]+` | `/api/v1`, `/api/v2` |

---

## Part E: TLS Handling

### TLS Secret Processing

```text
Secret (type: kubernetes.io/tls)
├── data:
│   ├── tls.crt → PEM certificate chain
│   └── tls.key → PEM private key
│
   Processing:
   1. Watch Secrets referenced by Ingress.spec.tls[].secretName
   2. Decode base64 PEM data
   3. Validate cert/key pair
   4. Build Pingora TLS config
   5. Hot reload on Secret update
```

### TLS Configuration

```text
TlsConfig:
├── cert_chain: Vec<Certificate>
├── private_key: PrivateKey
├── protocols: Vec<TlsVersion>  # TLS 1.2, 1.3
├── ciphers: Vec<CipherSuite>   # Modern ciphers only
├── alpn: Vec<String>           # h2, http/1.1
└── sni_required: bool          # Reject if no SNI
```

### Certificate Management

**Cert-manager integration (recommended):**
```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: myapp-tls
spec:
  secretName: myapp-tls-secret
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - myapp.example.com
```

The Ingress controller watches the resulting Secret automatically.

---

## Part F: Upstream Discovery

### Service to Upstream Mapping

```text
Kubernetes Service
      │
      ├── ClusterIP: 10.96.100.50
      ├── Ports:
      │   └── http: 8080/TCP
      │
      └── Endpoints (from EndpointSlice/Endpoints)
          ├── 10.244.1.5:8080 (Ready)
          ├── 10.244.2.8:8080 (Ready)
          └── 10.244.1.12:8080 (NotReady) ← Excluded

          ▼

UpstreamPool:
├── name: "default/api-service:8080"
├── endpoints:
│   ├── 10.244.1.5:8080 (weight: 1)
│   └── 10.244.2.8:8080 (weight: 1)
├── health_check:
│   └── from Ingress annotations or defaults
└── lb_algorithm: RoundRobin
```

### Endpoint Readiness

Only include endpoints where:
- `conditions.ready == true`
- Pod is Running
- Container passed readiness probe

Watch EndpointSlices (preferred) or Endpoints for changes.

---

## Part G: Annotations

### Supported Annotations

| Annotation | Type | Default | Description |
|------------|------|---------|-------------|
| `pingora.k8s-rs.io/ssl-redirect` | bool | true | Redirect HTTP to HTTPS |
| `pingora.k8s-rs.io/force-ssl-redirect` | bool | false | Always redirect (ignore X-Forwarded-Proto) |
| `pingora.k8s-rs.io/proxy-body-size` | size | 1m | Max request body size |
| `pingora.k8s-rs.io/proxy-read-timeout` | duration | 60s | Backend read timeout |
| `pingora.k8s-rs.io/proxy-send-timeout` | duration | 60s | Backend send timeout |
| `pingora.k8s-rs.io/proxy-connect-timeout` | duration | 5s | Backend connect timeout |
| `pingora.k8s-rs.io/load-balance` | string | round_robin | LB algorithm |
| `pingora.k8s-rs.io/upstream-hash-by` | string | - | Consistent hash key |
| `pingora.k8s-rs.io/whitelist-source-range` | CIDR list | - | Allowed client IPs |
| `pingora.k8s-rs.io/enable-cors` | bool | false | Enable CORS |
| `pingora.k8s-rs.io/cors-allow-origin` | string | * | CORS origin |
| `pingora.k8s-rs.io/rate-limit` | string | - | Rate limit (e.g., "100r/s") |

### Annotation Processing

```text
Ingress annotations:
  pingora.k8s-rs.io/ssl-redirect: "true"
  pingora.k8s-rs.io/proxy-body-size: "50m"
  pingora.k8s-rs.io/load-balance: "least_connections"

       ▼ Parse & Validate

AnnotationConfig:
├── ssl_redirect: true
├── proxy_body_size: 52428800  # bytes
├── load_balance: LeastConnections
└── ... defaults for unspecified
```

---

## Part H: Hot Reload

### Configuration Update Flow

```text
Change detected (Ingress/Service/Secret/Endpoints)
      │
      ▼
┌─────────────────────────┐
│   Debounce (100ms)      │──▶ Batch rapid changes
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│   Rebuild RouteConfig   │
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│    Validate Config      │──▶ Reject invalid configs
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│   Diff with Current     │──▶ Skip if no changes
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────┐
│   Atomic Swap Config    │──▶ Arc<RouteConfig> swap
└───────────┬─────────────┘
            │
            ▼
Pingora uses new config for new connections
(existing connections unaffected)
```

### Zero-Downtime Updates

- Use `Arc<RwLock<RouteConfig>>` or atomic swap
- Existing connections continue with old config
- New connections get new config immediately
- No listener restart required

---

## Part I: Health & Readiness

### Ingress Controller Health Endpoints

```text
GET /healthz    → Controller health
    ├── Kubernetes API reachable
    ├── Leader election (if enabled)
    └── Configuration valid

GET /readyz     → Ready to serve traffic
    ├── Initial sync complete
    ├── At least one valid route
    └── Pingora proxy running

GET /metrics    → Prometheus metrics
```

### Status Updates

Update Ingress status with assigned addresses:

```yaml
status:
  loadBalancer:
    ingress:
      - ip: 192.168.1.100      # External IP
        hostname: lb.example.com # Or hostname
        ports:
          - port: 80
            protocol: TCP
          - port: 443
            protocol: TCP
```

---

## Part J: Deployment Configuration

### Deployment Manifest

```yaml
# k8s/deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: pingora-ingress-controller
  namespace: ingress-system
spec:
  replicas: 2  # HA
  selector:
    matchLabels:
      app: pingora-ingress
  template:
    metadata:
      labels:
        app: pingora-ingress
    spec:
      serviceAccountName: pingora-ingress
      containers:
        - name: controller
          image: k8s-rs/pingora-ingress:latest
          ports:
            - name: http
              containerPort: 80
            - name: https
              containerPort: 443
            - name: metrics
              containerPort: 9090
          args:
            - --config=/etc/pingora/config.yaml
            - --ingress-class=pingora
            - --election-id=pingora-leader
          env:
            - name: POD_NAME
              valueFrom:
                fieldRef:
                  fieldPath: metadata.name
            - name: POD_NAMESPACE
              valueFrom:
                fieldRef:
                  fieldPath: metadata.namespace
          readinessProbe:
            httpGet:
              path: /readyz
              port: 9090
            initialDelaySeconds: 5
            periodSeconds: 5
          livenessProbe:
            httpGet:
              path: /healthz
              port: 9090
            initialDelaySeconds: 10
            periodSeconds: 10
          resources:
            requests:
              cpu: 100m
              memory: 128Mi
            limits:
              cpu: 1000m
              memory: 512Mi
```

### Service Configuration

```yaml
# k8s/service.yaml
apiVersion: v1
kind: Service
metadata:
  name: pingora-ingress
  namespace: ingress-system
spec:
  type: LoadBalancer  # Or NodePort for bare-metal
  selector:
    app: pingora-ingress
  ports:
    - name: http
      port: 80
      targetPort: 80
    - name: https
      port: 443
      targetPort: 443
```

---

## Part K: Testing

### Unit Tests

- Route matching logic
- Annotation parsing
- TLS certificate loading
- Configuration diffing

### Integration Tests

```text
Test scenarios:
├── Create Ingress → Routes configured
├── Update Ingress path → Route updated
├── Delete Ingress → Routes removed
├── Add TLS secret → HTTPS works
├── Update TLS secret → Cert rotated
├── Scale backend → Endpoints updated
├── Backend becomes unhealthy → Removed from pool
├── Multiple Ingresses → All routes work
└── Invalid Ingress → Rejected, status updated
```

### E2E Tests

```text
End-to-end flow:
1. Deploy test backend (echo server)
2. Create Service for backend
3. Create Ingress with path rules
4. Send HTTP request → Verify routing
5. Create TLS secret
6. Update Ingress with TLS
7. Send HTTPS request → Verify TLS
8. Verify redirect HTTP → HTTPS
9. Cleanup
```

---

## Verification Checklist

Before proceeding to Step 7, verify:

| Check | Command | Expected |
|-------|---------|----------|
| Proxy service builds | `cargo build -p proxy-service` | Success |
| IngressClass applied | `kubectl get ingressclass` | pingora listed |
| Controller runs | `cargo run -p proxy-service` | Watching Ingress... |
| HTTP routing works | `curl -H "Host: test.local" http://localhost/` | Backend response |
| HTTPS works | `curl -k https://test.local/` | TLS handshake OK |
| Path routing works | Test multiple paths | Correct backends |
| Hot reload works | Update Ingress | New routes active |

---

## Next Steps

Proceed to **[Step 7: Observability & Monitoring](step-7-observability.md)** to:
1. Add Prometheus metrics
2. Implement distributed tracing
3. Configure structured logging
4. Build dashboards

---

## References

- [Kubernetes Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
- [Ingress Controllers](https://kubernetes.io/docs/concepts/services-networking/ingress-controllers/)
- [IngressClass](https://kubernetes.io/docs/concepts/services-networking/ingress/#ingress-class)
- [Pingora TLS Configuration](https://github.com/cloudflare/pingora/blob/main/docs/user_guide/tls.md)
