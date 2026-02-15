# Secrets Management & Configuration

A comprehensive guide to managing secrets, configuration, and sensitive data in Java applications with Vault, AWS Secrets Manager, and production-safe patterns.

---

## Table of Contents

1. [Secrets Management Fundamentals](#secrets-management-fundamentals)
2. [HashiCorp Vault](#hashicorp-vault)
3. [AWS Secrets Manager](#aws-secrets-manager)
4. [Spring Cloud Config](#spring-cloud-config)
5. [Kubernetes Secrets](#kubernetes-secrets)
6. [Environment-Specific Configuration](#environment-specific-configuration)
7. [Secret Rotation](#secret-rotation)
8. [Encryption](#encryption)
9. [Audit and Compliance](#audit-and-compliance)
10. [Production Patterns](#production-patterns)

---

## Secrets Management Fundamentals

### What Are Secrets?

```
Secrets are sensitive configuration values:

Authentication:
- Database passwords
- API keys
- OAuth client secrets
- JWT signing keys

Integration:
- Third-party API tokens
- Webhook secrets
- Service account credentials

Encryption:
- Encryption keys
- TLS certificates
- Private keys

Infrastructure:
- Cloud provider credentials
- SSH keys
- Kubernetes service tokens
```

### Common Anti-Patterns

```
❌ DON'T:

1. Hardcoded in Source Code
   private static final String DB_PASSWORD = "supersecret123";

2. Committed to Git
   application.properties with passwords

3. Environment Variables Only
   export DB_PASSWORD=secret123

4. Unencrypted Configuration Files
   config.properties in plain text

5. Shared Across Environments
   Same password for dev/staging/prod

6. No Rotation
   Same secrets for years

7. Logged or Exposed
   LOG.info("Password: " + password);

✅ DO:

1. Use Secrets Management System
   - Vault, AWS Secrets Manager, etc.

2. Encrypt at Rest
   - Use encryption for storage

3. Rotate Regularly
   - Automated rotation

4. Principle of Least Privilege
   - Minimal access duration

5. Audit Access
   - Log who accessed what
```

---

## HashiCorp Vault

### Setup

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.cloud</groupId>
    <artifactId>spring-cloud-starter-vault-config</artifactId>
</dependency>
```

```yaml
# bootstrap.yml
spring:
  application:
    name: payment-service
  cloud:
    vault:
      uri: http://vault.example.com:8200
      authentication: TOKEN
      token: ${VAULT_TOKEN}
      kv:
        enabled: true
        backend: secret
        default-context: payment-service
        application-name: ${spring.application.name}
```

### Storing Secrets in Vault

```bash
# Enable KV secrets engine
vault secrets enable -path=secret kv-v2

# Write secrets
vault kv put secret/payment-service/prod \
  db.password=supersecret \
  stripe.api.key=sk_live_xyz123 \
  jwt.secret=secret-key-12345

# Read secrets
vault kv get secret/payment-service/prod

# List secrets
vault kv list secret/payment-service
```

### Spring Boot Integration

```java
@Configuration
@RefreshScope
public class VaultConfig {

    @Value("${db.password}")
    private String dbPassword;

    @Value("${stripe.api.key}")
    private String stripeApiKey;

    @Value("${jwt.secret}")
    private String jwtSecret;

    @Bean
    public DataSource dataSource() {
        HikariConfig config = new HikariConfig();
        config.setJdbcUrl("jdbc:postgresql://localhost:5432/payments");
        config.setUsername("payment_user");
        config.setPassword(dbPassword); // From Vault
        return new HikariDataSource(config);
    }
}

// Dynamic secrets refresh
@Component
public class SecretRefreshListener {

    @EventListener
    public void handleRefresh(RefreshScopeRefreshedEvent event) {
        log.info("Secrets refreshed from Vault");
    }
}
```

### Database Dynamic Secrets

```bash
# Enable database secrets engine
vault secrets enable database

# Configure PostgreSQL connection
vault write database/config/postgresql \
  plugin_name=postgresql-database-plugin \
  allowed_roles="payment-service" \
  connection_url="postgresql://{{username}}:{{password}}@localhost:5432/payments" \
  username="vault" \
  password="vault-password"

# Create role with TTL
vault write database/roles/payment-service \
  db_name=postgresql \
  creation_statements="CREATE ROLE \"{{name}}\" WITH LOGIN PASSWORD '{{password}}' VALID UNTIL '{{expiration}}'; \
    GRANT SELECT, INSERT, UPDATE ON ALL TABLES IN SCHEMA public TO \"{{name}}\";" \
  default_ttl="1h" \
  max_ttl="24h"

# Get dynamic credentials
vault read database/creds/payment-service
```

```java
@Service
public class DynamicDatabaseConfig {

    private final VaultTemplate vaultTemplate;

    @Scheduled(fixedRate = 3000000) // 50 minutes
    public void renewDatabaseCredentials() {
        VaultResponse response = vaultTemplate.read(
            "database/creds/payment-service"
        );

        String username = response.getData().get("username").toString();
        String password = response.getData().get("password").toString();

        // Update datasource
        updateDataSource(username, password);

        log.info("Database credentials renewed");
    }
}
```

---

## AWS Secrets Manager

### Setup

```xml
<dependency>
    <groupId>com.amazonaws</groupId>
    <artifactId>aws-java-sdk-secretsmanager</artifactId>
    <version>1.12.529</version>
</dependency>
```

### Storing and Retrieving Secrets

```java
@Configuration
public class SecretsManagerConfig {

    private final AWSSecretsManager secretsManager;

    public SecretsManagerConfig() {
        this.secretsManager = AWSSecretsManagerClientBuilder
            .standard()
            .withRegion(Regions.US_EAST_1)
            .build();
    }

    @Bean
    public String databasePassword() {
        return getSecret("payment-service/database");
    }

    @Bean
    public String stripeApiKey() {
        return getSecret("payment-service/stripe-api-key");
    }

    private String getSecret(String secretName) {
        GetSecretValueRequest request = new GetSecretValueRequest()
            .withSecretId(secretName)
            .withVersionStage("AWSCURRENT");

        try {
            GetSecretValueResult result = secretsManager.getSecretValue(request);

            if (result.getSecretString() != null) {
                return result.getSecretString();
            } else {
                return new String(
                    Base64.getDecoder().decode(result.getSecretBinary()).array()
                );
            }

        } catch (ResourceNotFoundException e) {
            log.error("Secret not found: {}", secretName);
            throw new SecretNotFoundException(secretName);
        } catch (InvalidRequestException | InvalidParameterException e) {
            log.error("Invalid request for secret: {}", secretName);
            throw new SecretRetrievalException(secretName, e);
        }
    }
}

// Caching secrets
@Service
public class CachedSecretsService {

    private final LoadingCache<String, String> secretsCache;

    public CachedSecretsService(AWSSecretsManager secretsManager) {
        this.secretsCache = CacheBuilder.newBuilder()
            .expireAfterWrite(Duration.ofMinutes(30))
            .build(new CacheLoader<>() {
                @Override
                public String load(String secretName) {
                    return retrieveSecret(secretsManager, secretName);
                }
            });
    }

    public String getSecret(String secretName) {
        try {
            return secretsCache.get(secretName);
        } catch (ExecutionException e) {
            throw new SecretRetrievalException(secretName, e);
        }
    }
}
```

### Automatic Rotation

```python
# Lambda function for secret rotation
import boto3
import json

def lambda_handler(event, context):
    service_client = boto3.client('secretsmanager')
    arn = event['SecretId']
    token = event['ClientRequestToken']
    step = event['Step']

    metadata = service_client.describe_secret(SecretId=arn)

    if step == "createSecret":
        create_secret(service_client, arn, token)

    elif step == "setSecret":
        set_secret(service_client, arn, token)

    elif step == "testSecret":
        test_secret(service_client, arn, token)

    elif step == "finishSecret":
        finish_secret(service_client, arn, token)

def create_secret(service_client, arn, token):
    # Generate new secret
    new_password = generate_secure_password()

    # Store pending version
    service_client.put_secret_value(
        SecretId=arn,
        ClientRequestToken=token,
        SecretString=json.dumps({"password": new_password}),
        VersionStages=['AWSPENDING']
    )

def set_secret(service_client, arn, token):
    # Update database password
    pending = get_secret_value(service_client, arn, "AWSPENDING", token)
    update_database_password(pending['password'])

def test_secret(service_client, arn, token):
    # Test new credentials
    pending = get_secret_value(service_client, arn, "AWSPENDING", token)
    test_database_connection(pending['password'])

def finish_secret(service_client, arn, token):
    # Promote AWSPENDING to AWSCURRENT
    service_client.update_secret_version_stage(
        SecretId=arn,
        VersionStage='AWSCURRENT',
        MoveToVersionId=token,
        RemoveFromVersionId=get_current_version_id(service_client, arn)
    )
```

---

## Spring Cloud Config

### Config Server Setup

```yaml
# config-server/application.yml
spring:
  cloud:
    config:
      server:
        git:
          uri: https://github.com/company/config-repo
          search-paths: '{application}'
          default-label: main
        encrypt:
          enabled: true

encrypt:
  key: ${ENCRYPT_KEY}
```

```java
@SpringBootApplication
@EnableConfigServer
public class ConfigServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(ConfigServerApplication.class, args);
    }
}
```

### Encrypting Secrets

```bash
# Encrypt value
curl http://config-server:8888/encrypt \
  -d "supersecret123"

# Response:
# AQAVPvL0dN...encrypted-value...

# Use in config file
# payment-service.yml
spring:
  datasource:
    password: '{cipher}AQAVPvL0dN...encrypted-value...'
```

### Client Configuration

```yaml
# bootstrap.yml
spring:
  application:
    name: payment-service
  cloud:
    config:
      uri: http://config-server:8888
      fail-fast: true
      retry:
        initial-interval: 1000
        max-attempts: 6
```

---

## Kubernetes Secrets

### Creating Secrets

```bash
# From literal values
kubectl create secret generic payment-db \
  --from-literal=username=payment_user \
  --from-literal=password=supersecret123

# From file
echo -n 'supersecret123' > ./password.txt
kubectl create secret generic payment-db \
  --from-file=password=./password.txt

# From YAML (base64 encoded)
apiVersion: v1
kind: Secret
metadata:
  name: payment-db
type: Opaque
data:
  username: cGF5bWVudF91c2Vy  # payment_user
  password: c3VwZXJzZWNyZXQxMjM=  # supersecret123
```

### Using Secrets in Pods

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
spec:
  template:
    spec:
      containers:
        - name: payment-service
          image: payment-service:latest

          # Environment variables from secret
          env:
            - name: DB_USERNAME
              valueFrom:
                secretKeyRef:
                  name: payment-db
                  key: username
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: payment-db
                  key: password

          # Mount secret as volume
          volumeMounts:
            - name: stripe-secret
              mountPath: /etc/secrets/stripe
              readOnly: true

      volumes:
        - name: stripe-secret
          secret:
            secretName: stripe-api-key
            items:
              - key: api-key
                path: stripe.key
```

### External Secrets Operator

```yaml
# Install External Secrets Operator
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: vault-backend
spec:
  provider:
    vault:
      server: "http://vault:8200"
      path: "secret"
      version: "v2"
      auth:
        kubernetes:
          mountPath: "kubernetes"
          role: "payment-service"

---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: payment-secrets
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: vault-backend
    kind: SecretStore

  target:
    name: payment-db-secret
    creationPolicy: Owner

  data:
    - secretKey: password
      remoteRef:
        key: payment-service/database
        property: password
```

---

## Environment-Specific Configuration

### Spring Profiles

```yaml
# application.yml (common)
spring:
  application:
    name: payment-service

# application-dev.yml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/payments_dev
    username: dev_user
    password: dev_password

logging:
  level:
    root: DEBUG

# application-prod.yml
spring:
  datasource:
    url: ${DB_URL}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}  # From Vault/Secrets Manager

logging:
  level:
    root: INFO

security:
  require-ssl: true
```

```java
@Configuration
@Profile("prod")
public class ProductionConfig {

    @Value("${db.password}")
    private String dbPassword; // From secrets manager

    @Bean
    public DataSource dataSource() {
        // Production datasource with secrets
    }
}

@Configuration
@Profile("dev")
public class DevelopmentConfig {

    @Bean
    public DataSource dataSource() {
        // Development datasource with local credentials
    }
}
```

---

## Secret Rotation

### Database Password Rotation

```java
@Service
public class DatabaseSecretRotationService {

    private final SecretsManagerClient secretsManager;
    private final DataSource dataSource;

    @Scheduled(cron = "0 0 2 * * ?") // 2 AM daily
    public void rotatePassword() {
        try {
            // 1. Generate new password
            String newPassword = generateSecurePassword();

            // 2. Update secret in Secrets Manager (pending)
            updateSecretPending(newPassword);

            // 3. Update database user password
            updateDatabasePassword(newPassword);

            // 4. Test new password
            testConnection(newPassword);

            // 5. Promote secret to current
            promoteSecretToCurrent();

            // 6. Update application datasource
            updateDataSource(newPassword);

            log.info("Database password rotated successfully");

        } catch (Exception e) {
            log.error("Password rotation failed", e);
            rollbackRotation();
            throw new RotationException("Failed to rotate password", e);
        }
    }

    private String generateSecurePassword() {
        SecureRandom random = new SecureRandom();
        StringBuilder password = new StringBuilder(32);

        String chars = "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789!@#$%^&*";

        for (int i = 0; i < 32; i++) {
            password.append(chars.charAt(random.nextInt(chars.length())));
        }

        return password.toString();
    }

    private void updateDatabasePassword(String newPassword) {
        try (Connection conn = dataSource.getConnection();
             Statement stmt = conn.createStatement()) {

            stmt.execute(String.format(
                "ALTER USER payment_user WITH PASSWORD '%s'",
                newPassword.replace("'", "''")
            ));
        }
    }
}
```

### API Key Rotation

```java
@Service
public class ApiKeyRotationService {

    @Scheduled(fixedRate = 7 * 24 * 60 * 60 * 1000) // Weekly
    public void rotateApiKeys() {
        List<ApiKey> activeKeys = apiKeyRepository.findByStatus(ACTIVE);

        for (ApiKey key : activeKeys) {
            if (key.isOlderThan(Duration.ofDays(30))) {
                // Create new key
                ApiKey newKey = generateNewApiKey(key.getClientId());
                apiKeyRepository.save(newKey);

                // Deprecate old key (grace period)
                key.setStatus(DEPRECATED);
                key.setExpiresAt(Instant.now().plus(Duration.ofDays(7)));
                apiKeyRepository.save(key);

                // Notify client
                notifyClientOfKeyRotation(key.getClientId(), newKey);

                log.info("API key rotated for client: {}", key.getClientId());
            }
        }
    }
}
```

---

## Encryption

### Encrypting Configuration Files

```java
@Configuration
public class JasyptConfig {

    @Bean
    public StringEncryptor stringEncryptor() {
        PooledPBEStringEncryptor encryptor = new PooledPBEStringEncryptor();
        SimpleStringPBEConfig config = new SimpleStringPBEConfig();

        config.setPassword(System.getenv("JASYPT_ENCRYPTOR_PASSWORD"));
        config.setAlgorithm("PBEWITHHMACSHA512ANDAES_256");
        config.setKeyObtentionIterations("1000");
        config.setPoolSize("1");
        config.setSaltGeneratorClassName("org.jasypt.salt.RandomSaltGenerator");
        config.setIvGeneratorClassName("org.jasypt.iv.RandomIvGenerator");
        config.setStringOutputType("base64");

        encryptor.setConfig(config);
        return encryptor;
    }
}
```

```yaml
# application.yml
spring:
  datasource:
    password: ENC(encrypted-value-here)

stripe:
  api-key: ENC(another-encrypted-value)
```

```bash
# Encrypt value
java -cp jasypt-1.9.3.jar org.jasypt.intf.cli.JasyptPBEStringEncryptionCLI \
  input="mysecretvalue" \
  password="${JASYPT_ENCRYPTOR_PASSWORD}" \
  algorithm=PBEWITHHMACSHA512ANDAES_256
```

---

## Production Patterns

### Least Privilege Access

```yaml
# Vault policy for payment-service
path "secret/data/payment-service/*" {
  capabilities = ["read"]
}

path "database/creds/payment-service" {
  capabilities = ["read"]
}

path "pki/issue/payment-service" {
  capabilities = ["create", "update"]
}
```

### Monitoring Secret Access

```java
@Aspect
@Component
public class SecretAccessAuditAspect {

    private static final Logger auditLog = LoggerFactory.getLogger("SECRET_AUDIT");

    @Around("@annotation(AuditSecretAccess)")
    public Object auditSecretAccess(ProceedingJoinPoint joinPoint) throws Throwable {
        String secretName = extractSecretName(joinPoint);
        String user = SecurityContextHolder.getContext().getAuthentication().getName();

        auditLog.info("Secret accessed - user: {}, secret: {}, method: {}",
            user,
            secretName,
            joinPoint.getSignature().toShortString()
        );

        try {
            return joinPoint.proceed();
        } catch (Exception e) {
            auditLog.error("Secret access failed - user: {}, secret: {}, error: {}",
                user, secretName, e.getMessage());
            throw e;
        }
    }
}
```

---

## Best Practices

### ✅ DO

- Use dedicated secrets management system
- Rotate secrets regularly
- Encrypt secrets at rest
- Audit secret access
- Use short-lived credentials when possible
- Implement least privilege
- Separate secrets per environment
- Use secret scanning in CI/CD
- Monitor for secret exposure
- Have rotation procedures documented

### ❌ DON'T

- Hardcode secrets in source code
- Commit secrets to version control
- Share secrets across environments
- Use default/weak passwords
- Log secrets
- Email/Slack secrets
- Store secrets in plaintext
- Use same secret indefinitely
- Grant broad access to secrets
- Expose secrets in error messages

---

*This guide provides production-ready patterns for secrets management. Proper secrets management is critical for security - treat secrets with the care they deserve.*
