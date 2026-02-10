# 🐳 Docker & Kubernetes Patterns Cheat Sheet

> **Purpose:** Production container patterns for Java services — Dockerfiles, K8s manifests, health probes, resource management, and deployment strategies.
> **Stack context:** Java 21+ / Spring Boot 3.x / Docker / Kubernetes

---

## 🏗️ Pattern 1: Multi-Stage Dockerfile

```dockerfile
# ══════════════════════════════════════════
# Stage 1: Build
# ══════════════════════════════════════════
FROM eclipse-temurin:21-jdk-alpine AS build
WORKDIR /app

# Cache Gradle wrapper
COPY gradle/ gradle/
COPY gradlew build.gradle.kts settings.gradle.kts gradle.properties ./
RUN ./gradlew --no-daemon dependencies || true   # Cache deps layer

# Build application
COPY src/ src/
RUN ./gradlew --no-daemon bootJar -x test \
    && java -Djarmode=tools -jar build/libs/*.jar extract --layers --launcher

# ══════════════════════════════════════════
# Stage 2: Runtime (minimal image)
# ══════════════════════════════════════════
FROM eclipse-temurin:21-jre-alpine AS runtime

# Security: non-root user
RUN addgroup -S app && adduser -S app -G app
WORKDIR /app

# Copy layered jar (best Docker cache utilization)
COPY --from=build /app/build/libs/*/dependencies/ ./
COPY --from=build /app/build/libs/*/spring-boot-loader/ ./
COPY --from=build /app/build/libs/*/snapshot-dependencies/ ./
COPY --from=build /app/build/libs/*/application/ ./

# JVM configuration
ENV JAVA_OPTS="\
  -XX:+UseContainerSupport \
  -XX:MaxRAMPercentage=75.0 \
  -XX:InitialRAMPercentage=50.0 \
  -XX:+UseZGC -XX:+ZGenerational \
  -XX:+ExitOnOutOfMemoryError \
  -XX:+HeapDumpOnOutOfMemoryError \
  -XX:HeapDumpPath=/tmp/heapdump.hprof \
  -Djava.security.egd=file:/dev/./urandom"

USER app
EXPOSE 8080 8081

HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD wget -qO- http://localhost:8081/actuator/health/liveness || exit 1

ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS org.springframework.boot.loader.launch.JarLauncher"]
```

### Image Size Comparison

| Base Image | Size | Use Case |
|---|---|---|
| `eclipse-temurin:21-jdk` | ~450MB | Build stage only |
| `eclipse-temurin:21-jre-alpine` | ~120MB | **Recommended runtime** |
| `eclipse-temurin:21-jre` | ~250MB | When Alpine has musl issues |
| Custom JLink | ~60MB | Maximum optimization |

---

## ☸️ Pattern 2: Kubernetes Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
  labels:
    app: payment-service
    version: "1.0.0"
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
      
      containers:
        - name: payment-service
          image: registry.example.com/payment-service:1.0.0
          imagePullPolicy: IfNotPresent

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
            - name: JAVA_OPTS
              value: "-XX:MaxRAMPercentage=75 -XX:+UseZGC -XX:+ZGenerational"
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

      volumes:
        - name: tmp
          emptyDir: {}

      topologySpreadConstraints:
        - maxSkew: 1
          topologyKey: topology.kubernetes.io/zone
          whenUnsatisfiable: DoNotSchedule
          labelSelector:
            matchLabels:
              app: payment-service
```

---

## 🔐 Pattern 3: ConfigMap & Secrets

```yaml
# ── ConfigMap for non-sensitive config ──
apiVersion: v1
kind: ConfigMap
metadata:
  name: payment-config
data:
  kafka-brokers: "kafka-0:9092,kafka-1:9092,kafka-2:9092"
  spring-profiles: "prod"
  log-level: "INFO"

---
# ── Secret for sensitive data ──
apiVersion: v1
kind: Secret
metadata:
  name: payment-secrets
type: Opaque
data:
  mongodb-uri: bW9uZ29kYjovL2FkbWluOnBhc3N3b3JkQGhvc3Q=   # base64 encoded

---
# ── External Secrets Operator (preferred for production) ──
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: payment-secrets
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: ClusterSecretStore
  target:
    name: payment-secrets
  data:
    - secretKey: mongodb-uri
      remoteRef:
        key: secret/data/payment-service
        property: mongodb-uri
```

---

## 📊 Pattern 4: Resource Management

### Resource Sizing Guide

```
MEMORY FORMULA:
  Container limit = JVM heap + non-heap overhead + safety margin
  
  JVM heap:        75% of container limit  (MaxRAMPercentage=75)
  Metaspace:       64-128MB
  Thread stacks:   threads × 512KB
  Direct buffers:  64-128MB
  Code cache:      48-64MB
  GC overhead:     3-10% of heap
  Native/OS:       50MB
  
  EXAMPLE (1Gi container):
    Heap:     768MB (75%)
    Non-heap: ~250MB
    Total:    ~1018MB ✅ fits in 1Gi

CPU GUIDELINES:
  requests: What the app needs during normal operation
  limits:   What the app needs during spikes
  
  Typical ratios:
    requests: 250m, limits: 1000m  (burst up to 4x)
    requests: 500m, limits: 2000m  (burst up to 4x)
  
  ⚠️ Setting CPU limits too low causes throttling
     Setting no CPU limits risks noisy neighbor
```

### Horizontal Pod Autoscaler

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: payment-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: payment-service
  minReplicas: 3
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Pods
      pods:
        metric:
          name: http_server_requests_per_second
        target:
          type: AverageValue
          averageValue: "100"
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
        - type: Pods
          value: 2
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Pods
          value: 1
          periodSeconds: 120
```

---

## 🔄 Pattern 5: Deployment Strategies

```
ROLLING UPDATE (default):
  Old pods removed one at a time, new pods added
  ✅ Zero downtime, simple
  ❌ Mixed versions temporarily

BLUE-GREEN:
  Full new deployment alongside old, switch traffic at once
  ✅ Instant rollback, no mixed versions
  ❌ Double resources during deployment

CANARY:
  Route small % of traffic to new version, gradually increase
  ✅ Risk mitigation, real traffic validation
  ❌ Complex routing setup
```

### Canary with Istio

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: payment-service
spec:
  hosts:
    - payment-service
  http:
    - route:
        - destination:
            host: payment-service
            subset: stable
          weight: 90
        - destination:
            host: payment-service
            subset: canary
          weight: 10
```

---

## 🚫 Docker/K8s Anti-Patterns

| Anti-Pattern | Fix |
|---|---|
| Running as root | `USER app` in Dockerfile |
| No resource limits | Always set requests AND limits |
| Liveness checks external deps | Liveness = JVM only, readiness = deps |
| Secrets in ConfigMap | Use Secrets or External Secrets Operator |
| No terminationGracePeriodSeconds | Set > Spring shutdown timeout |
| Fixed -Xmx in containers | Use MaxRAMPercentage (container-aware) |
| Single replica in prod | Minimum 3 replicas across zones |
| No topology spread | topologySpreadConstraints for AZ distribution |
| Latest tag | Pin image versions, use SHA digests in prod |

---

## 💡 Golden Rules

```
1.  MULTI-STAGE builds — build in JDK, run in JRE-alpine.
2.  NON-ROOT user always — security best practice.
3.  LAYERED jars — maximize Docker cache hits.
4.  SEPARATE management port — probes on 8081, app on 8080.
5.  terminationGracePeriodSeconds > shutdown timeout — give Spring time to drain.
6.  MaxRAMPercentage, not -Xmx — container-aware memory management.
7.  REQUESTS = normal, LIMITS = burst — don't over-provision or under-provision.
8.  3+ REPLICAS across zones — survive zone failures.
9.  EXTERNAL SECRETS for production — never hardcode credentials.
10. PIN image versions — reproducible deployments.
```

---

*Last updated: February 2026 | Stack: Java 21+ / Docker / Kubernetes / Spring Boot 3.x*
