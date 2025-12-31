---
name: postgresql-cookbook
description: PostgreSQL patterns and best practices cookbook. Use this skill when working with PostgreSQL databases to reference proven patterns for database design, queries, and operations.
allowed-tools: Read, Grep, Glob, Edit, Write
---

# PostgreSQL Cookbook

This cookbook provides proven PostgreSQL patterns and best practices for common database operations.

## Naming Conventions

### Overview

Always use **snake_case** for table names, column names, and all SQL identifiers. This is the de facto standard in the PostgreSQL ecosystem and prevents issues with case sensitivity and identifier quoting.

### Why snake_case?

1. **Case insensitivity**: PostgreSQL folds unquoted identifiers to lowercase. Using camelCase requires explicit quoting everywhere.
2. **Community standard**: The PostgreSQL community overwhelmingly uses snake_case.

### Pattern

**Good - snake_case:**
```sql
CREATE TABLE user_account (
  user_id BIGSERIAL PRIMARY KEY,
  email_address TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  last_login_at TIMESTAMPTZ
);

CREATE INDEX idx_user_account_email ON user_account(email_address);

SELECT user_id, email_address, created_at
FROM user_account
WHERE last_login_at > NOW() - INTERVAL '30 days';
```

**Bad - camelCase (requires quoting):**
```sql
-- DON'T DO THIS
CREATE TABLE "userAccount" (
  "userId" BIGSERIAL PRIMARY KEY,
  "emailAddress" TEXT NOT NULL,
  "createdAt" TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Every query requires quotes
SELECT "userId", "emailAddress"
FROM "userAccount"
WHERE "createdAt" > NOW();
```

### Best Practices

1. **Table names**: Singular snake_case (e.g., `user_account`, `order_item`)
2. **Column names**: snake_case (e.g., `user_id`, `email_address`, `created_at`)
3. **Indexes**: Prefix with `idx_` followed by table and columns (e.g., `idx_user_email`)
4. **Functions**: snake_case verbs (e.g., `cleanup_old_logs`, `process_pending_jobs`)
5. **Never quote**: If you find yourself needing quotes, rename to snake_case instead

## Stored Procedures

### Overview

Stored procedures (functions) in PostgreSQL can be useful for specific use cases, but should be used judiciously. They are acceptable when kept small and focused on database operations, but should never contain business logic.

### When to Use Stored Procedures

Stored procedures are appropriate for:

1. **Database maintenance tasks**: Cleanup operations, scheduled jobs
2. **Simple data transformations**: Formatting, calculations that stay within the database
3. **Triggers**: Automated actions on INSERT/UPDATE/DELETE

### Key Constraints

1. **Keep them small**: Functions should be short and focused on a single task
2. **No business logic**: Business rules belong in application code where they can be tested, versioned, and easily modified
3. **Avoid complex queries**: PostgreSQL cannot estimate execution costs for stored procedures, making query optimisation difficult. For complex queries, keep the SQL in your application code where the query planner can optimise it properly.

### Best Practices

1. **Keep functions small and focused**: Each function should do one thing well
2. **Application code for business logic**: Never put business rules in the database
3. **Avoid for complex queries**: PostgreSQL can't estimate costs for stored procedures
4. **Document purpose clearly**: Explain why the function exists in the database rather than application code
5. **Consider application-side alternatives**: Question whether the function could be better implemented in your application

## Scheduled Deletion of Old Records with pg_cron

### Overview

Use pg_cron to automatically delete old records on a schedule. This is useful for cleaning up audit logs, temporary data, or any time-based records without manual intervention or external cron jobs.

### Prerequisites

Install and enable the pg_cron extension (requires superuser privileges):

```sql
CREATE EXTENSION IF NOT EXISTS pg_cron;
```

### Implementation

Create a stored procedure for the deletion logic:

```sql
CREATE OR REPLACE FUNCTION cleanup_old_audit_logs()
RETURNS void
LANGUAGE plpgsql
AS $$
BEGIN
  DELETE FROM audit_logs
  WHERE created_at < NOW() - INTERVAL '90 days'
  LIMIT 10000;

  RAISE NOTICE 'Deleted old audit logs at %', NOW();
END;
$$;
```

Schedule the job to run daily:

```sql
SELECT cron.schedule(
  'delete-old-audit-logs',
  '0 3 * * *',  -- 3 AM daily
  'SELECT cleanup_old_audit_logs()'
);
```

### Best Practices

1. **Use appropriate retention periods**: Consider regulatory requirements and business needs
2. **Add LIMIT clause for large deletions**: Prevent long-running transactions that could block other operations
3. **Monitor job execution**: Check `cron.job_run_details` for failures
4. **Use indexes**: Ensure the date column is indexed for efficient deletion
   ```sql
   CREATE INDEX idx_audit_logs_created_at ON audit_logs(created_at);
   ```
5. **Consider partitioning**: For very large tables, partition by date and drop old partitions instead of DELETE
6. **Test timing**: Schedule during low-traffic periods to minimize impact

## Competing for Work with SELECT FOR UPDATE SKIP LOCKED

### Overview

Use `SELECT FOR UPDATE SKIP LOCKED` to allow multiple service instances to safely compete for work items from a shared queue without blocking each other or processing the same item twice.

### Implementation

```sql
-- Worker process claims the next available job
UPDATE jobs
SET
  status = 'processing',
  started_at = NOW(),
  worker_id = 'worker-123',
  attempts = attempts + 1
WHERE id = (
  SELECT id
  FROM jobs
  WHERE status = 'pending'
    AND attempts < 10
  ORDER BY created_at ASC
  FOR UPDATE SKIP LOCKED
  LIMIT 1
)
RETURNING *;
```

### How It Works

- `FOR UPDATE`: Locks the selected row(s) for update
- `SKIP LOCKED`: Skips rows that are already locked by other transactions, preventing workers from waiting
- `LIMIT 1`: Each worker claims exactly one job
- `ORDER BY`: Ensures FIFO processing (oldest jobs first)
- `attempts < 10`: Excludes jobs that have failed too many times

### Multi-Table Queries

When selecting across multiple tables (e.g., joins), use `FOR UPDATE OF table_name` to explicitly specify which table's rows to lock:

```sql
UPDATE jobs
SET
  status = 'processing',
  started_at = NOW(),
  attempts = attempts + 1
WHERE id = (
  SELECT j.id
  FROM jobs j
  JOIN job_metadata m ON j.id = m.job_id
  WHERE j.status = 'pending'
    AND j.attempts < 10
  ORDER BY j.created_at ASC
  FOR UPDATE OF j SKIP LOCKED
  LIMIT 1
)
RETURNING *;
```

Without `OF table_name`, PostgreSQL may lock rows from all joined tables, potentially causing race conditions where multiple workers claim the same job.

### Best Practices

1. **Always use within a transaction**: Ensure the claim and subsequent work are atomic
2. **Add appropriate indexes**: Index the status and ordering columns
   ```sql
   CREATE INDEX idx_jobs_pending ON jobs(status, created_at) WHERE status = 'pending';
   ```
3. **Include worker identification**: Track which worker is processing each job for debugging
4. **Implement retry limits**: Increment attempts counter and exclude jobs exceeding maximum retries
5. **Set timeouts**: Implement mechanisms to reclaim jobs from failed workers
6. **Use explicit table locking for joins**: Always use `FOR UPDATE OF table_name` when joining tables

## Advisory Locks for Concurrency Control

### Overview

Use advisory locks to control concurrency when you don't have a specific database row to lock. Advisory locks use arbitrary numbers as lock identifiers, making them ideal for serializing operations per user, resource, or any other logical grouping.

### Implementation

```sql
-- Lock using a numeric user ID
SELECT pg_advisory_xact_lock(user_id);

-- Perform operations knowing only one transaction can proceed for this user
INSERT INTO user_actions (user_id, action, created_at)
VALUES (user_id, 'some_action', NOW());

-- Lock is automatically released at transaction end
```

For non-numeric identifiers (like email addresses), hash them to get a number:

```sql
-- Lock using a hashed email address
SELECT pg_advisory_xact_lock(hashtext('user@example.com'));

-- Perform operations
-- ...
```

### Transaction vs Session Locks

- `pg_advisory_xact_lock(key)`: Lock is automatically released when the transaction ends (recommended)
- `pg_advisory_lock(key)`: Lock persists until explicitly released with `pg_advisory_unlock(key)` or session ends

Always prefer transaction-level locks (`pg_advisory_xact_lock`) to avoid accidentally holding locks indefinitely.

### Use Cases

1. **Serialize operations per user**: Ensure only one operation runs at a time for a specific user
2. **Coordinate external resources**: Lock on a hash of an external resource identifier
3. **Rate limiting**: Control access to shared resources without database rows
4. **Prevent duplicate processing**: Use a hash of request parameters to prevent concurrent duplicate requests

### Best Practices

1. **Use transaction-level locks**: Prefer `pg_advisory_xact_lock` over `pg_advisory_lock` for automatic cleanup
2. **Use consistent hashing**: When hashing strings, use the same hash function throughout your application
3. **Document lock keys**: Keep a registry of what lock numbers represent in your application
4. **Consider lock scope**: Advisory locks are database-wide, not table or row specific
5. **Use within transactions**: Always use advisory locks inside explicit transactions for predictable behavior

## Index Strategy

### Overview

Proper indexing is critical for query performance. Add indexes on columns that are frequently queried, use partial indexes to reduce index size, and create indexes concurrently on production systems to avoid blocking.

### Pattern

**Single column indexes** for fields used in WHERE clauses:

```sql
CREATE INDEX CONCURRENTLY idx_users_email ON users(email);
CREATE INDEX CONCURRENTLY idx_orders_user_id ON orders(user_id);
```

**Partial indexes** to exclude irrelevant rows and reduce index size:

```sql
-- Exclude soft-deleted rows
CREATE INDEX CONCURRENTLY idx_users_active
ON users(email)
WHERE deleted_at IS NULL;

-- Exclude completed jobs to avoid index updates when workers claim jobs
CREATE INDEX CONCURRENTLY idx_jobs_active
ON jobs(status, created_at)
WHERE status <> 'completed';
```

When a worker claims a job and changes status from 'pending' to 'processing', using `status <> 'completed'` avoids an index update since both statuses remain in the index. This is more efficient than `status = 'pending'` which would require index maintenance on every status change.

**Compound indexes** for specific query performance. Put the most general column first so it can be used independently:

```sql
-- Can be used for queries filtering on status alone, or status + created_at
CREATE INDEX CONCURRENTLY idx_jobs_status_created
ON jobs(status, created_at);

-- Example queries that benefit:
-- SELECT * FROM jobs WHERE status = 'pending';  -- Uses index
-- SELECT * FROM jobs WHERE status = 'pending' ORDER BY created_at;  -- Uses index
```

### CREATE INDEX CONCURRENTLY

Always use `CONCURRENTLY` when adding indexes to large tables in production:

```sql
CREATE INDEX CONCURRENTLY idx_users_email ON users(email);
```

**Why it matters:**
- Without `CONCURRENTLY`: Index creation locks the table, blocking all writes
- With `CONCURRENTLY`: Index builds in background, allowing normal operations
- Critical for deployments: Non-concurrent index creation can timeout application deployments

**Trade-offs:**
- Takes longer to build than regular index creation
- Cannot be used inside a transaction
- Worth it to avoid blocking production traffic

### Best Practices

1. **Index queried columns**: Add single column indexes on any fields used in WHERE clauses
2. **Use partial indexes**: Exclude irrelevant rows (soft deletes, completed jobs) to reduce index size and maintenance
3. **Compound index column order**: Put the most general column first for maximum reusability
4. **Always use CONCURRENTLY in production**: Avoid blocking writes and deployment timeouts on large tables
5. **Monitor index usage**: Check `pg_stat_user_indexes` to identify unused indexes

## Database Notice Integration with Application Logging

### Overview

PostgreSQL can emit NOTICE, WARNING, and other messages during query execution. Integrate these notices with your application's logging system so database-level events are visible alongside application logs. This is particularly useful for debugging triggers, functions, and constraints.

#### Key Benefits

1. **Unified Logging**: Database notices appear in application logs
2. **Debugging Aid**: See PostgreSQL function messages in application logs
3. **Severity Mapping**: PostgreSQL severities map to appropriate log levels
4. **No Additional Tools**: No need to check database logs separately

#### Pattern

```typescript
// src/Database.ts
import { Pool } from 'pg';

export default class Database {
  private pool: Pool;
  private severityMap: Record<string, "debug" | "info" | "warn" | "error" | "fatal"> = {
    DEBUG: "debug",
    LOG: "debug",
    INFO: "info",
    NOTICE: "info",
    WARNING: "warn",
    ERROR: "error",
    FATAL: "fatal",
    PANIC: "fatal",
  };

  constructor(config: any) {
    this.pool = new Pool(config);

    // Listen for errors on idle clients
    this.pool.on("error", (error) => {
      logger.error("Database pool error", error);
    });

    // Attach notice handler when new clients connect
    this.pool.on("connect", (client) => {
      client.on("notice", (notice) => {
        const { message = "Database notice", severity = "NOTICE", ...params } = notice;
        const logLevel = this.severityMap[severity?.toUpperCase()] || "info";
        logger[logLevel](message, params);
      });
    });
  }
}
```

### Best Practices

1. **Use the "connect" event**: Attach handlers when clients are physically created, not on each checkout
2. **Map severities consistently**: Use an object map for PostgreSQL severities to log levels
3. **Handle both pool and client errors**: Listen on pool for idle client errors
4. **Include notice metadata**: Pass through additional fields from notice object

## SQL Query Logging and Test Debugging

### Overview

Enable SQL query logging to see exactly what queries PostgreSQL executes. Combine this with RAISE NOTICE in tests to correlate test execution with database queries, making it easier to debug test failures and understand query performance.

### Enabling SQL Query Logging

Configure logging via connection options:

```typescript
const pool = new Pool({
  host: 'localhost',
  database: 'myapp_test',
  user: 'test_user',
  password: 'test_password',
  options: '-c log_statement=all -c client_min_messages=notice'
});
```

### Using RAISE NOTICE for Test Debugging

Add test context to database logs using RAISE NOTICE in test hooks:

```typescript
import { describe, it, beforeEach } from 'node:test';
import Database from '../src/Database';

describe("UserService", () => {
  let database: Database;

  beforeEach(async (ctx) => {
    // Log the current test name to the database
    await database.query(
      `DO $$ BEGIN RAISE NOTICE 'Starting test: %', $1; END $$`,
      [ctx.name]
    );

    // Clean up test data
    await database.nuke();
  });

  it("creates a new user", async () => {
    // Test implementation
    // All SQL queries will be logged with test context above
  });
});
```

### Example Log Output

PostgreSQL logs will show:

```
NOTICE:  Starting test: creates a new user
NOTICE:  Nuked tables: users, orders, products
LOG:  statement: INSERT INTO users (name, email) VALUES ('John', 'john@example.com')
LOG:  statement: SELECT * FROM users WHERE email = 'john@example.com'
```

This makes it easy to see which queries correspond to which tests, invaluable for debugging test failures or optimizing slow tests.

#### Best Practices

1. **Use connection options for test environments**: Set options per connection rather than modifying server configuration
2. **Log test names in beforeEach**: Use `ctx.name` to identify which test is running
3. **Disable in production**: Only enable `log_statement=all` in development/test environments (very verbose)
4. **Use RAISE NOTICE liberally in functions**: Add context to complex stored procedures for debugging
