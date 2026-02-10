# 🐘 Gradle Build & Plugin Patterns Cheat Sheet

> **Purpose:** Production Gradle patterns for multi-module builds, custom plugins, dependency management, and CI optimization. Reference before configuring any new project or build pipeline.
> **Stack context:** Gradle 8.x / Kotlin DSL / Java 21+ / Spring Boot 3.x

---

## 📋 Project Structure Decision

| Question | Determines |
|----------|-----------|
| How many deployable artifacts? | Single vs multi-module |
| Shared libraries across services? | Platform/BOM module |
| Custom code generation needed? | Custom Gradle plugin |
| Build taking > 2 minutes? | Caching, parallelism, daemon tuning |

---

## 🏗️ Pattern 1: Multi-Module Project Structure

### Layout

```
payment-platform/
├── build.gradle.kts                    ← Root: shared config, no source
├── settings.gradle.kts                 ← Module registration
├── gradle.properties                   ← Shared properties
├── gradle/
│   ├── libs.versions.toml              ← Version catalog
│   └── wrapper/
│
├── platform-bom/                       ← BOM for dependency alignment
│   └── build.gradle.kts
│
├── shared/
│   ├── shared-domain/                  ← Domain models, value objects
│   │   └── build.gradle.kts
│   ├── shared-messaging/              ← Kafka/JMS abstractions
│   │   └── build.gradle.kts
│   ├── shared-resilience/             ← Circuit breaker, retry libraries
│   │   └── build.gradle.kts
│   └── shared-test/                   ← Test fixtures, Testcontainers base
│       └── build.gradle.kts
│
├── services/
│   ├── payment-service/               ← Deployable service
│   │   └── build.gradle.kts
│   ├── fraud-service/
│   │   └── build.gradle.kts
│   └── settlement-service/
│       └── build.gradle.kts
│
└── buildSrc/                          ← Convention plugins
    ├── build.gradle.kts
    └── src/main/kotlin/
        ├── java-library-conventions.gradle.kts
        ├── spring-service-conventions.gradle.kts
        └── test-conventions.gradle.kts
```

### settings.gradle.kts

```kotlin
rootProject.name = "payment-platform"

include(
    "platform-bom",
    "shared:shared-domain",
    "shared:shared-messaging",
    "shared:shared-resilience",
    "shared:shared-test",
    "services:payment-service",
    "services:fraud-service",
    "services:settlement-service"
)

// Version catalog
dependencyResolutionManagement {
    versionCatalogs {
        create("libs") {
            from(files("gradle/libs.versions.toml"))
        }
    }
}

// Build cache
buildCache {
    local {
        isEnabled = true
        directory = file("${rootDir}/.gradle/build-cache")
    }
}
```

### gradle/libs.versions.toml (Version Catalog)

```toml
[versions]
java = "21"
spring-boot = "3.3.0"
spring-dependency-mgmt = "1.1.5"
resilience4j = "2.2.0"
mongock = "5.4.0"
testcontainers = "1.19.8"
archunit = "1.3.0"
pitest = "1.16.0"
spotless = "6.25.0"

[libraries]
# Spring Boot starters
spring-boot-starter-web = { module = "org.springframework.boot:spring-boot-starter-web" }
spring-boot-starter-data-mongodb = { module = "org.springframework.boot:spring-boot-starter-data-mongodb" }
spring-boot-starter-actuator = { module = "org.springframework.boot:spring-boot-starter-actuator" }
spring-boot-starter-test = { module = "org.springframework.boot:spring-boot-starter-test" }
spring-kafka = { module = "org.springframework.kafka:spring-kafka" }

# Resilience
resilience4j-spring-boot3 = { module = "io.github.resilience4j:resilience4j-spring-boot3", version.ref = "resilience4j" }
resilience4j-micrometer = { module = "io.github.resilience4j:resilience4j-micrometer", version.ref = "resilience4j" }

# Testing
testcontainers-bom = { module = "org.testcontainers:testcontainers-bom", version.ref = "testcontainers" }
testcontainers-mongodb = { module = "org.testcontainers:mongodb" }
testcontainers-kafka = { module = "org.testcontainers:kafka" }
archunit = { module = "com.tngtech.archunit:archunit-junit5", version.ref = "archunit" }

[bundles]
resilience = ["resilience4j-spring-boot3", "resilience4j-micrometer"]
testcontainers = ["testcontainers-mongodb", "testcontainers-kafka"]

[plugins]
spring-boot = { id = "org.springframework.boot", version.ref = "spring-boot" }
spring-dependency-mgmt = { id = "io.spring.dependency-management", version.ref = "spring-dependency-mgmt" }
spotless = { id = "com.diffplug.spotless", version.ref = "spotless" }
pitest = { id = "info.solidsoft.pitest", version.ref = "pitest" }
```

---

## 🔧 Pattern 2: Convention Plugins (buildSrc)

### buildSrc/build.gradle.kts

```kotlin
plugins {
    `kotlin-dsl`
}

repositories {
    gradlePluginPortal()
    mavenCentral()
}

dependencies {
    // Make Spring Boot plugin available in convention plugins
    implementation("org.springframework.boot:spring-boot-gradle-plugin:3.3.0")
    implementation("io.spring.gradle:dependency-management-plugin:1.1.5")
}
```

### java-library-conventions.gradle.kts

```kotlin
// Applied to ALL Java modules (shared libraries + services)
plugins {
    java
}

java {
    toolchain {
        languageVersion.set(JavaLanguageVersion.of(21))
    }
}

tasks.withType<JavaCompile>().configureEach {
    options.compilerArgs.addAll(listOf(
        "-parameters",                    // Preserve parameter names
        "--enable-preview",               // Enable preview features
        "-Xlint:all,-processing,-serial"  // Comprehensive warnings
    ))
}

tasks.withType<Test>().configureEach {
    useJUnitPlatform()
    jvmArgs("--enable-preview")
    maxParallelForks = Runtime.getRuntime().availableProcessors().div(2).coerceAtLeast(1)
    
    testLogging {
        events("PASSED", "FAILED", "SKIPPED")
        showExceptions = true
        showStandardStreams = false
    }
}

// Consistent dependency versions
dependencies {
    // Test dependencies for ALL modules
    "testImplementation"("org.assertj:assertj-core:3.25.3")
    "testImplementation"("org.mockito:mockito-core:5.11.0")
}
```

### spring-service-conventions.gradle.kts

```kotlin
// Applied to deployable Spring Boot services
plugins {
    id("java-library-conventions")
    id("org.springframework.boot")
    id("io.spring.dependency-management")
}

dependencyManagement {
    imports {
        mavenBom("org.testcontainers:testcontainers-bom:1.19.8")
    }
}

dependencies {
    implementation("org.springframework.boot:spring-boot-starter-actuator")
    implementation("io.micrometer:micrometer-registry-prometheus")
    
    testImplementation("org.springframework.boot:spring-boot-starter-test")
}

// Spring Boot jar configuration
tasks.named<org.springframework.boot.gradle.tasks.bundling.BootJar>("bootJar") {
    archiveFileName.set("${project.name}.jar")
    launchScript()   // Makes jar directly executable (./app.jar)
}
```

### Using Convention Plugins in Modules

```kotlin
// shared/shared-domain/build.gradle.kts
plugins {
    id("java-library-conventions")   // Just Java, no Spring Boot
}

// Clean — no dependencies on frameworks
dependencies {
    // Domain module: ZERO framework dependencies
}

// services/payment-service/build.gradle.kts
plugins {
    id("spring-service-conventions")   // Full Spring Boot service
}

dependencies {
    implementation(project(":shared:shared-domain"))
    implementation(project(":shared:shared-messaging"))
    implementation(project(":shared:shared-resilience"))
    
    implementation(libs.spring.boot.starter.web)
    implementation(libs.spring.boot.starter.data.mongodb)
    implementation(libs.spring.kafka)
    implementation(libs.bundles.resilience)
    
    testImplementation(project(":shared:shared-test"))
    testImplementation(libs.bundles.testcontainers)
    testImplementation(libs.archunit)
}
```

---

## 📦 Pattern 3: Dependency Management

### BOM (Bill of Materials)

```kotlin
// platform-bom/build.gradle.kts
plugins {
    `java-platform`
}

javaPlatform {
    allowDependenciesInConstraints()
}

dependencies {
    constraints {
        // Pin versions for ALL modules in the platform
        api("com.fasterxml.jackson:jackson-bom:2.17.0")
        api("io.github.resilience4j:resilience4j-bom:2.2.0")
        api("org.testcontainers:testcontainers-bom:1.19.8")
        
        // Internal modules
        api(project(":shared:shared-domain"))
        api(project(":shared:shared-messaging"))
    }
}

// Usage in services:
// dependencies {
//     implementation(platform(project(":platform-bom")))
// }
```

### Dependency Locking (Reproducible Builds)

```kotlin
// Root build.gradle.kts
allprojects {
    dependencyLocking {
        lockAllConfigurations()
    }
}

// Generate lock files: ./gradlew dependencies --write-locks
// Lock files: gradle/dependency-locks/*.lockfile
// Commit lock files to version control
```

### Vulnerability Scanning

```kotlin
// Root build.gradle.kts
plugins {
    id("org.owasp.dependencycheck") version "9.0.10"
}

dependencyCheck {
    failBuildOnCVSS = 7.0f   // Fail on HIGH+ vulnerabilities
    suppressionFile = "config/owasp-suppressions.xml"
    analyzers.apply {
        assemblyEnabled = false
        nodeEnabled = false
    }
}
```

---

## ⚡ Pattern 4: Build Performance

### gradle.properties

```properties
# ── Daemon ──
org.gradle.daemon=true
org.gradle.daemon.idletimeout=10800000   # 3 hours

# ── Memory ──
org.gradle.jvmargs=-Xmx2g -XX:+UseG1GC -XX:+HeapDumpOnOutOfMemoryError

# ── Parallelism ──
org.gradle.parallel=true
org.gradle.workers.max=4

# ── Caching ──
org.gradle.caching=true
org.gradle.configuration-cache=true      # Gradle 8.1+

# ── Kotlin DSL ──
kotlin.code.style=official

# ── Project properties ──
springBootVersion=3.3.0
javaVersion=21
```

### Task Optimization

```kotlin
// ── Avoid unnecessary test re-runs ──
tasks.withType<Test>().configureEach {
    // Only re-run if source or test files changed
    inputs.files(sourceSets["main"].output, sourceSets["test"].output)
    
    // Tag-based filtering
    systemProperty("junit.jupiter.execution.parallel.enabled", "true")
}

// ── Separate unit and integration tests ──
testing {
    suites {
        val test by getting(JvmTestSuite::class) {
            // Unit tests
        }
        register<JvmTestSuite>("integrationTest") {
            dependencies {
                implementation(project())
                implementation(libs.testcontainers.mongodb)
            }
            targets.all {
                testTask.configure {
                    shouldRunAfter(test)
                }
            }
        }
    }
}

tasks.named("check") {
    dependsOn(testing.suites.named("integrationTest"))
}
```

---

## 🔌 Pattern 5: Custom Gradle Plugin (Code Generation)

```kotlin
// buildSrc/src/main/kotlin/codegen-conventions.gradle.kts
import org.gradle.api.DefaultTask
import org.gradle.api.tasks.TaskAction
import org.gradle.api.tasks.Input
import org.gradle.api.tasks.OutputDirectory

abstract class GenerateEventClasses : DefaultTask() {

    @get:Input
    abstract val eventDefinitions: Property<String>

    @get:OutputDirectory
    abstract val outputDir: DirectoryProperty

    @TaskAction
    fun generate() {
        val outDir = outputDir.get().asFile
        outDir.mkdirs()
        // Parse event definitions, generate Java source files
        logger.lifecycle("Generating event classes to ${outDir.path}")
    }
}

// Register task in module build
tasks.register<GenerateEventClasses>("generateEvents") {
    eventDefinitions.set(file("src/main/resources/events.yaml").readText())
    outputDir.set(layout.buildDirectory.dir("generated/events"))
}

// Wire generated sources into compilation
sourceSets["main"].java.srcDir(tasks.named<GenerateEventClasses>("generateEvents").flatMap { it.outputDir })
```

---

## 🐳 Pattern 6: Docker Integration

```kotlin
// services/payment-service/build.gradle.kts
tasks.named<org.springframework.boot.gradle.tasks.bundling.BootBuildImage>("bootBuildImage") {
    imageName.set("payment-platform/${project.name}:${project.version}")
    environment.set(mapOf(
        "BP_JVM_VERSION" to "21",
        "BPE_JAVA_TOOL_OPTIONS" to "-XX:+UseZGC -XX:+ZGenerational"
    ))
    buildpacks.set(listOf(
        "urn:cnb:builder:paketo-buildpacks/java"
    ))
}

// Or custom Dockerfile task
tasks.register<Exec>("dockerBuild") {
    dependsOn("bootJar")
    commandLine("docker", "build",
        "-t", "payment-platform/${project.name}:${project.version}",
        "-f", "Dockerfile",
        ".")
}
```

---

## 🚫 Gradle Anti-Patterns

| Anti-Pattern | Why It's Dangerous | Fix |
|---|---|---|
| **Duplicated config across modules** | Inconsistent versions, hard to maintain | Convention plugins in buildSrc |
| **No version catalog** | Versions scattered, conflicts | gradle/libs.versions.toml |
| **allprojects{} with plugins** | Applies plugins to modules that don't need them | Convention plugins, apply selectively |
| **Dynamic versions** (+, latest) | Non-reproducible builds | Pin versions, dependency locking |
| **No build cache** | Slow CI, redundant work | `org.gradle.caching=true` |
| **Flat project (no modules)** | Monolithic builds, no reuse | Multi-module with shared libraries |
| **buildSrc as a dumping ground** | Slow incremental builds when buildSrc changes | Keep buildSrc focused on conventions |

---

## 💡 Golden Rules of Gradle Builds

```
1.  VERSION CATALOG for all dependencies — single source of truth.
2.  CONVENTION PLUGINS for shared config — DRY across modules.
3.  BUILD CACHE + CONFIGURATION CACHE — dramatic CI speed improvement.
4.  SEPARATE unit and integration tests — fast feedback first.
5.  BOM for version alignment — no dependency conflicts across modules.
6.  LOCK dependencies — reproducible builds, auditable changes.
7.  SCAN for vulnerabilities — OWASP dependency check in CI.
8.  PARALLEL builds — org.gradle.parallel=true, always.
9.  MINIMAL module dependencies — domain module has ZERO framework deps.
10. bootJar only on deployables — shared libraries use java-library plugin.
```

---

*Last updated: February 2026 | Stack: Gradle 8.x / Kotlin DSL / Java 21+ / Spring Boot 3.x*
