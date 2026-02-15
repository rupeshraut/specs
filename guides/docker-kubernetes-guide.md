# Docker & Kubernetes for Java Services

A comprehensive guide to containerizing, deploying, and operating Java applications on Docker and Kubernetes with production-ready patterns.

---

## Table of Contents

1. [Docker Fundamentals](#docker-fundamentals)
2. [Dockerfile Best Practices](#dockerfile-best-practices)
3. [Multi-Stage Builds](#multi-stage-builds)
4. [Image Optimization](#image-optimization)
5. [Java Container Configuration](#java-container-configuration)
6. [Kubernetes Fundamentals](#kubernetes-fundamentals)
7. [Deployment Strategies](#deployment-strategies)
8. [Health Probes](#health-probes)
9. [Resource Management](#resource-management)
10. [ConfigMaps & Secrets](#configmaps--secrets)
11. [Persistent Storage](#persistent-storage)
12. [Service Mesh](#service-mesh)
13. [Monitoring & Logging](#monitoring--logging)
14. [Security](#security)
15. [Production Patterns](#production-patterns)

---

## Docker Fundamentals

### Why Containerize Java Applications?

```
Benefits:
✅ Consistent environments (dev = staging = prod)
✅ Faster deployments
✅ Resource isolation
✅ Easy scaling
✅ Immutable infrastructure

Challenges:
⚠️  Larger image sizes (Java runtime)
⚠️  Startup time
⚠️  Memory management
⚠️  Layer optimization
```

---

## Multi-Stage Builds

### Optimal Java Dockerfile

```dockerfile
# Stage 1: Build with Gradle
FROM gradle:8.5-jdk21 AS builder

WORKDIR /app

# Copy only dependency files first (cache layer)
COPY build.gradle settings.gradle ./
COPY gradle/ ./gradle/

# Download dependencies (cached unless build files change)
RUN gradle dependencies --no-daemon

# Copy source code
COPY src/ ./src/

# Build application
RUN gradle clean build -x test --no-daemon && \
    java -Djarmode=layertools -jar build/libs/*.jar extract

# Stage 2: Runtime with optimized JRE
FROM eclipse-temurin:21-jre-alpine

# Create non-root user
RUN addgroup -S spring && adduser -S spring -G spring

# Install dumb-init for proper signal handling
RUN apk add --no-cache dumb-init

WORKDIR /app

# Copy extracted layers
COPY --from=builder /app/dependencies/ ./
COPY --from=builder /app/spring-boot-loader/ ./
COPY --from=builder /app/snapshot-dependencies/ ./
COPY --from=builder /app/application/ ./

# Set ownership
RUN chown -R spring:spring /app

USER spring:spring

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=40s --retries=3 \
    CMD wget --no-verbose --tries=1 --spider http://localhost:8080/actuator/health || exit 1

# JVM options
ENV JAVA_OPTS="-XX:+UseG1GC \
    -XX:MaxRAMPercentage=75.0 \
    -XX:+UseContainerSupport \
    -XX:+ExitOnOutOfMemoryError \
    -Djava.security.egd=file:/dev/./urandom"

EXPOSE 8080

ENTRYPOINT ["dumb-init", "--"]
CMD ["sh", "-c", "java $JAVA_OPTS org.springframework.boot.loader.JarLauncher"]
```

### With Maven

```dockerfile
# Stage 1: Build
FROM maven:3.9-eclipse-temurin-21 AS builder

WORKDIR /app

# Copy POM files
COPY pom.xml ./
COPY .mvn/ ./.mvn/

# Download dependencies
RUN mvn dependency:go-offline -B

# Copy source and build
COPY src/ ./src/
RUN mvn clean package -DskipTests && \
    java -Djarmode=layertools -jar target/*.jar extract

# Stage 2: Runtime
FROM eclipse-temurin:21-jre-alpine

RUN addgroup -S spring && adduser -S spring -G spring && \
    apk add --no-cache dumb-init

WORKDIR /app

COPY --from=builder --chown=spring:spring /app/dependencies/ ./
COPY --from=builder --chown=spring:spring /app/spring-boot-loader/ ./
COPY --from=builder --chown=spring:spring /app/snapshot-dependencies/ ./
COPY --from=builder --chown=spring:spring /app/application/ ./

USER spring:spring

ENV JAVA_OPTS="-XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0"

EXPOSE 8080

ENTRYPOINT ["dumb-init", "--"]
CMD ["sh", "-c", "java $JAVA_OPTS org.springframework.boot.loader.JarLauncher"]
```

---

## Image Optimization

### Layer Caching Strategy

```dockerfile
# ✅ GOOD: Layers ordered by change frequency
COPY build.gradle settings.gradle ./     # Rarely changes
RUN gradle dependencies                   # Cached
COPY src/ ./src/                         # Changes often
RUN gradle build                         # Only if source changed

# ❌ BAD: Invalidates cache on every build
COPY . ./
RUN gradle build
```

### Distroless Images

```dockerfile
# Smallest possible image
FROM gcr.io/distroless/java21-debian12

COPY --from=builder /app /app

WORKDIR /app

ENV JAVA_OPTS="-XX:+UseContainerSupport"

CMD ["org.springframework.boot.loader.JarLauncher"]
```

---

## Java Container Configuration

### JVM Memory Settings

```yaml
# application.yml
spring:
  application:
    name: payment-service

# Environment variables in Dockerfile or K8s
ENV JAVA_OPTS=" \
    -XX:+UseContainerSupport \
    -XX:MaxRAMPercentage=75.0 \
    -XX:InitialRAMPercentage=50.0 \
    -XX:MinRAMPercentage=25.0 \
    -XX:+UseG1GC \
    -XX:MaxGCPauseMillis=200 \
    -XX:+ExitOnOutOfMemoryError \
    -XX:+HeapDumpOnOutOfMemoryError \
    -XX:HeapDumpPath=/tmp/heapdump.hprof"
```

### Container-Aware Settings

```properties
# Automatically detect container memory limits
-XX:+UseContainerSupport

# Use percentage instead of absolute values
-XX:MaxRAMPercentage=75.0
-XX:InitialRAMPercentage=50.0

# Exit on OOM (let orchestrator restart)
-XX:+ExitOnOutOfMemoryError

# Generate heap dump for analysis
-XX:+HeapDumpOnOutOfMemoryError
-XX:HeapDumpPath=/dumps/heapdump.hprof
```

---

## Kubernetes Fundamentals

### Deployment YAML

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
  namespace: production
  labels:
    app: payment-service
    version: v1
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: payment-service
  template:
    metadata:
      labels:
        app: payment-service
        version: v1
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "8080"
        prometheus.io/path: "/actuator/prometheus"
    spec:
      serviceAccountName: payment-service

      # Pod anti-affinity - spread across nodes
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              podAffinityTerm:
                labelSelector:
                  matchExpressions:
                    - key: app
                      operator: In
                      values:
                        - payment-service
                topologyKey: kubernetes.io/hostname

      containers:
        - name: payment-service
          image: myregistry/payment-service:1.2.3
          imagePullPolicy: IfNotPresent

          ports:
            - name: http
              containerPort: 8080
              protocol: TCP

          env:
            - name: SPRING_PROFILES_ACTIVE
              value: "production"

            - name: JAVA_OPTS
              value: "-XX:+UseContainerSupport -XX:MaxRAMPercentage=75.0"

            - name: DATABASE_URL
              valueFrom:
                secretKeyRef:
                  name: payment-service-secrets
                  key: database-url

            - name: KAFKA_BOOTSTRAP_SERVERS
              valueFrom:
                configMapKeyRef:
                  name: payment-service-config
                  key: kafka-bootstrap-servers

          resources:
            requests:
              memory: "512Mi"
              cpu: "500m"
            limits:
              memory: "1Gi"
              cpu: "1000m"

          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 60
            periodSeconds: 10
            timeoutSeconds: 5
            failureThreshold: 3

          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 5
            timeoutSeconds: 3
            failureThreshold: 3

          startupProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 0
            periodSeconds: 10
            timeoutSeconds: 3
            failureThreshold: 30

          volumeMounts:
            - name: config
              mountPath: /app/config
              readOnly: true
            - name: tmp
              mountPath: /tmp

      volumes:
        - name: config
          configMap:
            name: payment-service-config
        - name: tmp
          emptyDir: {}

      # Security context
      securityContext:
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 1000
```

---

## Health Probes

### Spring Boot Actuator Configuration

```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health, info, metrics, prometheus

  endpoint:
    health:
      probes:
        enabled: true
      show-details: always
      group:
        liveness:
          include: livenessState, diskSpace
        readiness:
          include: readinessState, db, kafka, redis

  health:
    livenessState:
      enabled: true
    readinessState:
      enabled: true
```

### Custom Health Indicators

```java
@Component
public class DatabaseHealthIndicator implements HealthIndicator {

    private final DataSource dataSource;

    @Override
    public Health health() {
        try (Connection conn = dataSource.getConnection()) {
            return Health.up()
                .withDetail("database", "reachable")
                .build();
        } catch (Exception e) {
            return Health.down()
                .withDetail("database", "unreachable")
                .withException(e)
                .build();
        }
    }
}
```

### Kubernetes Probe Configuration

```yaml
# Startup Probe - for slow-starting apps
startupProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  initialDelaySeconds: 0
  periodSeconds: 10
  failureThreshold: 30  # 5 minutes max

# Liveness Probe - is app alive?
livenessProbe:
  httpGet:
    path: /actuator/health/liveness
    port: 8080
  periodSeconds: 10
  failureThreshold: 3

# Readiness Probe - can app serve traffic?
readinessProbe:
  httpGet:
    path: /actuator/health/readiness
    port: 8080
  periodSeconds: 5
  failureThreshold: 3
```

---

## Resource Management

### Resource Requests vs Limits

```yaml
resources:
  requests:
    # Guaranteed minimum
    memory: "512Mi"
    cpu: "500m"
  limits:
    # Maximum allowed
    memory: "1Gi"
    cpu: "1000m"
```

### QoS Classes

```yaml
# Guaranteed - requests == limits
resources:
  requests:
    memory: "1Gi"
    cpu: "1000m"
  limits:
    memory: "1Gi"
    cpu: "1000m"

# Burstable - requests < limits
resources:
  requests:
    memory: "512Mi"
    cpu: "500m"
  limits:
    memory: "2Gi"
    cpu: "2000m"

# BestEffort - no requests/limits (avoid in production)
```

### Horizontal Pod Autoscaler

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: payment-service-hpa
  namespace: production
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

    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80

  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 50
          periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
        - type: Percent
          value: 100
          periodSeconds: 15
```

---

## ConfigMaps & Secrets

### ConfigMap

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: payment-service-config
  namespace: production
data:
  application.yml: |
    spring:
      application:
        name: payment-service
      kafka:
        bootstrap-servers: kafka-cluster:9092

    app:
      feature-flags:
        new-payment-flow: true

  kafka-bootstrap-servers: "kafka-cluster:9092"
  log-level: "INFO"
```

### Secret

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: payment-service-secrets
  namespace: production
type: Opaque
stringData:
  database-url: "mongodb://user:pass@mongo:27017/payments"
  kafka-password: "secret-password"
  api-key: "super-secret-key"
```

### Using in Deployment

```yaml
env:
  # From ConfigMap
  - name: KAFKA_BOOTSTRAP_SERVERS
    valueFrom:
      configMapKeyRef:
        name: payment-service-config
        key: kafka-bootstrap-servers

  # From Secret
  - name: DATABASE_URL
    valueFrom:
      secretKeyRef:
        name: payment-service-secrets
        key: database-url

# Mount as files
volumeMounts:
  - name: config
    mountPath: /app/config
  - name: secrets
    mountPath: /app/secrets
    readOnly: true

volumes:
  - name: config
    configMap:
      name: payment-service-config
  - name: secrets
    secret:
      secretName: payment-service-secrets
```

---

## Deployment Strategies

### Rolling Update

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1        # Max pods over desired count
    maxUnavailable: 0  # Min pods available during update
```

### Blue-Green Deployment

```yaml
# Blue deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service-blue
spec:
  selector:
    matchLabels:
      app: payment-service
      version: blue

---
# Green deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service-green
spec:
  selector:
    matchLabels:
      app: payment-service
      version: green

---
# Service switches between blue/green
apiVersion: v1
kind: Service
metadata:
  name: payment-service
spec:
  selector:
    app: payment-service
    version: blue  # Change to 'green' to switch
```

### Canary Deployment

```yaml
# Stable version - 90% traffic
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service-stable
spec:
  replicas: 9

---
# Canary version - 10% traffic
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service-canary
spec:
  replicas: 1
```

---

## Production Checklist

### ✅ Docker Best Practices

- [ ] Use multi-stage builds
- [ ] Run as non-root user
- [ ] Use specific image tags (not `latest`)
- [ ] Minimize layers
- [ ] Use `.dockerignore`
- [ ] Scan images for vulnerabilities
- [ ] Use small base images (Alpine, Distroless)
- [ ] Include health checks
- [ ] Set resource limits
- [ ] Use proper signal handling (dumb-init)

### ✅ Kubernetes Best Practices

- [ ] Set resource requests and limits
- [ ] Configure liveness and readiness probes
- [ ] Use namespaces for isolation
- [ ] Apply pod anti-affinity rules
- [ ] Use ConfigMaps and Secrets (not env vars)
- [ ] Enable network policies
- [ ] Set pod security contexts
- [ ] Configure pod disruption budgets
- [ ] Use horizontal pod autoscaling
- [ ] Implement proper logging

---

*This guide provides production-ready patterns for containerizing and orchestrating Java applications. Adapt these patterns to your specific requirements and infrastructure.*
