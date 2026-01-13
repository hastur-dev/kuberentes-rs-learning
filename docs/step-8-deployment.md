# Step 8: Deployment & Production Readiness

## Objective

Prepare the project for production deployment including container images, Helm charts, CI/CD pipelines, and security hardening.

---

## Prerequisites

- Completed [Step 7: Observability](step-7-observability.md)
- All components tested and working
- Container registry access
- Kubernetes cluster for deployment

---

## Part A: Container Images

### Multi-Stage Dockerfile

```dockerfile
# Dockerfile (proxy-service example)
# Stage 1: Build
FROM rust:1.81-bookworm AS builder

WORKDIR /app

# Install dependencies for cross-compilation if needed
RUN apt-get update && apt-get install -y \
    musl-tools \
    pkg-config \
    libssl-dev \
    && rm -rf /var/lib/apt/lists/*

# Cache dependencies
COPY Cargo.toml Cargo.lock ./
COPY crates/shared/Cargo.toml crates/shared/
COPY crates/k8s-client/Cargo.toml crates/k8s-client/
COPY crates/pingora-proxy/Cargo.toml crates/pingora-proxy/
COPY crates/k8s-controller/Cargo.toml crates/k8s-controller/
COPY crates/k8s-operator/Cargo.toml crates/k8s-operator/
COPY services/proxy-service/Cargo.toml services/proxy-service/

# Create dummy source files for dependency caching
RUN mkdir -p crates/shared/src crates/k8s-client/src crates/pingora-proxy/src \
    crates/k8s-controller/src crates/k8s-operator/src services/proxy-service/src \
    && echo "fn main() {}" > services/proxy-service/src/main.rs \
    && touch crates/shared/src/lib.rs crates/k8s-client/src/lib.rs \
    crates/pingora-proxy/src/lib.rs crates/k8s-controller/src/lib.rs \
    crates/k8s-operator/src/lib.rs

# Build dependencies only
RUN cargo build --release -p proxy-service && rm -rf target/release/.fingerprint/proxy-service*

# Copy actual source
COPY crates crates
COPY services services

# Build the actual binary
RUN cargo build --release -p proxy-service

# Stage 2: Runtime
FROM debian:bookworm-slim AS runtime

# Install runtime dependencies
RUN apt-get update && apt-get install -y \
    ca-certificates \
    libssl3 \
    && rm -rf /var/lib/apt/lists/*

# Create non-root user
RUN groupadd -r pingora && useradd -r -g pingora pingora

WORKDIR /app

# Copy binary
COPY --from=builder /app/target/release/proxy-service /app/proxy-service

# Copy default config
COPY config.yaml.example /app/config.yaml

# Set ownership
RUN chown -R pingora:pingora /app

USER pingora

EXPOSE 80 443 9090

ENTRYPOINT ["/app/proxy-service"]
CMD ["--config", "/app/config.yaml"]
```

### Build & Push

```bash
# Build images
docker build -t k8s-rs/proxy-service:latest -f services/proxy-service/Dockerfile .
docker build -t k8s-rs/controller-service:latest -f services/controller-service/Dockerfile .

# Tag with version
VERSION=$(git describe --tags --always)
docker tag k8s-rs/proxy-service:latest k8s-rs/proxy-service:$VERSION
docker tag k8s-rs/controller-service:latest k8s-rs/controller-service:$VERSION

# Push to registry
docker push ghcr.io/hastur-dev/proxy-service:$VERSION
docker push ghcr.io/hastur-dev/controller-service:$VERSION
```

### Image Security

```dockerfile
# Security-focused runtime stage
FROM gcr.io/distroless/cc-debian12:nonroot AS runtime

COPY --from=builder /app/target/release/proxy-service /proxy-service

# Distroless: no shell, minimal attack surface
USER nonroot:nonroot

ENTRYPOINT ["/proxy-service"]
```

**Image scanning:**
```bash
# Trivy scan
trivy image k8s-rs/proxy-service:latest

# Grype scan
grype k8s-rs/proxy-service:latest
```

---

## Part B: Helm Chart

### Chart Structure

```text
charts/
└── k8s-rs/
    ├── Chart.yaml
    ├── values.yaml
    ├── templates/
    │   ├── _helpers.tpl
    │   ├── NOTES.txt
    │   ├── namespace.yaml
    │   ├── proxy/
    │   │   ├── deployment.yaml
    │   │   ├── service.yaml
    │   │   ├── hpa.yaml
    │   │   └── pdb.yaml
    │   ├── controller/
    │   │   ├── deployment.yaml
    │   │   ├── service.yaml
    │   │   └── rbac.yaml
    │   ├── common/
    │   │   ├── configmap.yaml
    │   │   ├── secret.yaml
    │   │   └── serviceaccount.yaml
    │   ├── crds/
    │   │   ├── proxyconfig-crd.yaml
    │   │   └── backendpool-crd.yaml
    │   └── monitoring/
    │       ├── servicemonitor.yaml
    │       └── prometheusrule.yaml
    └── crds/           # CRDs outside templates (installed separately)
        ├── proxyconfig.yaml
        └── backendpool.yaml
```

### Chart.yaml

```yaml
apiVersion: v2
name: k8s-rs
description: Kubernetes management platform with Pingora proxy
type: application
version: 0.1.0
appVersion: "0.1.0"
kubeVersion: ">=1.28.0-0"
keywords:
  - kubernetes
  - proxy
  - ingress
  - pingora
  - rust
home: https://github.com/hastur-dev/kuberentes-rs-learning
sources:
  - https://github.com/hastur-dev/kuberentes-rs-learning
maintainers:
  - name: hastur-dev
dependencies: []
```

### values.yaml

```yaml
# Global settings
global:
  imageRegistry: ghcr.io/hastur-dev
  imagePullSecrets: []

# Proxy service
proxy:
  enabled: true
  replicaCount: 2

  image:
    repository: proxy-service
    tag: ""  # Defaults to Chart appVersion
    pullPolicy: IfNotPresent

  service:
    type: LoadBalancer
    httpPort: 80
    httpsPort: 443
    annotations: {}

  ingress:
    enabled: false
    className: ""
    hosts: []
    tls: []

  resources:
    requests:
      cpu: 100m
      memory: 128Mi
    limits:
      cpu: 1000m
      memory: 512Mi

  autoscaling:
    enabled: true
    minReplicas: 2
    maxReplicas: 10
    targetCPUUtilization: 70
    targetMemoryUtilization: 80

  podDisruptionBudget:
    enabled: true
    minAvailable: 1

  nodeSelector: {}
  tolerations: []
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
        - weight: 100
          podAffinityTerm:
            labelSelector:
              matchLabels:
                app.kubernetes.io/component: proxy
            topologyKey: kubernetes.io/hostname

# Controller service
controller:
  enabled: true
  replicaCount: 2

  image:
    repository: controller-service
    tag: ""
    pullPolicy: IfNotPresent

  leaderElection:
    enabled: true
    leaseName: k8s-rs-controller
    leaseNamespace: ""  # Defaults to release namespace

  resources:
    requests:
      cpu: 50m
      memory: 64Mi
    limits:
      cpu: 500m
      memory: 256Mi

  nodeSelector: {}
  tolerations: []

# Configuration
config:
  logLevel: info
  logFormat: json

  kubernetes:
    namespace: ""  # All namespaces

  pingora:
    listenAddr: "0.0.0.0"
    httpPort: 80
    httpsPort: 443
    adminPort: 9090

  observability:
    metricsPort: 9091
    tracingEnabled: false
    otlpEndpoint: ""

# RBAC
rbac:
  create: true

# Service account
serviceAccount:
  create: true
  name: ""
  annotations: {}

# Monitoring
monitoring:
  serviceMonitor:
    enabled: false
    interval: 15s
    namespace: ""
  prometheusRule:
    enabled: false
    namespace: ""
```

### Template: Deployment

```yaml
# templates/proxy/deployment.yaml
{{- if .Values.proxy.enabled }}
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "k8s-rs.fullname" . }}-proxy
  labels:
    {{- include "k8s-rs.labels" . | nindent 4 }}
    app.kubernetes.io/component: proxy
spec:
  {{- if not .Values.proxy.autoscaling.enabled }}
  replicas: {{ .Values.proxy.replicaCount }}
  {{- end }}
  selector:
    matchLabels:
      {{- include "k8s-rs.selectorLabels" . | nindent 6 }}
      app.kubernetes.io/component: proxy
  template:
    metadata:
      labels:
        {{- include "k8s-rs.selectorLabels" . | nindent 8 }}
        app.kubernetes.io/component: proxy
      annotations:
        checksum/config: {{ include (print $.Template.BasePath "/common/configmap.yaml") . | sha256sum }}
    spec:
      serviceAccountName: {{ include "k8s-rs.serviceAccountName" . }}
      securityContext:
        runAsNonRoot: true
        runAsUser: 65534
        fsGroup: 65534
      containers:
        - name: proxy
          image: "{{ .Values.global.imageRegistry }}/{{ .Values.proxy.image.repository }}:{{ .Values.proxy.image.tag | default .Chart.AppVersion }}"
          imagePullPolicy: {{ .Values.proxy.image.pullPolicy }}
          args:
            - --config=/etc/k8s-rs/config.yaml
          ports:
            - name: http
              containerPort: {{ .Values.config.pingora.httpPort }}
            - name: https
              containerPort: {{ .Values.config.pingora.httpsPort }}
            - name: admin
              containerPort: {{ .Values.config.pingora.adminPort }}
            - name: metrics
              containerPort: {{ .Values.config.observability.metricsPort }}
          livenessProbe:
            httpGet:
              path: /healthz
              port: admin
            initialDelaySeconds: 10
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /readyz
              port: admin
            initialDelaySeconds: 5
            periodSeconds: 5
          resources:
            {{- toYaml .Values.proxy.resources | nindent 12 }}
          volumeMounts:
            - name: config
              mountPath: /etc/k8s-rs
              readOnly: true
          securityContext:
            allowPrivilegeEscalation: false
            readOnlyRootFilesystem: true
            capabilities:
              drop:
                - ALL
      volumes:
        - name: config
          configMap:
            name: {{ include "k8s-rs.fullname" . }}-config
      {{- with .Values.proxy.nodeSelector }}
      nodeSelector:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.proxy.affinity }}
      affinity:
        {{- toYaml . | nindent 8 }}
      {{- end }}
      {{- with .Values.proxy.tolerations }}
      tolerations:
        {{- toYaml . | nindent 8 }}
      {{- end }}
{{- end }}
```

### Install Chart

```bash
# Install CRDs first
kubectl apply -f charts/k8s-rs/crds/

# Install chart
helm install k8s-rs charts/k8s-rs \
  --namespace k8s-rs-system \
  --create-namespace \
  --values custom-values.yaml

# Upgrade
helm upgrade k8s-rs charts/k8s-rs \
  --namespace k8s-rs-system \
  --values custom-values.yaml

# Uninstall
helm uninstall k8s-rs --namespace k8s-rs-system
```

---

## Part C: CI/CD Pipeline

### GitHub Actions

```yaml
# .github/workflows/ci.yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  CARGO_TERM_COLOR: always
  RUSTFLAGS: "-Dwarnings"

jobs:
  check:
    name: Check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install Rust
        uses: dtolnay/rust-toolchain@stable

      - name: Cache
        uses: Swatinem/rust-cache@v2

      - name: Check formatting
        run: cargo fmt --all -- --check

      - name: Clippy
        run: cargo clippy --all-targets --all-features

      - name: Build
        run: cargo build --all-targets

      - name: Test
        run: cargo test --all

  security:
    name: Security Audit
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Install Rust
        uses: dtolnay/rust-toolchain@stable

      - name: Install cargo-audit
        run: cargo install cargo-audit

      - name: Audit
        run: cargo audit

  build-images:
    name: Build Images
    runs-on: ubuntu-latest
    needs: [check]
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    permissions:
      contents: read
      packages: write

    steps:
      - uses: actions/checkout@v4

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to GHCR
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Extract metadata
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/${{ github.repository }}/proxy-service
          tags: |
            type=sha,prefix=
            type=ref,event=branch
            type=semver,pattern={{version}}

      - name: Build and push proxy-service
        uses: docker/build-push-action@v5
        with:
          context: .
          file: services/proxy-service/Dockerfile
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

      - name: Build and push controller-service
        uses: docker/build-push-action@v5
        with:
          context: .
          file: services/controller-service/Dockerfile
          push: true
          tags: ghcr.io/${{ github.repository }}/controller-service:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  helm-lint:
    name: Helm Lint
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Helm
        uses: azure/setup-helm@v3
        with:
          version: v3.14.0

      - name: Lint chart
        run: helm lint charts/k8s-rs

      - name: Template chart
        run: helm template k8s-rs charts/k8s-rs --debug
```

### Release Workflow

```yaml
# .github/workflows/release.yaml
name: Release

on:
  push:
    tags:
      - 'v*'

jobs:
  release:
    name: Release
    runs-on: ubuntu-latest
    permissions:
      contents: write
      packages: write

    steps:
      - uses: actions/checkout@v4

      - name: Build release binaries
        run: |
          cargo build --release -p proxy-service
          cargo build --release -p controller-service

      - name: Build and push images
        # ... same as CI but with version tags

      - name: Package Helm chart
        run: |
          helm package charts/k8s-rs --version ${{ github.ref_name }}

      - name: Create GitHub Release
        uses: softprops/action-gh-release@v1
        with:
          files: |
            target/release/proxy-service
            target/release/controller-service
            k8s-rs-*.tgz
          generate_release_notes: true
```

---

## Part D: Security Hardening

### Pod Security

```yaml
# SecurityContext for pods
securityContext:
  runAsNonRoot: true
  runAsUser: 65534
  runAsGroup: 65534
  fsGroup: 65534
  seccompProfile:
    type: RuntimeDefault

# Container security context
containerSecurityContext:
  allowPrivilegeEscalation: false
  readOnlyRootFilesystem: true
  capabilities:
    drop:
      - ALL
    add:
      - NET_BIND_SERVICE  # If binding to port < 1024
```

### Network Policies

```yaml
# Allow only necessary traffic
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: k8s-rs-proxy
  namespace: k8s-rs-system
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/component: proxy
  policyTypes:
    - Ingress
    - Egress
  ingress:
    # Allow HTTP/HTTPS from anywhere
    - ports:
        - port: 80
        - port: 443
    # Allow metrics from monitoring
    - from:
        - namespaceSelector:
            matchLabels:
              name: monitoring
      ports:
        - port: 9090
        - port: 9091
  egress:
    # Allow to Kubernetes API
    - to:
        - namespaceSelector: {}
          podSelector:
            matchLabels:
              component: kube-apiserver
      ports:
        - port: 6443
    # Allow to backend services
    - to:
        - namespaceSelector: {}
      ports:
        - port: 80
        - port: 443
        - port: 8080
    # Allow DNS
    - to:
        - namespaceSelector:
            matchLabels:
              name: kube-system
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - port: 53
          protocol: UDP
```

### RBAC Best Practices

```yaml
# Minimal permissions for proxy
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: k8s-rs-proxy
rules:
  # Read-only access to Ingress/Services/Endpoints
  - apiGroups: ["networking.k8s.io"]
    resources: ["ingresses"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["services", "endpoints"]
    verbs: ["get", "list", "watch"]
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get", "list", "watch"]
    resourceNames: []  # Further restrict to specific secrets

# Separate role for controller with write access
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: k8s-rs-controller
rules:
  # Full access to our CRDs
  - apiGroups: ["k8s-rs.io"]
    resources: ["*"]
    verbs: ["*"]
  # Manage owned resources
  - apiGroups: ["apps"]
    resources: ["deployments"]
    verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
  # ... additional permissions
```

### Secrets Management

```yaml
# Use external secrets (e.g., Vault, AWS Secrets Manager)
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: k8s-rs-secrets
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: k8s-rs-secrets
    creationPolicy: Owner
  data:
    - secretKey: tls.crt
      remoteRef:
        key: k8s-rs/tls
        property: certificate
    - secretKey: tls.key
      remoteRef:
        key: k8s-rs/tls
        property: private_key
```

---

## Part E: Production Checklist

### Pre-Production Verification

| Category | Check | Status |
|----------|-------|--------|
| **Images** | Built with multi-stage Dockerfile | ☐ |
| | Using distroless/minimal base | ☐ |
| | No secrets in image layers | ☐ |
| | Image scanning passed | ☐ |
| **Security** | Running as non-root | ☐ |
| | Read-only filesystem | ☐ |
| | Capabilities dropped | ☐ |
| | Network policies applied | ☐ |
| | RBAC is minimal | ☐ |
| **Reliability** | Resource limits set | ☐ |
| | Liveness/readiness probes | ☐ |
| | Pod Disruption Budget | ☐ |
| | Anti-affinity rules | ☐ |
| | HPA configured | ☐ |
| **Observability** | Metrics exposed | ☐ |
| | Dashboards created | ☐ |
| | Alerts configured | ☐ |
| | Logs are structured | ☐ |
| | Tracing enabled | ☐ |
| **Operations** | Helm chart tested | ☐ |
| | Upgrade tested | ☐ |
| | Rollback tested | ☐ |
| | Backup/restore documented | ☐ |

### Runbook Topics

Document procedures for:

1. **Installation** - Fresh install steps
2. **Upgrade** - Version upgrade procedure
3. **Rollback** - How to rollback failed upgrade
4. **Scaling** - Manual and automatic scaling
5. **TLS Certificate Rotation** - Cert renewal process
6. **Troubleshooting** - Common issues and solutions
7. **Disaster Recovery** - Recovery from failures
8. **On-Call Procedures** - Alert response playbooks

---

## Part F: Performance Tuning

### Resource Sizing Guide

| Component | Small | Medium | Large |
|-----------|-------|--------|-------|
| **Proxy** | | | |
| Replicas | 2 | 3-5 | 5-10+ |
| CPU request | 100m | 500m | 1000m |
| CPU limit | 500m | 2000m | 4000m |
| Memory request | 128Mi | 256Mi | 512Mi |
| Memory limit | 256Mi | 512Mi | 1Gi |
| **Controller** | | | |
| Replicas | 2 | 2 | 2-3 |
| CPU request | 50m | 100m | 200m |
| CPU limit | 200m | 500m | 1000m |
| Memory request | 64Mi | 128Mi | 256Mi |
| Memory limit | 128Mi | 256Mi | 512Mi |

### Tuning Parameters

```yaml
# Proxy tuning
pingora:
  # Connection pool
  pool:
    maxConnections: 10000
    idleTimeout: 60s

  # Worker threads (defaults to CPU count)
  workers: 4

  # Request limits
  limits:
    maxRequestBodyBytes: 104857600  # 100MB
    requestTimeout: 300s
    keepaliveTimeout: 75s

# Controller tuning
controller:
  # Reconciliation
  maxConcurrentReconciles: 10
  rateLimiter:
    baseDelay: 5s
    maxDelay: 300s
    qps: 50
    burst: 100
```

---

## Verification Checklist

Project completion verification:

| Check | Command | Expected |
|-------|---------|----------|
| All tests pass | `cargo test --all` | Green |
| No clippy warnings | `cargo clippy -- -D warnings` | Clean |
| Images build | `docker build ...` | Success |
| Images scan clean | `trivy image ...` | No critical CVEs |
| Helm lint passes | `helm lint charts/k8s-rs` | No errors |
| Helm install works | `helm install ...` | Deployed |
| Health checks pass | `curl /healthz` | 200 OK |
| Metrics available | `curl /metrics` | Prometheus format |
| CI pipeline green | Check GitHub Actions | All jobs pass |

---

## Summary

You now have a complete production-ready Kubernetes management platform:

```text
kubernetes-rs-learning
├── Rust-based Kubernetes client (kube-rs)
├── Pingora proxy (replacing Nginx)
├── Custom controllers and operators
├── Full Ingress controller implementation
├── Comprehensive observability
├── Production Helm chart
├── CI/CD pipelines
└── Security hardening
```

### What's Next?

Consider these advanced topics:

1. **Multi-cluster support** - Federated management
2. **Service mesh integration** - Istio/Linkerd compatibility
3. **GitOps** - ArgoCD/Flux integration
4. **Advanced traffic management** - Canary, blue-green deployments
5. **WebAssembly filters** - Pingora WASM plugin support

---

## References

- [Docker Multi-Stage Builds](https://docs.docker.com/build/building/multi-stage/)
- [Helm Best Practices](https://helm.sh/docs/chart_best_practices/)
- [Kubernetes Security](https://kubernetes.io/docs/concepts/security/)
- [GitHub Actions for Rust](https://github.com/actions-rs)
- [Production Kubernetes](https://kubernetes.io/docs/setup/production-environment/)
