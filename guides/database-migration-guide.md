# Database Migration & Schema Evolution

A comprehensive guide to managing database schema changes with Flyway and Liquibase, zero-downtime migrations, and production-safe patterns.

---

## Table of Contents

1. [Migration Fundamentals](#migration-fundamentals)
2. [Flyway vs Liquibase](#flyway-vs-liquibase)
3. [Flyway Deep Dive](#flyway-deep-dive)
4. [Liquibase Deep Dive](#liquibase-deep-dive)
5. [Zero-Downtime Migrations](#zero-downtime-migrations)
6. [Rollback Strategies](#rollback-strategies)
7. [Testing Migrations](#testing-migrations)
8. [Production Deployment](#production-deployment)
9. [Common Migration Patterns](#common-migration-patterns)
10. [Troubleshooting](#troubleshooting)

---

## Migration Fundamentals

### Why Database Migrations?

```
❌ Without Migrations:
- Manual SQL scripts
- Inconsistent environments
- No version history
- Hard to rollback
- Risky deployments

✅ With Migrations:
- Automated schema changes
- Version controlled
- Repeatable deployments
- Safe rollbacks
- Environment consistency
```

### Core Principles

```
1. Version Everything
   - Every change is a numbered migration
   - Never modify existing migrations
   - Always create new migrations

2. Forward Only
   - Migrations run in order
   - Once applied, never change
   - Rollback with new migration

3. Idempotent
   - Can run multiple times safely
   - Check before creating/dropping

4. Tested
   - Test in dev/staging first
   - Automated testing
   - Dry run in production
```

---

## Flyway vs Liquibase

### Comparison

| Feature | Flyway | Liquibase |
|---------|--------|-----------|
| **Format** | SQL files | SQL, XML, YAML, JSON |
| **Learning Curve** | Easy | Moderate |
| **Database Support** | Excellent | Excellent |
| **Rollback** | Paid (Teams) | Free |
| **Conditionals** | Limited | Extensive |
| **Context/Labels** | No | Yes |
| **Diff Generation** | Paid | Free |
| **Best For** | Simple, SQL-focused | Complex, multi-DB |

### When to Use What

```java
// Use Flyway if:
- Simple migrations
- SQL-first approach
- PostgreSQL/MySQL focused
- Team familiar with SQL

// Use Liquibase if:
- Multiple database types
- Need rollback (free version)
- Complex conditionals
- Need diff generation
```

---

## Flyway Deep Dive

### Setup

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-core</artifactId>
</dependency>
<dependency>
    <groupId>org.flywaydb</groupId>
    <artifactId>flyway-database-postgresql</artifactId>
</dependency>
```

```yaml
# application.yml
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: true
    validate-on-migrate: true
    out-of-order: false
    table: flyway_schema_history
```

### Migration Files

```sql
-- V1__create_payment_table.sql
-- Version 1: Create payment table

CREATE TABLE IF NOT EXISTS payments (
    id VARCHAR(36) PRIMARY KEY,
    user_id VARCHAR(36) NOT NULL,
    amount DECIMAL(19, 4) NOT NULL,
    currency VARCHAR(3) NOT NULL,
    status VARCHAR(20) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_payments_user_id ON payments(user_id);
CREATE INDEX idx_payments_status ON payments(status);
CREATE INDEX idx_payments_created_at ON payments(created_at);
```

```sql
-- V2__add_payment_method.sql
-- Version 2: Add payment method column

ALTER TABLE payments
ADD COLUMN payment_method VARCHAR(50);

-- Backfill existing data
UPDATE payments
SET payment_method = 'CARD'
WHERE payment_method IS NULL;

-- Make column NOT NULL
ALTER TABLE payments
ALTER COLUMN payment_method SET NOT NULL;
```

### Repeatable Migrations

```sql
-- R__create_payment_view.sql
-- Repeatable: Payment summary view

CREATE OR REPLACE VIEW payment_summary AS
SELECT
    user_id,
    COUNT(*) as total_payments,
    SUM(amount) as total_amount,
    MAX(created_at) as last_payment_date
FROM payments
WHERE status = 'COMPLETED'
GROUP BY user_id;
```

### Callbacks

```java
@Component
public class FlywayCallback implements Callback {

    @Override
    public boolean supports(Event event, Context context) {
        return event == Event.AFTER_MIGRATE;
    }

    @Override
    public boolean canHandleInTransaction(Event event, Context context) {
        return true;
    }

    @Override
    public void handle(Event event, Context context) {
        if (event == Event.AFTER_MIGRATE) {
            log.info("Migration completed successfully");
            // Clear caches, refresh materialized views, etc.
        }
    }
}
```

---

## Liquibase Deep Dive

### Setup

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.liquibase</groupId>
    <artifactId>liquibase-core</artifactId>
</dependency>
```

```yaml
# application.yml
spring:
  liquibase:
    enabled: true
    change-log: classpath:db/changelog/db.changelog-master.yaml
    default-schema: public
    liquibase-schema: public
    drop-first: false
```

### XML Changelog

```xml
<!-- db.changelog-master.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
    xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
        http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.20.xsd">

    <changeSet id="1" author="developer">
        <createTable tableName="payments">
            <column name="id" type="varchar(36)">
                <constraints primaryKey="true" nullable="false"/>
            </column>
            <column name="user_id" type="varchar(36)">
                <constraints nullable="false"/>
            </column>
            <column name="amount" type="decimal(19,4)">
                <constraints nullable="false"/>
            </column>
            <column name="currency" type="varchar(3)">
                <constraints nullable="false"/>
            </column>
            <column name="status" type="varchar(20)">
                <constraints nullable="false"/>
            </column>
            <column name="created_at" type="timestamp" defaultValueComputed="CURRENT_TIMESTAMP">
                <constraints nullable="false"/>
            </column>
        </createTable>

        <createIndex indexName="idx_payments_user_id" tableName="payments">
            <column name="user_id"/>
        </createIndex>

        <rollback>
            <dropTable tableName="payments"/>
        </rollback>
    </changeSet>

</databaseChangeLog>
```

### YAML Changelog

```yaml
# db.changelog-master.yaml
databaseChangeLog:
  - changeSet:
      id: 1
      author: developer
      changes:
        - createTable:
            tableName: payments
            columns:
              - column:
                  name: id
                  type: varchar(36)
                  constraints:
                    primaryKey: true
                    nullable: false
              - column:
                  name: user_id
                  type: varchar(36)
                  constraints:
                    nullable: false
              - column:
                  name: amount
                  type: decimal(19,4)
                  constraints:
                    nullable: false
              - column:
                  name: status
                  type: varchar(20)
                  constraints:
                    nullable: false

      rollback:
        - dropTable:
            tableName: payments
```

### Conditional Migrations

```yaml
- changeSet:
    id: 2
    author: developer
    context: production
    changes:
      - addColumn:
          tableName: payments
          columns:
            - column:
                name: encryption_key_id
                type: varchar(50)

# Run only in production
# liquibase --contexts=production update
```

---

## Zero-Downtime Migrations

### Expand-Contract Pattern

```sql
-- Phase 1: EXPAND - Add new column (nullable)
-- V10__add_email_column.sql
ALTER TABLE users
ADD COLUMN email VARCHAR(255);

-- Deploy new code that writes to BOTH old and new columns
-- Application code:
-- user.setEmailAddress(email); // Old column
-- user.setEmail(email);          // New column

-- Phase 2: Backfill data
-- V11__backfill_email.sql
UPDATE users
SET email = email_address
WHERE email IS NULL AND email_address IS NOT NULL;

-- Phase 3: Make new column NOT NULL
-- V12__make_email_not_null.sql
ALTER TABLE users
ALTER COLUMN email SET NOT NULL;

-- Deploy new code that only uses new column

-- Phase 4: CONTRACT - Remove old column
-- V13__remove_email_address.sql
ALTER TABLE users
DROP COLUMN email_address;
```

### Adding Column with Default

```sql
-- ❌ BAD: Locks table during backfill
ALTER TABLE payments
ADD COLUMN payment_method VARCHAR(50) NOT NULL DEFAULT 'CARD';

-- ✅ GOOD: Zero-downtime approach

-- Step 1: Add nullable column
ALTER TABLE payments
ADD COLUMN payment_method VARCHAR(50);

-- Step 2: Backfill in batches (application code)
UPDATE payments
SET payment_method = 'CARD'
WHERE payment_method IS NULL
  AND id IN (
      SELECT id FROM payments
      WHERE payment_method IS NULL
      LIMIT 1000
  );

-- Step 3: Make NOT NULL after backfill complete
ALTER TABLE payments
ALTER COLUMN payment_method SET NOT NULL;

-- Step 4: Add default constraint
ALTER TABLE payments
ALTER COLUMN payment_method SET DEFAULT 'CARD';
```

### Renaming Column

```sql
-- Phase 1: Add new column
ALTER TABLE users
ADD COLUMN full_name VARCHAR(255);

-- Phase 2: Dual write (application code)
-- Write to both columns

-- Phase 3: Backfill
UPDATE users
SET full_name = name
WHERE full_name IS NULL;

-- Phase 4: Switch reads to new column
-- Application only reads full_name

-- Phase 5: Drop old column
ALTER TABLE users
DROP COLUMN name;
```

---

## Rollback Strategies

### Forward-Only Rollback

```sql
-- V15__add_premium_flag.sql
ALTER TABLE users
ADD COLUMN is_premium BOOLEAN DEFAULT false;

-- V16__rollback_premium_flag.sql (if needed)
ALTER TABLE users
DROP COLUMN is_premium;
```

### Liquibase Rollback

```yaml
- changeSet:
    id: 5
    author: developer
    changes:
      - addColumn:
          tableName: users
          columns:
            - column:
                name: premium_tier
                type: varchar(20)

    rollback:
      - dropColumn:
          tableName: users
          columnName: premium_tier

# Rollback command:
# liquibase rollback --count=1
```

---

## Testing Migrations

### Integration Tests

```java
@SpringBootTest
@Testcontainers
class MigrationTest {

    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>(
        "postgres:15-alpine"
    );

    @DynamicPropertySource
    static void properties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }

    @Test
    void migrationsRunSuccessfully() {
        // Migrations run automatically during context startup
        // If they fail, this test fails

        assertThat(jdbcTemplate.queryForObject(
            "SELECT COUNT(*) FROM payments",
            Long.class
        )).isZero();
    }

    @Test
    void canInsertPaymentAfterMigration() {
        jdbcTemplate.update(
            "INSERT INTO payments (id, user_id, amount, currency, status) " +
            "VALUES (?, ?, ?, ?, ?)",
            UUID.randomUUID().toString(),
            "user123",
            new BigDecimal("100.00"),
            "USD",
            "PENDING"
        );

        Long count = jdbcTemplate.queryForObject(
            "SELECT COUNT(*) FROM payments",
            Long.class
        );

        assertThat(count).isEqualTo(1);
    }
}
```

---

## Production Deployment

### Pre-Deployment Checklist

```bash
# 1. Review all migrations
ls -la src/main/resources/db/migration/

# 2. Validate migrations locally
./mvnw flyway:validate

# 3. Test in staging
./mvnw flyway:migrate -Dflyway.url=jdbc:postgresql://staging-db:5432/payments

# 4. Create backup
pg_dump -h prod-db -U postgres payments > backup_$(date +%Y%m%d_%H%M%S).sql

# 5. Dry run (Flyway Pro)
./mvnw flyway:migrate -Dflyway.dryRunOutput=migration-dry-run.sql

# 6. Deploy during maintenance window
# Run migrations before deploying code

# 7. Monitor
tail -f /var/log/flyway/flyway.log
```

### Deployment Strategy

```
1. Backup Database
   ↓
2. Run Migrations
   ↓
3. Verify Migration Success
   ↓
4. Deploy Application Code
   ↓
5. Smoke Test
   ↓
6. Monitor
```

---

## Common Migration Patterns

### Adding Index (No Downtime)

```sql
-- Create index concurrently (PostgreSQL)
CREATE INDEX CONCURRENTLY idx_payments_user_status
ON payments(user_id, status);

-- For other databases, create during low-traffic period
```

### Data Migration

```sql
-- Batch update to avoid long locks
DO $$
DECLARE
    batch_size INTEGER := 1000;
    rows_updated INTEGER;
BEGIN
    LOOP
        UPDATE payments
        SET normalized_amount = amount * 100
        WHERE normalized_amount IS NULL
        LIMIT batch_size;

        GET DIAGNOSTICS rows_updated = ROW_COUNT;
        EXIT WHEN rows_updated = 0;

        -- Commit every batch
        COMMIT;

        -- Small delay between batches
        PERFORM pg_sleep(0.1);
    END LOOP;
END $$;
```

---

## Best Practices

### ✅ DO

- Version all schema changes
- Test migrations in staging first
- Backup before production migration
- Use zero-downtime patterns
- Keep migrations small and focused
- Document complex migrations
- Use meaningful migration names
- Review migration history regularly

### ❌ DON'T

- Modify existing migrations
- Skip version numbers
- Combine unrelated changes
- Deploy without testing
- Run migrations manually
- Use DROP without backup
- Ignore failed migrations
- Assume rollback is free

---

*This guide provides production-safe patterns for database schema evolution. Migrations are critical infrastructure - plan carefully and test thoroughly.*
