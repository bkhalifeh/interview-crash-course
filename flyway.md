# Flyway Database Migration Tool: Ultimate Interview Crash Course

## 1. Core Concepts You Must Know Cold

### What is Flyway?
Flyway is an open-source database migration tool that emphasizes **convention over configuration**. It manages version control for your database schema, enabling teams to evolve database structures reliably and consistently across all environments.

**Why it matters in interviews:** Companies ask about Flyway because database migrations are critical in CI/CD pipelines, microservices architectures, and any system that requires schema evolution without downtime.

---

### Migration Types

#### **Versioned Migrations (V)**
The most common type. Applied exactly once, in order, and tracked in the schema history table.

```sql
-- V1__Create_users_table.sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    email VARCHAR(255) NOT NULL UNIQUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- V2__Add_users_name_column.sql
ALTER TABLE users ADD COLUMN name VARCHAR(100);
```

**Naming Convention:** `V{version}__{description}.sql`
- `V` prefix (uppercase)
- Version number (e.g., `1`, `1.1`, `20231215`)
- Double underscore `__` (CRITICAL - single underscore fails)
- Description with underscores
- `.sql` extension

#### **Undo Migrations (U)** (Flyway Teams/Enterprise only)
Reverses versioned migrations. Same naming but with `U` prefix.

```sql
-- U2__Remove_users_name_column.sql
ALTER TABLE users DROP COLUMN name;
```

#### **Repeatable Migrations (R)**
Applied every time their checksum changes. Used for views, stored procedures, seed data.

```sql
-- R__Create_user_statistics_view.sql
CREATE OR REPLACE VIEW user_statistics AS
SELECT COUNT(*) as total_users FROM users;
```

**Key insight:** Repeatable migrations run AFTER all versioned migrations, ordered alphabetically by description.

---

### Schema History Table (`flyway_schema_history`)

Flyway tracks all applied migrations in this table:

| Column | Purpose |
|--------|---------|
| `installed_rank` | Order of execution |
| `version` | Migration version (NULL for repeatable) |
| `description` | Extracted from filename |
| `type` | SQL, JDBC, SPRING_JDBC |
| `script` | Filename |
| `checksum` | CRC32 of migration content |
| `installed_by` | Database user |
| `installed_on` | Timestamp |
| `execution_time` | Milliseconds |
| `success` | Boolean |

**Interview question trap:** "What happens if you modify an already-applied migration?" 
**Answer:** Flyway detects checksum mismatch and FAILS validation. You must either repair or create a new migration.

---

### Core Commands

```bash
# Apply pending migrations
flyway migrate

# Show migration status and info
flyway info

# Validate applied migrations match local files
flyway validate

# Fix schema history (reset checksums, remove failed entries)
flyway repair

# Remove all objects from schema (DANGEROUS)
flyway clean

# Create schema history table without running migrations
flyway baseline
```

**`baseline` deep-dive:** Used when Flyway is introduced to an existing database. Sets a baseline version so Flyway ignores all migrations up to that version.

```bash
flyway -baselineVersion=5 -baselineDescription="Initial baseline" baseline
```

---

### Configuration Methods

#### 1. Configuration File (`flyway.conf`)
```properties
flyway.url=jdbc:postgresql://localhost:5432/mydb
flyway.user=admin
flyway.password=secret
flyway.locations=classpath:db/migration,filesystem:/opt/migrations
flyway.schemas=public,audit
flyway.baselineOnMigrate=true
flyway.outOfOrder=false
flyway.validateOnMigrate=true
```

#### 2. Environment Variables
```bash
export FLYWAY_URL=jdbc:postgresql://localhost:5432/mydb
export FLYWAY_USER=admin
export FLYWAY_PASSWORD=secret
```

#### 3. Command Line Arguments
```bash
flyway -url=jdbc:postgresql://localhost:5432/mydb -user=admin migrate
```

#### 4. Programmatic (Java)
```java
Flyway flyway = Flyway.configure()
    .dataSource(url, user, password)
    .locations("classpath:db/migration")
    .load();
flyway.migrate();
```

**Priority order (highest to lowest):** Command line > Environment variables > Config files > Programmatic defaults

---

### Placeholders & Callbacks

#### Placeholders
```sql
-- V3__Create_environment_table.sql
CREATE TABLE ${table_prefix}config (
    key VARCHAR(100),
    environment VARCHAR(50) DEFAULT '${env}'
);
```

```properties
flyway.placeholders.table_prefix=app_
flyway.placeholders.env=production
```

#### Callbacks
Execute custom code at specific points:

| Callback | When |
|----------|------|
| `beforeMigrate` | Before migrate starts |
| `beforeEachMigrate` | Before each migration |
| `afterEachMigrate` | After each migration |
| `afterMigrate` | After all migrations |
| `beforeValidate` | Before validation |
| `afterValidate` | After validation |
| `beforeClean` | Before clean |
| `afterClean` | After clean |

```sql
-- beforeMigrate__audit_log.sql (in callbacks folder)
INSERT INTO migration_audit (action, timestamp) 
VALUES ('MIGRATION_STARTED', CURRENT_TIMESTAMP);
```

---

### Java-based Migrations

When SQL isn't enough:

```java
package db.migration;

import org.flywaydb.core.api.migration.BaseJavaMigration;
import org.flywaydb.core.api.migration.Context;

public class V4__Complex_data_migration extends BaseJavaMigration {
    @Override
    public void migrate(Context context) throws Exception {
        try (var stmt = context.getConnection().createStatement()) {
            // Complex logic, external API calls, data transformations
            var rs = stmt.executeQuery("SELECT id, data FROM legacy_table");
            while (rs.next()) {
                // Transform and insert
            }
        }
    }
}
```

**Use cases:** Data encryption, external API integration, complex data transformations, conditional logic.

---

### Transaction Handling

**Critical interview concept:**

- **Most databases:** Each migration runs in its own transaction. Failure = automatic rollback.
- **DDL in MySQL/MariaDB:** DDL statements (CREATE, ALTER) cause implicit commits. Cannot be rolled back!
- **PostgreSQL:** DDL IS transactional. Full rollback on failure.

```sql
-- V5__Safe_migration.sql
-- PostgreSQL: This entire block is atomic
BEGIN;
CREATE TABLE orders (id SERIAL PRIMARY KEY);
CREATE INDEX idx_orders_date ON orders(created_at);
-- If index creation fails, table creation is also rolled back
COMMIT;
```

---

## 2. Most Frequently Asked Interview Questions & Topics

### Easy (Warm-up / Phone Screen)

| Question | Key Points |
|----------|------------|
| What is Flyway and why use it? | Version control for DB, team collaboration, CI/CD integration, rollback tracking |
| Explain the naming convention | V{version}__{description}.sql, double underscore critical |
| What's the schema history table? | `flyway_schema_history`, tracks all migrations, checksums, timestamps |
| Difference between `migrate` and `baseline`? | migrate applies pending; baseline marks existing DB at specific version |
| How do you run Flyway? | CLI, Maven/Gradle, programmatic Java API, Spring Boot auto-config |

### Medium (Technical Interview)

| Question | Key Points |
|----------|------------|
| Versioned vs Repeatable migrations? | V = once, ordered; R = every checksum change, after V, alphabetical |
| What happens with checksum mismatch? | Validation fails, must repair or maintain consistency |
| How to handle existing database? | Use `baseline`, set baselineVersion to current schema state |
| Out-of-order migrations - pros/cons? | Allows parallel development; can cause dependency issues |
| How does Flyway handle failures? | Transaction rollback (DB-dependent), marks as failed, blocks subsequent |
| Placeholders use case? | Environment-specific values, table prefixes, dynamic config |

### Hard (Senior / System Design)

| Question | Key Points |
|----------|------------|
| Zero-downtime migrations strategy? | Expand-contract pattern, backward-compatible changes, multi-phase |
| Flyway in microservices? | Per-service schema, migration ownership, coordination challenges |
| Blue-green deployments with Flyway? | Forward-compatible migrations, feature flags, shared DB challenges |
| Rollback strategies without Undo? | Forward-fix migrations, backup/restore, manual scripts |
| Handling large table migrations? | Batching, pt-online-schema-change, avoid locks |
| Multi-tenancy approaches? | Schema-per-tenant, placeholders, programmatic location |

### System Design Integration Questions

1. **Design a CI/CD pipeline with database migrations**
   - Flyway in Docker container
   - Migration validation in PR checks
   - Staging environment testing
   - Production deployment with health checks

2. **Design schema migration for distributed system**
   - Eventual consistency considerations
   - Service-owned schemas
   - Cross-service data dependencies

3. **Design rollback strategy for critical production failure**
   - Immediate: forward-fix migration
   - Short-term: backup restoration
   - Long-term: blue-green with canary

---

## 3. One-Page Cheat Sheet

### Naming Conventions
```
V{version}__{description}.sql     → Versioned (applied once)
U{version}__{description}.sql     → Undo (Teams only)
R__{description}.sql              → Repeatable (on checksum change)

Version formats: 1, 001, 1.2, 1.2.3, 20231215120000
```

### Essential Commands
| Command | Purpose | Safe for Prod? |
|---------|---------|----------------|
| `migrate` | Apply pending migrations | ✅ |
| `info` | Show migration status | ✅ |
| `validate` | Check consistency | ✅ |
| `repair` | Fix history table | ⚠️ Use carefully |
| `baseline` | Mark existing DB | ✅ (once) |
| `clean` | Drop all objects | ❌ NEVER |

### Key Configuration
```properties
flyway.url=jdbc:...
flyway.user=
flyway.password=
flyway.locations=classpath:db/migration
flyway.schemas=public
flyway.baselineOnMigrate=true    # Auto-baseline empty DB
flyway.outOfOrder=false          # Strict ordering
flyway.validateOnMigrate=true    # Validate before migrate
flyway.cleanDisabled=true        # ALWAYS in production
```

### Time/Space Complexity (Conceptual)
| Operation | Complexity | Notes |
|-----------|------------|-------|
| Finding pending migrations | O(m) | m = local migration files |
| Schema history lookup | O(n) | n = applied migrations (indexed) |
| Validation | O(n + m) | Compare both sets |
| Migration execution | O(migration) | Depends on SQL complexity |

### Rules of Thumb
1. **Never modify applied migrations** - Create new ones instead
2. **Always validate in CI** - Catch issues before production
3. **Use transactions** - Except DDL on MySQL
4. **Baseline existing DBs** - Don't recreate history
5. **Disable clean in prod** - `cleanDisabled=true`
6. **Test migrations** - Staging must mirror production schema
7. **Version numbers** - Use timestamps for teams: `V20231215120000__`

### Spring Boot Integration
```yaml
# application.yml - Auto-configured!
spring:
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: true
```

---

## 4. Best Practices & Common Mistakes

### ✅ What Senior Engineers Do

| Practice | Why |
|----------|-----|
| **Timestamp-based versions** (`V20231215120000__`) | Prevents merge conflicts |
| **One change per migration** | Easier rollback, clear history |
| **Idempotent repeatable migrations** | `CREATE OR REPLACE`, `DROP IF EXISTS` |
| **Backward-compatible changes first** | Zero-downtime deployments |
| **Separate DDL and DML** | Different transaction behavior |
| **Test on production-like data** | Catch performance issues |
| **Include migration in code review** | Schema changes are code |
| **Use `cleanDisabled=true`** | Prevent accidental data loss |
| **Validate in CI pipeline** | `flyway validate` on every build |
| **Document breaking changes** | Migration description + comments |

### ❌ What Gets You Rejected

| Mistake | Why It's Bad | Fix |
|---------|--------------|-----|
| Modifying applied migrations | Checksum mismatch, breaks all envs | Create new migration |
| Using `clean` in production | Deletes everything | Set `cleanDisabled=true` |
| Manual schema changes | Out-of-sync history | Always use migrations |
| Single underscore in filename | Migration not detected | Use double underscore `__` |
| Hardcoded environment values | Breaks across environments | Use placeholders |
| Gigantic migrations | Lock tables, timeout | Batch changes |
| Ignoring failed migrations | Blocks all subsequent | Repair or fix |
| No rollback strategy | Stuck on failure | Plan forward-fix migrations |
| Skipping staging tests | Production surprises | Mirror prod in staging |

### Expand-Contract Pattern (Senior-Level)

For zero-downtime column rename:

```sql
-- Phase 1: EXPAND (V10)
ALTER TABLE users ADD COLUMN full_name VARCHAR(200);
UPDATE users SET full_name = name;

-- Phase 2: Application uses BOTH columns (deploy code)

-- Phase 3: CONTRACT (V11 - after all instances updated)
ALTER TABLE users DROP COLUMN name;
```

---

## 5. Minimal but Complete Code Examples

### Example 1: Complete Spring Boot Setup

```java
// build.gradle
dependencies {
    implementation 'org.flywaydb:flyway-core:9.22.0'
    implementation 'org.springframework.boot:spring-boot-starter-jdbc'
    runtimeOnly 'org.postgresql:postgresql:42.6.0'
}
```

```yaml
# src/main/resources/application.yml
spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/myapp
    username: ${DB_USER:postgres}
    password: ${DB_PASSWORD:secret}
  flyway:
    enabled: true
    locations: classpath:db/migration
    baseline-on-migrate: true
    clean-disabled: true
    out-of-order: false
    validate-on-migrate: true
```

```sql
-- src/main/resources/db/migration/V1__Initial_schema.sql
CREATE TABLE users (
    id BIGSERIAL PRIMARY KEY,
    email VARCHAR(255) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_users_email ON users(email);

-- src/main/resources/db/migration/V2__Add_user_profile.sql
CREATE TABLE user_profiles (
    user_id BIGINT PRIMARY KEY REFERENCES users(id) ON DELETE CASCADE,
    first_name VARCHAR(100),
    last_name VARCHAR(100),
    avatar_url TEXT
);

-- src/main/resources/db/migration/R__User_stats_view.sql
CREATE OR REPLACE VIEW user_statistics AS
SELECT 
    COUNT(*) as total_users,
    COUNT(*) FILTER (WHERE created_at > CURRENT_DATE - INTERVAL '30 days') as new_users_30d
FROM users;
```

---

### Example 2: Programmatic Flyway with Custom Configuration

```java
import org.flywaydb.core.Flyway;
import org.flywaydb.core.api.MigrationInfo;
import org.flywaydb.core.api.output.MigrateResult;

public class FlywayManager {
    
    private final Flyway flyway;
    
    public FlywayManager(String url, String user, String password) {
        this.flyway = Flyway.configure()
            .dataSource(url, user, password)
            .locations("classpath:db/migration", "classpath:db/seed")
            .schemas("public", "audit")
            .baselineOnMigrate(true)
            .baselineVersion("0")
            .cleanDisabled(true)
            .outOfOrder(false)
            .validateOnMigrate(true)
            .placeholders(Map.of(
                "schema", "public",
                "environment", System.getenv("ENV")
            ))
            .load();
    }
    
    public void migrate() {
        // Show current state
        printInfo();
        
        // Validate before migrating
        flyway.validate();
        
        // Apply migrations
        MigrateResult result = flyway.migrate();
        
        System.out.printf("Applied %d migrations in %dms%n", 
            result.migrationsExecuted, 
            result.getTotalMigrationTime());
        
        if (!result.success) {
            throw new RuntimeException("Migration failed: " + result.warnings);
        }
    }
    
    public void printInfo() {
        System.out.println("\n=== Migration Status ===");
        for (MigrationInfo info : flyway.info().all()) {
            System.out.printf("%-10s | %-40s | %-10s | %s%n",
                info.getVersion(),
                info.getDescription(),
                info.getState(),
                info.getInstalledOn()
            );
        }
    }
    
    public void repair() {
        // Use after manually fixing failed migration
        flyway.repair();
    }
    
    // FOR DEVELOPMENT ONLY
    public void resetDatabase() {
        if (!"development".equals(System.getenv("ENV"))) {
            throw new IllegalStateException("Clean only allowed in development!");
        }
        flyway.clean();
        flyway.migrate();
    }
}

// Usage
public static void main(String[] args) {
    FlywayManager manager = new FlywayManager(
        "jdbc:postgresql://localhost:5432/myapp",
        "postgres",
        "password"
    );
    manager.migrate();
}
```

---

### Example 3: Java-Based Migration with Complex Logic

```java
package db.migration;

import org.flywaydb.core.api.migration.BaseJavaMigration;
import org.flywaydb.core.api.migration.Context;
import java.sql.*;
import java.security.MessageDigest;
import java.util.Base64;

/**
 * V3__Encrypt_sensitive_data.java
 * 
 * Migrates plaintext sensitive data to encrypted format.
 * Cannot be done in pure SQL due to encryption logic.
 */
public class V3__Encrypt_sensitive_data extends BaseJavaMigration {
    
    private static final int BATCH_SIZE = 1000;
    
    @Override
    public void migrate(Context context) throws Exception {
        Connection conn = context.getConnection();
        
        // Add encrypted column
        try (Statement stmt = conn.createStatement()) {
            stmt.execute(
                "ALTER TABLE users ADD COLUMN IF NOT EXISTS " +
                "ssn_encrypted VARCHAR(255)"
            );
        }
        
        // Migrate data in batches
        String selectSql = "SELECT id, ssn FROM users WHERE ssn IS NOT NULL " +
                          "AND ssn_encrypted IS NULL LIMIT ?";
        String updateSql = "UPDATE users SET ssn_encrypted = ?, ssn = NULL WHERE id = ?";
        
        int totalProcessed = 0;
        
        try (PreparedStatement select = conn.prepareStatement(selectSql);
             PreparedStatement update = conn.prepareStatement(updateSql)) {
            
            select.setInt(1, BATCH_SIZE);
            
            while (true) {
                ResultSet rs = select.executeQuery();
                int batchCount = 0;
                
                while (rs.next()) {
                    long id = rs.getLong("id");
                    String ssn = rs.getString("ssn");
                    
                    // Encrypt (simplified - use proper encryption in production)
                    String encrypted = encrypt(ssn);
                    
                    update.setString(1, encrypted);
                    update.setLong(2, id);
                    update.addBatch();
                    batchCount++;
                }
                
                if (batchCount == 0) break;
                
                update.executeBatch();
                totalProcessed += batchCount;
                
                System.out.printf("Processed %d records...%n", totalProcessed);
            }
        }
        
        System.out.printf("Migration complete. Total encrypted: %d%n", totalProcessed);
    }
    
    private String encrypt(String plaintext) throws Exception {
        // DEMO ONLY - Use proper AES encryption in production
        MessageDigest md = MessageDigest.getInstance("SHA-256");
        byte[] hash = md.digest(plaintext.getBytes());
        return Base64.getEncoder().encodeToString(hash);
    }
    
    @Override
    public Integer getChecksum() {
        // Return consistent checksum for validation
        return 3; // Increment if logic changes
    }
}
```

---

### Example 4: Docker & CI/CD Integration

```dockerfile
# Dockerfile.migration
FROM flyway/flyway:9.22-alpine

# Copy migration files
COPY sql/migrations /flyway/sql

# Copy configuration
COPY flyway.conf /flyway/conf/flyway.conf

ENTRYPOINT ["flyway"]
CMD ["migrate"]
```

```yaml
# docker-compose.yml
version: '3.8'
services:
  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: secret
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

  flyway:
    build:
      context: .
      dockerfile: Dockerfile.migration
    depends_on:
      db:
        condition: service_healthy
    environment:
      FLYWAY_URL: jdbc:postgresql://db:5432/myapp
      FLYWAY_USER: postgres
      FLYWAY_PASSWORD: secret
    command: migrate
```

```yaml
# .github/workflows/ci.yml
name: CI Pipeline

on: [push, pull_request]

jobs:
  validate-migrations:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_DB: test_db
          POSTGRES_PASSWORD: test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Run Flyway Validate
        run: |
          docker run --rm --network host \
            -v $(pwd)/sql/migrations:/flyway/sql \
            flyway/flyway:9.22-alpine \
            -url=jdbc:postgresql://localhost:5432/test_db \
            -user=postgres \
            -password=test \
            validate
      
      - name: Run Flyway Migrate
        run: |
          docker run --rm --network host \
            -v $(pwd)/sql/migrations:/flyway/sql \
            flyway/flyway:9.22-alpine \
            -url=jdbc:postgresql://localhost:5432/test_db \
            -user=postgres \
            -password=test \
            migrate
      
      - name: Run Flyway Info
        run: |
          docker run --rm --network host \
            -v $(pwd)/sql/migrations:/flyway/sql \
            flyway/flyway:9.22-alpine \
            -url=jdbc:postgresql://localhost:5432/test_db \
            -user=postgres \
            -password=test \
            info
```

---

### Example 5: Multi-Tenant Migration Strategy

```java
import org.flywaydb.core.Flyway;
import javax.sql.DataSource;
import java.util.List;

/**
 * Demonstrates schema-per-tenant migration pattern
 */
public class MultiTenantMigrationService {
    
    private final DataSource dataSource;
    private final List<String> tenantSchemas;
    
    public MultiTenantMigrationService(DataSource dataSource, List<String> tenants) {
        this.dataSource = dataSource;
        this.tenantSchemas = tenants;
    }
    
    /**
     * Migrate all tenant schemas
     */
    public void migrateAllTenants() {
        for (String tenant : tenantSchemas) {
            migrateTenant(tenant);
        }
    }
    
    /**
     * Migrate single tenant schema
     */
    public void migrateTenant(String tenantSchema) {
        System.out.printf("Migrating tenant: %s%n", tenantSchema);
        
        Flyway flyway = Flyway.configure()
            .dataSource(dataSource)
            .schemas(tenantSchema)
            .locations("classpath:db/migration/tenant")
            .baselineOnMigrate(true)
            .placeholders(Map.of(
                "schema", tenantSchema,
                "tenant_id", tenantSchema.replace("tenant_", "")
            ))
            .table("flyway_schema_history") // Each schema gets own history
            .load();
        
        flyway.migrate();
    }
    
    /**
     * Migrate shared/common schema (cross-tenant data)
     */
    public void migrateSharedSchema() {
        Flyway flyway = Flyway.configure()
            .dataSource(dataSource)
            .schemas("shared")
            .locations("classpath:db/migration/shared")
            .baselineOnMigrate(true)
            .load();
        
        flyway.migrate();
    }
    
    /**
     * Provision new tenant
     */
    public void provisionTenant(String tenantId) {
        String schema = "tenant_" + tenantId;
        
        // Create schema
        try (var conn = dataSource.getConnection();
             var stmt = conn.createStatement()) {
            stmt.execute("CREATE SCHEMA IF NOT EXISTS " + schema);
        } catch (Exception e) {
            throw new RuntimeException("Failed to create schema", e);
        }
        
        // Run migrations
        migrateTenant(schema);
        
        tenantSchemas.add(schema);
    }
}

// Migration file with placeholder
// src/main/resources/db/migration/tenant/V1__Tenant_tables.sql
/*
CREATE TABLE ${schema}.orders (
    id BIGSERIAL PRIMARY KEY,
    tenant_id VARCHAR(50) DEFAULT '${tenant_id}',
    order_number VARCHAR(100) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
*/
```

---

## 6. Recommended Problems to Solve Today

Since Flyway is a tool rather than an algorithm topic, these are **practical exercises** and **system design scenarios**:

### Hands-On Exercises

| # | Exercise | Difficulty | Time |
|---|----------|------------|------|
| 1 | Set up Flyway with H2 in-memory DB, create 3 migrations | Easy | 30 min |
| 2 | Implement expand-contract pattern for column rename | Medium | 45 min |
| 3 | Create Java-based migration with data transformation | Medium | 1 hour |
| 4 | Build Docker + Flyway + PostgreSQL CI pipeline | Medium | 1 hour |
| 5 | Simulate and recover from failed migration | Medium | 30 min |

### System Design Scenarios (Whiteboard Practice)

| # | Scenario | Difficulty | Focus |
|---|----------|------------|-------|
| 6 | Design migration strategy for monolith-to-microservices split | Hard | Schema ownership, data migration |
| 7 | Design zero-downtime deployment with database changes | Hard | Blue-green, expand-contract |
| 8 | Design multi-region database migration synchronization | Hard | Eventual consistency, conflict resolution |
| 9 | Handle 100M row table migration without locking | Hard | Batching, online schema change tools |
| 10 | Design rollback strategy for production incident | Medium | Forward-fix, backup strategies |

### Debugging Scenarios

| Scenario | What to Practice |
|----------|------------------|
| Checksum mismatch error | Understand repair vs recreate |
| Out-of-order migration blocked | `outOfOrder` configuration |
| Failed migration blocking pipeline | Repair and fix workflow |
| Migration timeout on large table | Batching strategies |

---

## 7. 30-Minute Quiz

Test yourself! Answers at the end.

### Questions

**Q1.** What is the correct naming convention for a Flyway versioned migration?
- A) `V1_Create_users.sql`
- B) `V1__Create_users.sql`
- C) `v1__create_users.sql`
- D) `1__Create_users.sql`

**Q2.** What happens when Flyway detects a checksum mismatch for an already-applied migration?
- A) Automatically re-runs the migration
- B) Ignores the change and continues
- C) Fails validation and blocks execution
- D) Creates a new migration automatically

**Q3.** Which command should NEVER be used in production?
- A) `flyway migrate`
- B) `flyway validate`
- C) `flyway clean`
- D) `flyway info`

**Q4.** When are Repeatable migrations (R__) executed relative to Versioned migrations (V)?
- A) Before all versioned migrations
- B) After all pending versioned migrations
- C) In the order they appear in the folder
- D) Only when manually triggered

**Q5.** What is the purpose of `flyway baseline`?
- A) Reset the database to empty state
- B) Mark an existing database at a specific version without running migrations
- C) Create backup of current schema
- D) Validate all migrations

**Q6.** In which database is DDL (CREATE TABLE) NOT transactional?
- A) PostgreSQL
- B) MySQL
- C) SQL Server
- D) All databases support DDL transactions

**Q7.** What is the `expand-contract` pattern used for?
- A) Compressing database storage
- B) Zero-downtime schema changes
- C) Encrypting sensitive data
- D) Optimizing query performance

**Q8.** How should you handle a situation where you need to modify an already-applied migration?
- A) Edit the file and run `flyway repair`
- B) Delete the entry from `flyway_schema_history`
- C) Create a new migration with the corrective change
- D) Use `flyway clean` and start over

**Q9.** What is the purpose of Java-based migrations?
- A) Better performance than SQL
- B) Complex logic that cannot be expressed in SQL
- C) Required for PostgreSQL databases
- D) Automatically generated by Flyway

**Q10.** Which configuration prevents the accidental deletion of all database objects?
- A) `flyway.validateOnMigrate=true`
- B) `flyway.outOfOrder=false`
- C) `flyway.cleanDisabled=true`
- D) `flyway.baselineOnMigrate=true`

---

### Answers

<details>
<summary>Click to reveal answers</summary>

**Q1: B** - `V1__Create_users.sql`
The double underscore is mandatory. Version prefix must be uppercase `V`.

**Q2: C** - Fails validation and blocks execution
Flyway validates checksums to ensure migration integrity. Mismatch indicates tampering.

**Q3: C** - `flyway clean`
This command drops ALL objects in the configured schemas. Always set `cleanDisabled=true` in production.

**Q4: B** - After all pending versioned migrations
Repeatable migrations run last, ordered alphabetically by description.

**Q5: B** - Mark an existing database at a specific version
Used when introducing Flyway to an existing database that already has schema.

**Q6: B** - MySQL
MySQL performs implicit commits on DDL statements. PostgreSQL DDL is fully transactional.

**Q7: B** - Zero-downtime schema changes
Expand (add new), transition (use both), contract (remove old) allows rolling deployments.

**Q8: C** - Create a new migration with the corrective change
Never modify applied migrations. Always move forward with new migrations.

**Q9: B** - Complex logic that cannot be expressed in SQL
Data transformations, external API calls, conditional logic, encryption.

**Q10: C** - `flyway.cleanDisabled=true`
This configuration prevents `flyway clean` from running, protecting production data.

</details>

**Scoring:**
- 9-10: Ready for interview
- 7-8: Review weak areas
- 5-6: Need more practice
- <5: Start from Core Concepts

---

## 8. Further Learning

### Top Resource #1: Article/Video
**"Database Migrations Done Right" by Vlad Mihalcea**
- Covers patterns beyond Flyway basics
- Zero-downtime strategies
- Real-world production scenarios
- URL: Search "Vlad Mihalcea database migrations" (his blog)

### Official Documentation
**Flyway Documentation**
- https://documentation.red-gate.com/fd
- Focus sections:
  - Concepts > Migrations
  - Configuration > Parameters
  - Usage > Command-line
  - Tutorials > Getting Started

### Bonus: Quick Reference
- **GitHub:** https://github.com/flyway/flyway (read issues for real problems)
- **Comparison:** Flyway vs Liquibase (common interview question)
  - Flyway: Convention over configuration, simpler, SQL-native
  - Liquibase: More flexible, XML/JSON/YAML, better rollback

---

## Interview Day Quick Review (5 minutes before)

1. **Naming:** `V{version}__{description}.sql` - DOUBLE underscore
2. **Commands:** migrate, info, validate, repair, baseline (never clean in prod)
3. **Schema History Table:** `flyway_schema_history` tracks everything
4. **Checksum:** Change = failure, create new migration instead
5. **Repeatable:** Runs after versioned, on every checksum change
6. **Zero-downtime:** Expand-contract pattern
7. **Transactions:** PostgreSQL = full DDL transactions; MySQL = implicit commits
8. **Configuration priority:** CLI > Env vars > Config file > Defaults
9. **Best practice:** `cleanDisabled=true`, timestamp versions, one change per file
10. **Java migrations:** For complex logic SQL can't handle

Good luck! 🚀
