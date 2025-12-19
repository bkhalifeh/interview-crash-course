# Ultimate Liquibase Crash Course for FAANG Interviews (1-3 Days)

---

## 1. Core Concepts You Must Know Cold

### What is Liquibase?
Liquibase is an open-source, database-independent library for tracking, managing, and applying database schema changes. It uses **changelogs** to define changes and tracks execution via a **DATABASECHANGELOG** table.

**Why it matters in interviews:** Companies use Liquibase for CI/CD pipelines, version-controlled schema evolution, and zero-downtime deployments. You'll be asked about migration strategies, rollback mechanisms, and how it fits into DevOps workflows.

---

### Changelog (The Master File)
The changelog is the root file that Liquibase reads. It contains or references all changesets.

**Formats supported:** XML, YAML, JSON, SQL

```xml
<!-- db.changelog-master.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
    xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
        http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-latest.xsd">
    
    <!-- Include other changelogs -->
    <include file="db/changelog/changes/001-create-users-table.xml"/>
    <include file="db/changelog/changes/002-add-email-column.xml"/>
    
    <!-- Or use includeAll for a directory -->
    <includeAll path="db/changelog/changes/"/>
</databaseChangeLog>
```

```yaml
# db.changelog-master.yaml
databaseChangeLog:
  - include:
      file: db/changelog/changes/001-create-users-table.yaml
  - includeAll:
      path: db/changelog/changes/
```

---

### ChangeSet (The Atomic Unit)
A changeset is the smallest unit of change. Each changeset is uniquely identified by `id` + `author` + `filename`.

**Critical Rule:** Once a changeset is executed, NEVER modify it. Create a new changeset instead.

```xml
<changeSet id="1" author="john.doe">
    <createTable tableName="users">
        <column name="id" type="BIGINT" autoIncrement="true">
            <constraints primaryKey="true" nullable="false"/>
        </column>
        <column name="username" type="VARCHAR(255)">
            <constraints nullable="false" unique="true"/>
        </column>
        <column name="created_at" type="TIMESTAMP" defaultValueComputed="CURRENT_TIMESTAMP"/>
    </createTable>
</changeSet>
```

```yaml
databaseChangeLog:
  - changeSet:
      id: 1
      author: john.doe
      changes:
        - createTable:
            tableName: users
            columns:
              - column:
                  name: id
                  type: BIGINT
                  autoIncrement: true
                  constraints:
                    primaryKey: true
                    nullable: false
              - column:
                  name: username
                  type: VARCHAR(255)
                  constraints:
                    nullable: false
                    unique: true
```

---

### DATABASECHANGELOG Table
Liquibase automatically creates and maintains this tracking table:

| Column | Purpose |
|--------|---------|
| `ID` | ChangeSet ID |
| `AUTHOR` | ChangeSet author |
| `FILENAME` | Changelog file path |
| `DATEEXECUTED` | When it was run |
| `ORDEREXECUTED` | Sequence number |
| `EXECTYPE` | EXECUTED, FAILED, SKIPPED, RERAN, MARK_RAN |
| `MD5SUM` | Checksum to detect modifications |
| `DESCRIPTION` | Auto-generated description |
| `COMMENTS` | User comments |
| `TAG` | Optional tag for rollback |
| `LIQUIBASE` | Version used |
| `CONTEXTS` | Context labels |
| `LABELS` | Label expressions |
| `DEPLOYMENT_ID` | Groups changesets from same deployment |

**DATABASECHANGELOGLOCK Table:** Prevents concurrent executions.

---

### Rollback Strategies
Liquibase supports three rollback approaches:

**1. Auto-Generated Rollback** (for supported change types):
```xml
<changeSet id="2" author="jane">
    <addColumn tableName="users">
        <column name="email" type="VARCHAR(255)"/>
    </addColumn>
    <!-- Liquibase auto-generates: DROP COLUMN email -->
</changeSet>
```

**2. Custom Rollback:**
```xml
<changeSet id="3" author="jane">
    <sql>UPDATE users SET status = 'ACTIVE' WHERE status IS NULL</sql>
    <rollback>
        <sql>UPDATE users SET status = NULL WHERE status = 'ACTIVE'</sql>
    </rollback>
</changeSet>
```

**3. Empty Rollback (explicitly no rollback):**
```xml
<changeSet id="4" author="jane">
    <sql>INSERT INTO audit_log VALUES ('migration_completed')</sql>
    <rollback/>
</changeSet>
```

**Rollback Commands:**
```bash
# Rollback last N changesets
liquibase rollbackCount 3

# Rollback to a specific tag
liquibase rollback v1.0

# Rollback to a date
liquibase rollbackToDate 2024-01-15T10:00:00

# Generate rollback SQL without executing
liquibase rollbackCountSQL 3
```

---

### Contexts and Labels
**Contexts:** Environment-based execution control

```xml
<changeSet id="5" author="john" context="dev, test">
    <insert tableName="users">
        <column name="username" value="testuser"/>
    </insert>
</changeSet>

<changeSet id="6" author="john" context="prod">
    <sql>CREATE INDEX idx_users_email ON users(email)</sql>
</changeSet>
```

```bash
# Run only dev context
liquibase --contexts=dev update

# Run dev OR test
liquibase --contexts="dev,test" update

# Run dev AND test (use expressions)
liquibase --contexts="dev and test" update
```

**Labels:** More flexible, expression-based filtering

```xml
<changeSet id="7" author="john" labels="jira-123, sprint-5, feature-auth">
    <addColumn tableName="users">
        <column name="password_hash" type="VARCHAR(512)"/>
    </addColumn>
</changeSet>
```

```bash
# Complex label expressions
liquibase --labelFilter="jira-123 or (sprint-5 and feature-auth)" update
```

---

### Preconditions
Guard clauses that determine if a changeset should run:

```xml
<changeSet id="8" author="john">
    <preConditions onFail="MARK_RAN" onError="HALT">
        <not>
            <tableExists tableName="users"/>
        </not>
    </preConditions>
    <createTable tableName="users">
        <column name="id" type="BIGINT"/>
    </createTable>
</changeSet>

<!-- More precondition examples -->
<changeSet id="9" author="john">
    <preConditions onFail="CONTINUE">
        <and>
            <dbms type="postgresql"/>
            <runningAs username="admin"/>
            <columnExists tableName="users" columnName="email"/>
        </and>
    </preConditions>
    <createIndex indexName="idx_email" tableName="users">
        <column name="email"/>
    </createIndex>
</changeSet>
```

**onFail/onError options:**
- `HALT` - Stop execution immediately (default)
- `CONTINUE` - Skip this changeset, continue with next
- `MARK_RAN` - Skip but mark as executed in tracking table
- `WARN` - Log warning, continue execution

---

### Change Types (Most Common)

| Category | Change Types |
|----------|-------------|
| **Table** | `createTable`, `dropTable`, `renameTable`, `setTableRemarks` |
| **Column** | `addColumn`, `dropColumn`, `renameColumn`, `modifyDataType`, `addDefaultValue`, `dropDefaultValue` |
| **Constraints** | `addPrimaryKey`, `dropPrimaryKey`, `addForeignKeyConstraint`, `dropForeignKeyConstraint`, `addUniqueConstraint`, `addNotNullConstraint` |
| **Index** | `createIndex`, `dropIndex` |
| **Data** | `insert`, `update`, `delete`, `loadData`, `loadUpdateData` |
| **SQL** | `sql`, `sqlFile` |
| **Procedural** | `createProcedure`, `createView`, `createSequence` |

---

### Properties and Substitution

```xml
<!-- Define properties -->
<property name="table.name" value="users"/>
<property name="varchar.type" value="VARCHAR(255)" dbms="mysql"/>
<property name="varchar.type" value="VARCHAR2(255)" dbms="oracle"/>

<changeSet id="10" author="john">
    <createTable tableName="${table.name}">
        <column name="name" type="${varchar.type}"/>
    </createTable>
</changeSet>
```

**External properties file (liquibase.properties):**
```properties
driver=org.postgresql.Driver
url=jdbc:postgresql://localhost:5432/mydb
username=admin
password=secret
changeLogFile=db/changelog/db.changelog-master.xml
```

---

### Running Liquibase

```bash
# Apply all pending changes
liquibase update

# Preview SQL without executing
liquibase updateSQL

# Update and tag the database
liquibase update --tag=v1.0

# Check status
liquibase status

# Validate changelog syntax
liquibase validate

# Generate changelog from existing database
liquibase generateChangeLog --changeLogFile=generated.xml

# Compare two databases
liquibase diff --referenceUrl=jdbc:postgresql://prod:5432/mydb

# Synchronize changelog (mark as ran without executing)
liquibase changelogSync

# Clear all checksums (use cautiously)
liquibase clearCheckSums
```

---

### Spring Boot Integration

```yaml
# application.yml
spring:
  liquibase:
    enabled: true
    change-log: classpath:db/changelog/db.changelog-master.xml
    contexts: ${spring.profiles.active}
    default-schema: public
    liquibase-schema: public
    drop-first: false  # NEVER true in production
```

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.liquibase</groupId>
    <artifactId>liquibase-core</artifactId>
</dependency>
```

**Programmatic control:**
```java
@Configuration
public class LiquibaseConfig {
    
    @Bean
    public SpringLiquibase liquibase(DataSource dataSource) {
        SpringLiquibase liquibase = new SpringLiquibase();
        liquibase.setDataSource(dataSource);
        liquibase.setChangeLog("classpath:db/changelog/db.changelog-master.xml");
        liquibase.setContexts("dev,test");
        liquibase.setShouldRun(true);
        return liquibase;
    }
}
```

---

## 2. Most Frequently Asked Interview Questions & Topics

### Easy (Warm-up / Screening)

1. **What is Liquibase and why use it over raw SQL scripts?**
   - Version control for database
   - Database-agnostic
   - Automatic rollback generation
   - Tracking and auditing
   - CI/CD integration

2. **Explain the difference between changelog and changeset.**
   - Changelog = container file (can include other changelogs)
   - Changeset = atomic unit of change (identified by id+author+filename)

3. **What formats does Liquibase support?**
   - XML, YAML, JSON, SQL

4. **How does Liquibase track executed changes?**
   - DATABASECHANGELOG table with checksums

5. **What happens if you modify an already-executed changeset?**
   - Checksum mismatch error on next run
   - Fix: `clearCheckSums` or `validCheckSum` attribute

---

### Medium (Core Technical)

6. **Explain contexts vs labels. When would you use each?**
   - Contexts: Simple environment filtering (dev/test/prod)
   - Labels: Complex expressions, feature flags, JIRA tickets
   - Labels support AND/OR/NOT expressions

7. **How do you handle rollbacks in Liquibase?**
   - Auto-generated (addColumn → dropColumn)
   - Custom rollback blocks
   - Rollback commands: rollbackCount, rollback (tag), rollbackToDate

8. **What are preconditions? Give a production use case.**
   - Guard clauses before changeset execution
   - Use case: Migrating legacy systems where table might already exist

9. **How do you handle database-specific SQL?**
   - `dbms` attribute: `<changeSet dbms="postgresql">`
   - Property substitution with dbms-specific values
   - Separate changelogs per database

10. **Explain the DATABASECHANGELOGLOCK table.**
    - Prevents concurrent Liquibase executions
    - Contains LOCKED, LOCKGRANTED, LOCKEDBY columns
    - Stuck lock fix: `liquibase releaseLocks`

11. **How do you integrate Liquibase with CI/CD?**
    - Run `liquibase validate` in PR checks
    - Run `updateSQL` for review
    - Run `update` in deployment pipeline
    - Use `status --verbose` to check pending changes

---

### Hard (Senior/Staff Level / System Design)

12. **Design a zero-downtime migration strategy for adding a NOT NULL column to a billion-row table.**
    ```
    Step 1: Add column as nullable
    Step 2: Backfill data in batches (outside Liquibase or custom SQL)
    Step 3: Add NOT NULL constraint
    Step 4: Add default value for new rows
    ```
    ```xml
    <changeSet id="add-status-step1" author="senior">
        <addColumn tableName="orders">
            <column name="status" type="VARCHAR(50)"/>
        </addColumn>
    </changeSet>
    
    <changeSet id="add-status-step2" author="senior">
        <sql>
            UPDATE orders SET status = 'PENDING' WHERE status IS NULL
            -- In reality: batch this for large tables
        </sql>
    </changeSet>
    
    <changeSet id="add-status-step3" author="senior">
        <addNotNullConstraint tableName="orders" columnName="status"/>
    </changeSet>
    ```

13. **How do you handle schema migrations in a microservices architecture?**
    - Each service owns its database and changelog
    - Shared libraries for common migration patterns
    - Feature flags for coordinated rollouts
    - Backward-compatible changes only (expand-contract pattern)

14. **Explain the expand-contract pattern for zero-downtime deployments.**
    ```
    Expand Phase:
    1. Add new column (nullable)
    2. Deploy app that writes to both old and new columns
    3. Backfill new column from old
    4. Deploy app that reads from new column
    
    Contract Phase:
    5. Remove old column reads from app
    6. Remove old column writes from app
    7. Drop old column
    ```

15. **How do you manage Liquibase across multiple environments with different schemas?**
    - Contexts for environment-specific changes
    - Property files per environment
    - Profiles in Spring Boot
    - Separate changelog branches (avoid if possible)

16. **What's your strategy for handling failed migrations in production?**
    - Immediate: Fix forward with corrective changeset
    - Rollback: If rollback is defined and safe
    - Manual intervention: For data migrations
    - Post-mortem: Add preconditions to prevent recurrence

17. **Compare Liquibase vs Flyway. When would you choose each?**

    | Aspect | Liquibase | Flyway |
    |--------|-----------|--------|
    | Format | XML/YAML/JSON/SQL | SQL/Java |
    | Rollback | Built-in support | Pro feature only |
    | Learning curve | Steeper | Simpler |
    | Database support | 50+ | 30+ |
    | Diff/Generate | Yes | Limited |
    | Best for | Complex enterprises | Simple projects |

---

## 3. One-Page Cheat Sheet

### Essential Commands
| Command | Purpose |
|---------|---------|
| `liquibase update` | Apply pending changes |
| `liquibase updateSQL` | Preview SQL only |
| `liquibase rollbackCount N` | Rollback last N changesets |
| `liquibase rollback TAG` | Rollback to tag |
| `liquibase status` | Show pending changesets |
| `liquibase validate` | Check changelog syntax |
| `liquibase tag TAG` | Tag current state |
| `liquibase diff` | Compare databases |
| `liquibase generateChangeLog` | Reverse-engineer schema |
| `liquibase clearCheckSums` | Reset all checksums |
| `liquibase releaseLocks` | Force release lock |
| `liquibase changelogSync` | Mark all as executed |

### ChangeSet Attributes
| Attribute | Values | Purpose |
|-----------|--------|---------|
| `id` | String (required) | Unique identifier |
| `author` | String (required) | Change author |
| `context` | CSV | Environment filter |
| `labels` | CSV | Expression-based filter |
| `runOnChange` | true/false | Re-run if modified |
| `runAlways` | true/false | Run every time |
| `failOnError` | true/false | Halt on failure |
| `dbms` | postgresql,mysql,etc | Database filter |
| `runInTransaction` | true/false | Transaction wrapper |
| `runOrder` | first/last | Execution priority |

### Precondition onFail Options
| Option | Behavior |
|--------|----------|
| `HALT` | Stop execution (default) |
| `CONTINUE` | Skip, continue |
| `MARK_RAN` | Skip, mark executed |
| `WARN` | Log warning, continue |

### Common Preconditions
```
tableExists, columnExists, viewExists, indexExists
primaryKeyExists, foreignKeyExists, sequenceExists
rowCount, sqlCheck, changeSetExecuted
dbms, runningAs, changeLogPropertyDefined
```

### Best Practice Rules of Thumb
- ✅ One logical change per changeset
- ✅ Meaningful IDs (timestamps or sequential)
- ✅ Always define rollback for data changes
- ✅ Use contexts for environment differences
- ✅ Validate in CI before merge
- ❌ NEVER modify executed changesets
- ❌ NEVER use `dropFirst` in production
- ❌ NEVER run Liquibase as root DB user
- ❌ AVOID `runAlways` except for views/procedures

### Quick Syntax Reference
```xml
<!-- Create table -->
<createTable tableName="X">
    <column name="id" type="BIGINT" autoIncrement="true">
        <constraints primaryKey="true"/>
    </column>
</createTable>

<!-- Add column -->
<addColumn tableName="X">
    <column name="Y" type="VARCHAR(255)"/>
</addColumn>

<!-- Add foreign key -->
<addForeignKeyConstraint 
    baseTableName="orders" baseColumnNames="user_id"
    referencedTableName="users" referencedColumnNames="id"
    constraintName="fk_orders_users"/>

<!-- Create index -->
<createIndex indexName="idx_X" tableName="Y">
    <column name="Z"/>
</createIndex>

<!-- Raw SQL -->
<sql>UPDATE users SET active = true</sql>
<sqlFile path="scripts/migration.sql"/>

<!-- Load CSV data -->
<loadData tableName="X" file="data.csv"/>
```

---

## 4. Best Practices & Common Mistakes

### What Senior Engineers Do ✅

1. **Atomic Changesets**
   ```xml
   <!-- GOOD: One logical change -->
   <changeSet id="202401151200-add-email" author="senior">
       <addColumn tableName="users">
           <column name="email" type="VARCHAR(255)"/>
       </addColumn>
   </changeSet>
   ```

2. **Idempotent Changes with Preconditions**
   ```xml
   <changeSet id="create-users-table" author="senior">
       <preConditions onFail="MARK_RAN">
           <not><tableExists tableName="users"/></not>
       </preConditions>
       <createTable tableName="users">...</createTable>
   </changeSet>
   ```

3. **Explicit Rollbacks for Complex Changes**
   ```xml
   <changeSet id="merge-columns" author="senior">
       <sql>
           UPDATE users SET full_name = first_name || ' ' || last_name
       </sql>
       <rollback>
           <sql>
               UPDATE users SET first_name = split_part(full_name, ' ', 1),
                              last_name = split_part(full_name, ' ', 2)
           </sql>
       </rollback>
   </changeSet>
   ```

4. **Organized Changelog Structure**
   ```
   db/
   └── changelog/
       ├── db.changelog-master.xml
       ├── releases/
       │   ├── v1.0/
       │   │   ├── 001-create-users.xml
       │   │   └── 002-create-orders.xml
       │   └── v1.1/
       │       └── 001-add-email.xml
       └── data/
           └── seed-data.xml
   ```

5. **Use Valid Timestamp-Based IDs**
   ```xml
   <!-- Pattern: YYYYMMDDHHMMSS-description -->
   <changeSet id="20240115143022-add-user-preferences" author="team">
   ```

6. **Separate Structure from Data**
   ```xml
   <!-- Structure changes: context=all -->
   <changeSet id="1" author="dev" context="dev,test,prod">
       <createTable tableName="config">...</createTable>
   </changeSet>
   
   <!-- Test data: context=dev,test only -->
   <changeSet id="2" author="dev" context="dev,test">
       <insert tableName="config">...</insert>
   </changeSet>
   ```

---

### What Gets You Rejected ❌

1. **Modifying Executed Changesets**
   ```xml
   <!-- WRONG: Changing this after execution -->
   <changeSet id="1" author="dev">
       <addColumn tableName="users">
           <column name="email" type="VARCHAR(255)"/>
           <!-- Later someone adds: type="VARCHAR(500)" -->
           <!-- Result: Checksum error! -->
       </addColumn>
   </changeSet>
   ```
   **Fix:** Create new changeset with `modifyDataType`

2. **Giant Monolithic Changesets**
   ```xml
   <!-- WRONG: Everything in one changeset -->
   <changeSet id="1" author="dev">
       <createTable tableName="users">...</createTable>
       <createTable tableName="orders">...</createTable>
       <createTable tableName="products">...</createTable>
       <addForeignKeyConstraint.../>
       <createIndex.../>
       <insert.../>
   </changeSet>
   ```
   **Fix:** One logical change per changeset

3. **Using `runAlways` Carelessly**
   ```xml
   <!-- WRONG: This runs every deployment -->
   <changeSet id="1" author="dev" runAlways="true">
       <insert tableName="users">
           <column name="id" value="1"/>
       </insert>
   </changeSet>
   <!-- Result: Duplicate key errors! -->
   ```
   **Fix:** Use for views/stored procedures only, with `createOrReplace`

4. **Ignoring Database Differences**
   ```xml
   <!-- WRONG: PostgreSQL-specific syntax for all DBs -->
   <changeSet id="1" author="dev">
       <sql>ALTER TABLE users ADD COLUMN data JSONB</sql>
   </changeSet>
   ```
   **Fix:** Use `dbms` attribute or abstract types

5. **No Rollback for Data Changes**
   ```xml
   <!-- WRONG: Irreversible data modification -->
   <changeSet id="1" author="dev">
       <sql>DELETE FROM users WHERE inactive = true</sql>
       <!-- No rollback = data lost forever -->
   </changeSet>
   ```
   **Fix:** Always include rollback or empty rollback with comment

6. **Hardcoded Environment Values**
   ```xml
   <!-- WRONG: Production password in changelog -->
   <changeSet id="1" author="dev">
       <sql>CREATE USER admin PASSWORD 'secret123'</sql>
   </changeSet>
   ```
   **Fix:** Use property substitution from secure source

7. **Not Validating Before Merge**
   ```bash
   # This should be in every PR check
   liquibase validate
   liquibase updateSQL  # Review the output
   ```

---

## 5. Minimal but Complete Code Examples

### Example 1: Complete Application Setup (Spring Boot)

```xml
<!-- db/changelog/db.changelog-master.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
    xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
        http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-latest.xsd">

    <!-- Properties for database abstraction -->
    <property name="uuid.type" value="UUID" dbms="postgresql"/>
    <property name="uuid.type" value="CHAR(36)" dbms="mysql,h2"/>
    <property name="now" value="CURRENT_TIMESTAMP" dbms="postgresql,mysql"/>
    <property name="now" value="NOW()" dbms="h2"/>

    <!-- Include all release changelogs in order -->
    <include file="db/changelog/v1.0/001-create-users.xml"/>
    <include file="db/changelog/v1.0/002-create-orders.xml"/>
    <include file="db/changelog/v1.1/001-add-user-email.xml"/>
    
    <!-- Seed data for non-production -->
    <include file="db/changelog/seed/test-data.xml" context="dev,test"/>
</databaseChangeLog>
```

```xml
<!-- db/changelog/v1.0/001-create-users.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
        http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-latest.xsd">

    <changeSet id="20240101120000-create-users-table" author="migration-team">
        <comment>Create core users table with audit columns</comment>
        
        <createTable tableName="users">
            <column name="id" type="${uuid.type}">
                <constraints primaryKey="true" nullable="false"/>
            </column>
            <column name="username" type="VARCHAR(100)">
                <constraints nullable="false" unique="true"/>
            </column>
            <column name="password_hash" type="VARCHAR(255)">
                <constraints nullable="false"/>
            </column>
            <column name="status" type="VARCHAR(20)" defaultValue="ACTIVE">
                <constraints nullable="false"/>
            </column>
            <column name="created_at" type="TIMESTAMP" defaultValueComputed="${now}">
                <constraints nullable="false"/>
            </column>
            <column name="updated_at" type="TIMESTAMP"/>
        </createTable>

        <!-- Index for common queries -->
        <createIndex indexName="idx_users_username" tableName="users">
            <column name="username"/>
        </createIndex>
        
        <createIndex indexName="idx_users_status" tableName="users">
            <column name="status"/>
        </createIndex>
        
        <rollback>
            <dropIndex indexName="idx_users_status" tableName="users"/>
            <dropIndex indexName="idx_users_username" tableName="users"/>
            <dropTable tableName="users"/>
        </rollback>
    </changeSet>
</databaseChangeLog>
```

---

### Example 2: Foreign Keys and Relationships

```xml
<!-- db/changelog/v1.0/002-create-orders.xml -->
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
        http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-latest.xsd">

    <changeSet id="20240102100000-create-orders-table" author="migration-team">
        <preConditions onFail="HALT">
            <tableExists tableName="users"/>
        </preConditions>
        
        <createTable tableName="orders">
            <column name="id" type="BIGINT" autoIncrement="true">
                <constraints primaryKey="true"/>
            </column>
            <column name="user_id" type="${uuid.type}">
                <constraints nullable="false"/>
            </column>
            <column name="order_number" type="VARCHAR(50)">
                <constraints nullable="false" unique="true"/>
            </column>
            <column name="total_amount" type="DECIMAL(19,4)">
                <constraints nullable="false"/>
            </column>
            <column name="status" type="VARCHAR(20)" defaultValue="PENDING"/>
            <column name="created_at" type="TIMESTAMP" defaultValueComputed="${now}"/>
        </createTable>
    </changeSet>

    <changeSet id="20240102100001-add-orders-foreign-key" author="migration-team">
        <addForeignKeyConstraint 
            constraintName="fk_orders_user_id"
            baseTableName="orders" 
            baseColumnNames="user_id"
            referencedTableName="users" 
            referencedColumnNames="id"
            onDelete="RESTRICT"
            onUpdate="CASCADE"/>
    </changeSet>

    <changeSet id="20240102100002-add-orders-indexes" author="migration-team">
        <createIndex indexName="idx_orders_user_id" tableName="orders">
            <column name="user_id"/>
        </createIndex>
        <createIndex indexName="idx_orders_status_created" tableName="orders">
            <column name="status"/>
            <column name="created_at"/>
        </createIndex>
    </changeSet>
</databaseChangeLog>
```

---

### Example 3: Complex Migration with Data Transformation

```xml
<!-- db/changelog/v1.1/001-add-user-email.xml -->
<!-- Scenario: Add email column, migrate from username where applicable -->
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
        http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-latest.xsd">

    <!-- Step 1: Add nullable column -->
    <changeSet id="20240115140000-add-email-column" author="migration-team">
        <addColumn tableName="users">
            <column name="email" type="VARCHAR(255)"/>
        </addColumn>
    </changeSet>

    <!-- Step 2: Backfill data (for users with email-like usernames) -->
    <changeSet id="20240115140001-backfill-email" author="migration-team">
        <comment>
            Migrate email from username where username looks like email.
            For production with millions of rows, use batch processing script instead.
        </comment>
        
        <sql dbms="postgresql">
            UPDATE users 
            SET email = username 
            WHERE username LIKE '%@%.%' 
            AND email IS NULL
        </sql>
        <sql dbms="mysql">
            UPDATE users 
            SET email = username 
            WHERE username REGEXP '^[^@]+@[^@]+\\.[^@]+$' 
            AND email IS NULL
        </sql>
        
        <rollback>
            <sql>UPDATE users SET email = NULL WHERE email = username</sql>
        </rollback>
    </changeSet>

    <!-- Step 3: Add constraints after data is clean -->
    <changeSet id="20240115140002-add-email-constraints" author="migration-team">
        <preConditions onFail="HALT">
            <sqlCheck expectedResult="0">
                SELECT COUNT(*) FROM users WHERE email IS NULL
            </sqlCheck>
        </preConditions>
        
        <addNotNullConstraint tableName="users" columnName="email"/>
        <addUniqueConstraint tableName="users" columnName="email" 
                            constraintName="uk_users_email"/>
    </changeSet>

    <!-- Step 4: Create index for email lookups -->
    <changeSet id="20240115140003-add-email-index" author="migration-team">
        <createIndex indexName="idx_users_email" tableName="users">
            <column name="email"/>
        </createIndex>
    </changeSet>
</databaseChangeLog>
```

---

### Example 4: Load Data from CSV and Use SQL Files

```xml
<!-- db/changelog/seed/test-data.xml -->
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
        http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-latest.xsd">

    <!-- Load test users from CSV -->
    <changeSet id="seed-test-users" author="dev-team" context="dev,test">
        <loadData tableName="users" file="db/changelog/seed/users.csv">
            <column name="id" type="STRING"/>
            <column name="username" type="STRING"/>
            <column name="password_hash" type="STRING"/>
            <column name="email" type="STRING"/>
            <column name="status" type="STRING"/>
        </loadData>
        
        <rollback>
            <delete tableName="users">
                <where>username IN ('testuser1', 'testuser2', 'admin')</where>
            </delete>
        </rollback>
    </changeSet>

    <!-- Execute complex SQL from file -->
    <changeSet id="seed-test-orders" author="dev-team" context="dev,test">
        <sqlFile path="db/changelog/seed/orders.sql" 
                 relativeToChangelogFile="true"
                 splitStatements="true"
                 endDelimiter=";"/>
        
        <rollback>
            <sqlFile path="db/changelog/seed/orders-rollback.sql"
                     relativeToChangelogFile="true"/>
        </rollback>
    </changeSet>
</databaseChangeLog>
```

**db/changelog/seed/users.csv:**
```csv
id,username,password_hash,email,status
550e8400-e29b-41d4-a716-446655440001,testuser1,$2a$10$...,test1@example.com,ACTIVE
550e8400-e29b-41d4-a716-446655440002,testuser2,$2a$10$...,test2@example.com,ACTIVE
550e8400-e29b-41d4-a716-446655440003,admin,$2a$10$...,admin@example.com,ACTIVE
```

---

### Example 5: Stored Procedures and Views (runOnChange)

```xml
<!-- db/changelog/v2.0/001-create-views-procedures.xml -->
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
        http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-latest.xsd">

    <!-- Views with runOnChange - re-run when modified -->
    <changeSet id="create-active-users-view" author="dba" runOnChange="true">
        <createView viewName="v_active_users" replaceIfExists="true">
            SELECT u.id, u.username, u.email, u.created_at,
                   COUNT(o.id) as order_count,
                   COALESCE(SUM(o.total_amount), 0) as total_spent
            FROM users u
            LEFT JOIN orders o ON u.id = o.user_id
            WHERE u.status = 'ACTIVE'
            GROUP BY u.id, u.username, u.email, u.created_at
        </createView>
        
        <rollback>
            <dropView viewName="v_active_users"/>
        </rollback>
    </changeSet>

    <!-- Stored procedure - PostgreSQL example -->
    <changeSet id="create-cleanup-procedure" author="dba" 
               runOnChange="true" dbms="postgresql">
        <createProcedure procedureName="cleanup_old_orders">
            CREATE OR REPLACE PROCEDURE cleanup_old_orders(days_old INTEGER)
            LANGUAGE plpgsql
            AS $$
            BEGIN
                DELETE FROM orders 
                WHERE status = 'CANCELLED' 
                AND created_at &lt; NOW() - (days_old || ' days')::INTERVAL;
                
                COMMIT;
            END;
            $$;
        </createProcedure>
        
        <rollback>
            <sql>DROP PROCEDURE IF EXISTS cleanup_old_orders</sql>
        </rollback>
    </changeSet>

    <!-- Stored procedure - MySQL example -->
    <changeSet id="create-cleanup-procedure-mysql" author="dba" 
               runOnChange="true" dbms="mysql">
        <sql splitStatements="false">
            DROP PROCEDURE IF EXISTS cleanup_old_orders;
            
            CREATE PROCEDURE cleanup_old_orders(IN days_old INT)
            BEGIN
                DELETE FROM orders 
                WHERE status = 'CANCELLED' 
                AND created_at &lt; DATE_SUB(NOW(), INTERVAL days_old DAY);
            END;
        </sql>
        
        <rollback>
            <sql>DROP PROCEDURE IF EXISTS cleanup_old_orders</sql>
        </rollback>
    </changeSet>
</databaseChangeLog>
```

---

## 6. Recommended Problems to Solve Today

Since Liquibase is a DevOps/database tool rather than an algorithmic topic, here are practical exercises that mirror real interview scenarios:

### Hands-On Exercises

| # | Exercise | Difficulty | Skills Tested |
|---|----------|------------|---------------|
| 1 | **Set up a complete project** with Spring Boot + PostgreSQL + Liquibase. Create users/orders tables with proper FKs | Easy | Basic setup, changesets |
| 2 | **Add a new NOT NULL column** to an existing table with 1M+ mock rows. Ensure zero downtime | Medium | Migration strategy |
| 3 | **Implement multi-environment config**: dev (with seed data), test (with test fixtures), prod (schema only) | Medium | Contexts, organization |
| 4 | **Create a failing migration and recover**: simulate a partial failure, write recovery changeset | Medium | Error handling, rollback |
| 5 | **Rename a column** that has a foreign key reference from another table | Medium | Constraint management |
| 6 | **Merge two columns** (first_name + last_name → full_name) with proper rollback | Hard | Data migration, rollback |
| 7 | **Split a table** (users → users + user_profiles) while maintaining data integrity | Hard | Complex migration |
| 8 | **Generate changelog from existing database**, clean it up, validate it works on fresh DB | Medium | Reverse engineering |
| 9 | **Set up CI/CD pipeline** (GitHub Actions) that validates changelogs and runs updateSQL on PRs | Medium | DevOps integration |
| 10 | **Design schema for multi-tenant SaaS** with tenant-specific and shared tables using Liquibase | Hard | Architecture, design |

### System Design Questions

1. **Design a database migration system** like Liquibase. What are the core components? How do you handle distributed deployments?

2. **How would you migrate a monolith's single database** to microservices with separate databases using Liquibase?

3. **Design a blue-green deployment strategy** with database migrations. How does Liquibase fit in?

---

## 7. 30-Minute Quiz

Test yourself! Answers are at the end.

### Questions

**Q1.** What uniquely identifies a changeset in Liquibase?
- A) id only
- B) id + author
- C) id + author + filename
- D) id + filename

**Q2.** What happens if you modify an already-executed changeset and run `liquibase update`?
- A) The changeset is re-executed with new changes
- B) Liquibase skips the changeset silently
- C) Liquibase throws a checksum validation error
- D) The old version is rolled back first

**Q3.** Which precondition option marks a changeset as executed without running it?
- A) CONTINUE
- B) HALT
- C) MARK_RAN
- D) SKIP

**Q4.** What is the correct way to handle database-specific SQL for PostgreSQL only?
- A) `<changeSet database="postgresql">`
- B) `<changeSet dbms="postgresql">`
- C) `<changeSet platform="postgresql">`
- D) `<changeSet vendor="postgresql">`

**Q5.** Which attribute should you use for views and stored procedures that may be updated?
- A) `runAlways="true"`
- B) `runOnChange="true"`
- C) `alwaysUpdate="true"`
- D) `rerun="true"`

**Q6.** What command generates SQL without executing it?
- A) `liquibase update --dry-run`
- B) `liquibase updateSQL`
- C) `liquibase preview`
- D) `liquibase validate`

**Q7.** How do you rollback to a specific tag?
- A) `liquibase rollbackTag v1.0`
- B) `liquibase rollback v1.0`
- C) `liquibase rollbackToTag v1.0`
- D) `liquibase revert --tag=v1.0`

**Q8.** What is the DATABASECHANGELOGLOCK table used for?
- A) Storing rollback history
- B) Preventing concurrent Liquibase executions
- C) Locking specific tables during migration
- D) Storing encryption keys

**Q9.** In a Spring Boot application, which property sets the changelog location?
- A) `spring.liquibase.changelog`
- B) `spring.liquibase.change-log`
- C) `liquibase.changelog-file`
- D) `spring.database.changelog`

**Q10.** What is the expand-contract pattern used for?
- A) Compressing database files
- B) Zero-downtime schema migrations
- C) Horizontal database scaling
- D) Data encryption

---

### Answers

<details>
<summary>Click to reveal answers</summary>

**Q1: C** - id + author + filename uniquely identifies a changeset

**Q2: C** - Liquibase calculates MD5 checksums and throws a validation error if they don't match

**Q3: C** - MARK_RAN skips execution but records the changeset as run in DATABASECHANGELOG

**Q4: B** - The `dbms` attribute filters changesets by database type

**Q5: B** - `runOnChange="true"` re-executes when the changeset content changes (use for views/procedures)

**Q6: B** - `updateSQL` outputs the SQL that would be executed without running it

**Q7: B** - `liquibase rollback <tag>` rolls back to the specified tag

**Q8: B** - The lock table prevents multiple Liquibase instances from running simultaneously

**Q9: B** - `spring.liquibase.change-log` (note the hyphen, not camelCase)

**Q10: B** - Expand-contract allows schema changes without breaking running applications (add new → migrate → remove old)

</details>

---

## 8. Further Learning

### Top 2 Resources Only

1. **Official Documentation (Primary Reference)**
   - https://docs.liquibase.com/
   - Start with: [Get Started](https://docs.liquibase.com/start/home.html) → [Concepts](https://docs.liquibase.com/concepts/home.html) → [Best Practices](https://docs.liquibase.com/concepts/bestpractices.html)

2. **Video Course: Liquibase Fundamentals**
   - [Liquibase University](https://learn.liquibase.com/) (Free certification available)
   - Or YouTube: "Liquibase Tutorial" by Java Brains (concise overview)

---

**Study Timeline:**
- **Day 1:** Core concepts + Examples 1-2 + Easy questions
- **Day 2:** Examples 3-5 + Medium questions + Exercises 1-5
- **Day 3:** Hard questions + System design + Quiz + Remaining exercises

Good luck with your interviews! 🚀
