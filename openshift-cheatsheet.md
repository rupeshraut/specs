# 🎩 Red Hat OpenShift Cheat Sheet

> **Purpose:** Production patterns for deploying and operating Java services on OpenShift — builds, deployments, routes, security contexts, and OpenShift-specific features beyond vanilla Kubernetes.
> **Stack context:** Java 21+ / Spring Boot 3.x / OpenShift 4.x / OKD / oc CLI

---

## 📋 OpenShift vs Vanilla Kubernetes — Key Differences

| Feature | Kubernetes | OpenShift |
|---------|-----------|-----------|
| CLI tool | `kubectl` | `oc` (superset of kubectl) |
| Container registry | External (ECR, GCR) | Built-in registry |
| Builds | External (CI/CD) | Built-in S2I, BuildConfig |
| Routes/Ingress | Ingress + controller | **Route** (native, TLS built-in) |
| Security | Permissive by default | **SCC** (restricted by default) |
| Container user | Root allowed | **Non-root enforced** (UID range) |
| Projects | Namespaces | **Projects** (namespace + RBAC) |
| Image streams | N/A | **ImageStream** (tag tracking, triggers) |
| Monitoring | Install Prometheus | Built-in monitoring stack |
| Service mesh | Install Istio | **OpenShift Service Mesh** (managed) |

---

## 🏗️ Pattern 1: Dockerfile for OpenShift

### OpenShift Security Constraints

```
OpenShift runs containers as a RANDOM UID from a namespace-assigned range.
This means:
  ❌ Cannot assume UID 0 (root)
  ❌ Cannot assume a specific UID (like 1000)
  ❌ Cannot bind to ports < 1024
  ✅ Must work with ANY UID in the assigned range
  ✅ Group is always root (GID 0)
  ✅ Use GID 0 for file permissions
```

### OpenShift-Compatible Dockerfile

```dockerfile
# ══════════════════════════════════════════
# Stage 1: Build
# ══════════════════════════════════════════
FROM registry.access.redhat.com/ubi9/openjdk-21:latest AS build
WORKDIR /app
USER 0

COPY gradle/ gradle/
COPY gradlew build.gradle.kts settings.gradle.kts gradle.properties ./
RUN ./gradlew --no-daemon dependencies || true

COPY src/ src/
RUN ./gradlew --no-daemon bootJar -x test

# ══════════════════════════════════════════
# Stage 2: Runtime (OpenShift compatible)
# ══════════════════════════════════════════
FROM registry.access.redhat.com/ubi9/openjdk-21-runtime:latest AS runtime

# Labels for OpenShift catalog and metadata
LABEL io.openshift.tags="java,spring-boot,payment-service" \
      io.k8s.description="Payment processing service" \
      io.openshift.expose-services="8080:http,8081:management" \
      maintainer="platform-team@example.com"

# OpenShift runs with random UID but GID=0
# All files must be group-readable/writable by root group
WORKDIR /app

COPY --from=build --chown=1001:0 /app/build/libs/*.jar app.jar

# Ensure directories are writable by root group (GID 0)
# OpenShift assigns random UID but keeps GID 0
RUN chmod -R g+rwX /app && \
    mkdir -p /tmp/hsperfdata && chmod -R g+rwX /tmp

ENV JAVA_OPTS="\
  -XX:+UseContainerSupport \
  -XX:MaxRAMPercentage=75.0 \
  -XX:InitialRAMPercentage=50.0 \
  -XX:+UseZGC -XX:+ZGenerational \
  -XX:+ExitOnOutOfMemoryError \
  -XX:+HeapDumpOnOutOfMemoryError \
  -XX:HeapDumpPath=/tmp/heapdump.hprof \
  -Djava.security.egd=file:/dev/./urandom"

# Do NOT set USER to specific UID — OpenShift assigns one
# USER 1001  ← Don't hardcode; OpenShift overrides this anyway

EXPOSE 8080 8081

ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

### UBI (Universal Base Image) Selection

| Image | Size | Use Case |
|---|---|---|
| `ubi9/openjdk-21-runtime` | ~250MB | **Recommended** — Red Hat supported, FIPS-ready |
| `ubi9-minimal` + manual JRE | ~120MB | Minimal footprint, more control |
| `ubi9/openjdk-21` | ~400MB | Full JDK (for builds, JFR tools) |

---

## 📦 Pattern 2: Source-to-Image (S2I) Build

> **When:** You want OpenShift to build images from source without a Dockerfile.

```yaml
# BuildConfig — S2I from Git repository
apiVersion: build.openshift.io/v1
kind: BuildConfig
metadata:
  name: payment-service
spec:
  source:
    type: Git
    git:
      uri: https://github.com/example/payment-service.git
      ref: main
    contextDir: /
  strategy:
    type: Source
    sourceStrategy:
      from:
        kind: ImageStreamTag
        namespace: openshift
        name: java:21    # OpenShift built-in Java S2I builder
      env:
        - name: JAVA_OPTS_APPEND
          value: "-XX:+UseZGC -XX:+ZGenerational"
        - name: GRADLE_ARGS
          value: "bootJar -x test"
  output:
    to:
      kind: ImageStreamTag
      name: payment-service:latest
  triggers:
    - type: ConfigChange
    - type: ImageChange
    - type: GitHub
      github:
        secret: webhook-secret
  resources:
    limits:
      memory: "2Gi"
      cpu: "2"
```

### Docker Strategy Build (Preferred for Complex Builds)

```yaml
apiVersion: build.openshift.io/v1
kind: BuildConfig
metadata:
  name: payment-service
spec:
  source:
    type: Git
    git:
      uri: https://github.com/example/payment-service.git
      ref: main
  strategy:
    type: Docker
    dockerStrategy:
      dockerfilePath: Dockerfile
      buildArgs:
        - name: JAVA_VERSION
          value: "21"
  output:
    to:
      kind: ImageStreamTag
      name: payment-service:latest
  resources:
    limits:
      memory: "2Gi"
      cpu: "2"
```

---

## ☸️ Pattern 3: DeploymentConfig & Deployment

### Modern Deployment (Preferred — Kubernetes-native)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
  labels:
    app: payment-service
    app.kubernetes.io/name: payment-service
    app.kubernetes.io/version: "2.1.0"
    app.kubernetes.io/part-of: payment-platform
    app.openshift.io/runtime: java
spec:
  replicas: 3
  revisionHistoryLimit: 5
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  selector:
    matchLabels:
      app: payment-service
  template:
    metadata:
      labels:
        app: payment-service
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8081"
        prometheus.io/path: "/actuator/prometheus"
    spec:
      serviceAccountName: payment-service
      terminationGracePeriodSeconds: 45

      # ── OpenShift Security Context ──
      securityContext:
        runAsNonRoot: true
        # Don't set runAsUser — let OpenShift assign from range
        seccompProfile:
          type: RuntimeDefault

      containers:
        - name: payment-service
          image: image-registry.openshift-image-registry.svc:5000/payment-ns/payment-service:2.1.0
          imagePullPolicy: IfNotPresent

          securityContext:
            allowPrivilegeEscalation: false
            capabilities:
              drop: ["ALL"]
            readOnlyRootFilesystem: true

          ports:
            - name: http
              containerPort: 8080
            - name: management
              containerPort: 8081

          resources:
            requests:
              memory: "512Mi"
              cpu: "250m"
            limits:
              memory: "1Gi"
              cpu: "1000m"

          env:
            - name: ENVIRONMENT
              value: "production"
            - name: MONGODB_URI
              valueFrom:
                secretKeyRef:
                  name: payment-secrets
                  key: mongodb-uri
            - name: KAFKA_BROKERS
              valueFrom:
                configMapKeyRef:
                  name: payment-config
                  key: kafka-brokers

          startupProbe:
            httpGet:
              path: /actuator/health/liveness
              port: management
            initialDelaySeconds: 10
            periodSeconds: 5
            failureThreshold: 30

          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: management
            periodSeconds: 10
            failureThreshold: 3

          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: management
            periodSeconds: 5
            failureThreshold: 3

          volumeMounts:
            - name: tmp
              mountPath: /tmp
            - name: app-config
              mountPath: /app/config
              readOnly: true

      volumes:
        - name: tmp
          emptyDir: {}
        - name: app-config
          configMap:
            name: payment-config

      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: payment-service
```

---

## 🌐 Pattern 4: Routes (OpenShift-Specific)

### TLS Edge-Terminated Route

```yaml
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: payment-service
  labels:
    app: payment-service
  annotations:
    # Rate limiting (HAProxy)
    haproxy.router.openshift.io/rate-limit-connections: "true"
    haproxy.router.openshift.io/rate-limit-connections.concurrent-tcp: "100"
    haproxy.router.openshift.io/rate-limit-connections.rate-http: "50"
    # Timeouts
    haproxy.router.openshift.io/timeout: 30s
    # IP whitelisting (optional)
    # haproxy.router.openshift.io/ip_whitelist: "10.0.0.0/8 172.16.0.0/12"
spec:
  host: payment-api.apps.cluster.example.com
  path: /api/v1
  to:
    kind: Service
    name: payment-service
    weight: 100
  port:
    targetPort: http
  tls:
    termination: edge
    insecureEdgeTerminationPolicy: Redirect    # HTTP → HTTPS redirect
    certificate: |
      -----BEGIN CERTIFICATE-----
      ...
      -----END CERTIFICATE-----
    key: |
      -----BEGIN RSA PRIVATE KEY-----
      ...
      -----END RSA PRIVATE KEY-----

---
# Internal management route (no external exposure)
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: payment-service-mgmt
  annotations:
    haproxy.router.openshift.io/ip_whitelist: "10.0.0.0/8"   # Internal only
spec:
  host: payment-mgmt.apps.internal.example.com
  to:
    kind: Service
    name: payment-service
  port:
    targetPort: management
  tls:
    termination: edge
```

### Canary / A-B Routing

```yaml
# Route with traffic split between stable and canary
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: payment-service
spec:
  host: payment-api.apps.cluster.example.com
  to:
    kind: Service
    name: payment-service-stable
    weight: 90
  alternateBackends:
    - kind: Service
      name: payment-service-canary
      weight: 10
  port:
    targetPort: http
  tls:
    termination: edge
```

---

## 🔐 Pattern 5: Security — SCC & RBAC

### Security Context Constraints (SCC)

```
OpenShift SCC hierarchy (most restrictive → least):
  restricted-v2   ← DEFAULT for all pods (use this)
  restricted
  nonroot-v2
  nonroot
  hostnetwork-v2
  hostaccess
  anyuid           ← Avoid unless absolutely necessary
  privileged       ← Never for application pods

The 'restricted-v2' SCC enforces:
  ✅ Run as non-root
  ✅ Random UID assignment
  ✅ No privilege escalation
  ✅ Drop all Linux capabilities
  ✅ Seccomp profile = RuntimeDefault
  ✅ Read-only root filesystem supported
```

### Service Account & RBAC

```yaml
# Service account for the payment service
apiVersion: v1
kind: ServiceAccount
metadata:
  name: payment-service
  labels:
    app: payment-service

---
# Role with minimum permissions
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: payment-service-role
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "watch"]
  - apiGroups: [""]
    resources: ["secrets"]
    verbs: ["get"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: payment-service-binding
subjects:
  - kind: ServiceAccount
    name: payment-service
roleRef:
  kind: Role
  name: payment-service-role
  apiGroup: rbac.authorization.k8s.io
```

### Network Policy (Microsegmentation)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: payment-service-policy
spec:
  podSelector:
    matchLabels:
      app: payment-service
  policyTypes:
    - Ingress
    - Egress
  ingress:
    # Allow from OpenShift router
    - from:
        - namespaceSelector:
            matchLabels:
              network.openshift.io/policy-group: ingress
      ports:
        - port: 8080
    # Allow from monitoring
    - from:
        - namespaceSelector:
            matchLabels:
              name: openshift-monitoring
      ports:
        - port: 8081
    # Allow from within same namespace
    - from:
        - podSelector: {}
      ports:
        - port: 8080
        - port: 8081
  egress:
    # MongoDB
    - to:
        - namespaceSelector:
            matchLabels:
              name: mongodb-ns
      ports:
        - port: 27017
    # Kafka
    - to:
        - namespaceSelector:
            matchLabels:
              name: kafka-ns
      ports:
        - port: 9092
        - port: 9093
    # DNS
    - to: []
      ports:
        - port: 53
          protocol: UDP
        - port: 53
          protocol: TCP
```

---

## 📊 Pattern 6: Monitoring & Observability

### ServiceMonitor (Prometheus Integration)

```yaml
# OpenShift Monitoring automatically discovers ServiceMonitors
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: payment-service
  labels:
    app: payment-service
spec:
  selector:
    matchLabels:
      app: payment-service
  endpoints:
    - port: management
      path: /actuator/prometheus
      interval: 15s
      scheme: http
  namespaceSelector:
    matchNames:
      - payment-ns
```

### PrometheusRule (Alerts)

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PrometheusRule
metadata:
  name: payment-service-alerts
  labels:
    app: payment-service
spec:
  groups:
    - name: payment-service.rules
      rules:
        - alert: PaymentServiceDown
          expr: up{job="payment-service"} == 0
          for: 1m
          labels:
            severity: critical
          annotations:
            summary: "Payment service instance {{ $labels.instance }} is DOWN"

        - alert: HighErrorRate
          expr: >
            sum(rate(http_server_requests_seconds_count{job="payment-service",status=~"5.."}[5m]))
            / sum(rate(http_server_requests_seconds_count{job="payment-service"}[5m])) > 0.05
          for: 2m
          labels:
            severity: critical

        - alert: HighP99Latency
          expr: >
            histogram_quantile(0.99,
              sum(rate(http_server_requests_seconds_bucket{job="payment-service"}[5m])) by (le))
            > 2
          for: 5m
          labels:
            severity: warning
```

---

## 🖼️ Pattern 7: ImageStream & Triggers

```yaml
# ImageStream — tracks image tags, enables triggers
apiVersion: image.openshift.io/v1
kind: ImageStream
metadata:
  name: payment-service
spec:
  lookupPolicy:
    local: true     # Allow local name resolution
  tags:
    - name: latest
      from:
        kind: DockerImage
        name: registry.example.com/payment-service:latest
      importPolicy:
        scheduled: true    # Periodically check for new image
        importMode: PreserveOriginal
    - name: stable
      from:
        kind: ImageStreamTag
        name: payment-service:2.1.0
```

---

## ⚡ Pattern 8: oc CLI Quick Reference

```bash
# ── Project management ──
oc new-project payment-platform         # Create project (namespace + RBAC)
oc project payment-platform             # Switch project
oc get projects                         # List all projects

# ── Deployment ──
oc apply -f deployment.yaml             # Apply manifests
oc rollout status deployment/payment-service    # Watch rollout
oc rollout undo deployment/payment-service      # Rollback
oc rollout history deployment/payment-service   # History

# ── Scaling ──
oc scale deployment/payment-service --replicas=5
oc autoscale deployment/payment-service --min=3 --max=10 --cpu-percent=70

# ── Debugging ──
oc get pods -l app=payment-service
oc logs -f deployment/payment-service --since=5m
oc logs -f deployment/payment-service -c payment-service | grep ERROR
oc rsh <pod-name>                       # Shell into pod
oc exec <pod-name> -- jcmd 1 Thread.print      # Thread dump
oc exec <pod-name> -- jcmd 1 GC.heap_dump /tmp/heap.hprof
oc cp <pod-name>:/tmp/heap.hprof ./heap.hprof  # Copy heap dump out

# ── Port forwarding ──
oc port-forward svc/payment-service 8080:8080   # App
oc port-forward svc/payment-service 8081:8081   # Actuator

# ── Routes ──
oc get routes
oc expose svc/payment-service --port=8080       # Create route
oc create route edge --service=payment-service   # TLS route

# ── Secrets ──
oc create secret generic payment-secrets \
  --from-literal=mongodb-uri="mongodb://admin:pass@host:27017/db"
oc set env deployment/payment-service --from=secret/payment-secrets

# ── Builds ──
oc start-build payment-service                   # Trigger build
oc start-build payment-service --from-dir=.      # Build from local directory
oc logs -f bc/payment-service                    # Watch build logs

# ── Resource usage ──
oc adm top pods -l app=payment-service           # CPU/memory usage
oc describe node <node-name>                     # Node resource summary

# ── Events ──
oc get events --sort-by=.lastTimestamp | head -20
```

---

## 🔄 Pattern 9: CI/CD Pipeline (Tekton/OpenShift Pipelines)

```yaml
# Simple pipeline: build → test → deploy
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: payment-service-pipeline
spec:
  params:
    - name: git-url
    - name: git-revision
      default: main
    - name: image-name
      default: image-registry.openshift-image-registry.svc:5000/payment-ns/payment-service

  workspaces:
    - name: shared-workspace

  tasks:
    - name: fetch-source
      taskRef:
        name: git-clone
        kind: ClusterTask
      workspaces:
        - name: output
          workspace: shared-workspace
      params:
        - name: url
          value: $(params.git-url)
        - name: revision
          value: $(params.git-revision)

    - name: build-and-test
      taskRef:
        name: gradle
      runAfter: ["fetch-source"]
      workspaces:
        - name: source
          workspace: shared-workspace
      params:
        - name: TASKS
          value: ["build", "test"]

    - name: build-image
      taskRef:
        name: buildah
        kind: ClusterTask
      runAfter: ["build-and-test"]
      workspaces:
        - name: source
          workspace: shared-workspace
      params:
        - name: IMAGE
          value: $(params.image-name):$(params.git-revision)

    - name: deploy
      taskRef:
        name: openshift-client
        kind: ClusterTask
      runAfter: ["build-image"]
      params:
        - name: SCRIPT
          value: |
            oc set image deployment/payment-service \
              payment-service=$(params.image-name):$(params.git-revision)
            oc rollout status deployment/payment-service --timeout=300s
```

---

## 🚫 OpenShift Anti-Patterns

| Anti-Pattern | Why It's Dangerous | Fix |
|---|---|---|
| **Running as root** | Violates SCC, security risk | Use UBI images, GID 0 permissions |
| **Hardcoded UID in Dockerfile** | OpenShift assigns random UID | Don't set USER to specific UID |
| **Using `anyuid` SCC** | Bypasses security controls | Fix the image to work with restricted-v2 |
| **No resource limits** | Pod eviction, noisy neighbor | Always set requests and limits |
| **Secrets in ConfigMaps** | Not encrypted, visible in logs | Use Secrets or External Secrets |
| **No NetworkPolicy** | All pods can talk to all pods | Microsegmentation per service |
| **DeploymentConfig** (legacy) | OpenShift-specific, not portable | Use Kubernetes Deployment |
| **Ignoring readOnlyRootFilesystem** | Writable root = attack surface | Mount /tmp as emptyDir |
| **No ServiceMonitor** | Missing from OpenShift monitoring | Always create ServiceMonitor |
| **Route without TLS** | Traffic in plain text | Always use edge or reencrypt TLS |

---

## 💡 Golden Rules of OpenShift

```
1.  UBI base images — Red Hat supported, vulnerability-scanned, FIPS-ready.
2.  GID 0 permissions — OpenShift uses random UID but GID is always 0.
3.  restricted-v2 SCC is your friend — don't fight it, design for it.
4.  Routes > Ingress — use OpenShift-native routing with built-in TLS.
5.  NetworkPolicy for every service — microsegmentation is free security.
6.  ServiceMonitor for every service — integrate with OpenShift monitoring stack.
7.  Kubernetes Deployment > DeploymentConfig — portability, better ecosystem support.
8.  ImageStreams for tag tracking — automatic rollouts on new image pushes.
9.  readOnlyRootFilesystem + emptyDir for /tmp — minimize attack surface.
10. oc debug <pod> for troubleshooting — runs a debug pod with same security context.
```

---

*Last updated: February 2026 | Stack: Java 21+ / Spring Boot 3.x / OpenShift 4.x / UBI 9*
