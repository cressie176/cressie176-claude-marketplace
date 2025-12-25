---
name: typescript-tdd-cookbook
description: TypeScript TDD patterns and implementation cookbook. Use this skill when generating TypeScript code to reference proven test-driven development patterns.
allowed-tools: Read
---

# TypeScript TDD Cookbook

Great tests tell a simple, focused story. You see instantly what behaviour is under test and which values actually matter. That means pulling the noise out of sight and keeping only the essential details in view. The patterns in this cookbook exist to support that aim: they strip away clutter, clarify intent and help your tests express the behaviour you actually care about rather than wrestling with boilerplate, ceremony or verbose APIs. At the heart of this is maximising the signal-to-noise ratio: every line in your test should contribute to understanding the behaviour.

**Note:** While examples use Node's test framework and assert library, the principles apply equally to Jest, Vitest, Mocha, or any other testing framework.

## Use Node's Built-in Test Framework

### Overview

Node's built-in test framework and assertion library are more than adequate for most testing needs. Unless your codebase already uses a third-party framework, prefer Node's native tools to reduce dependencies, maintenance burden, and cognitive overhead. Node's assertion library is deliberately minimal, avoiding the verbose fluent APIs that add noise without adding clarity.

### Key Benefits

1. **Zero Dependencies**: No additional packages to install, update, or maintain
2. **High Signal-to-Noise**: Concise assertions without fluent API ceremony
3. **Reduced Maintenance**: One less dependency to track and upgrade
4. **Aliasing Support**: Import with short names to minimise boilerplate

### Usage Examples

#### Import with Aliases

```typescript
import { describe, it, before, beforeEach, after } from 'node:test';
import { equal as eq, notEqual as neq, ok, rejects, match } from 'node:assert/strict';
```

#### Prefer Multiple Assertions Over Deep Equality

```typescript
it("creates user with correct properties", () => {
  const user = createUser({ name: 'Alice', age: 30 });

  eq(user.name, 'Alice');
  eq(user.age, 30);
  ok(user.id);
});
```

Deep equality assertions make failures harder to debug because you can't immediately see which property failed. Multiple `eq()` calls are clearer and pinpoint failures precisely. Use `ok()` for unpredictable values like generated IDs, and `eq()` for everything else.

**Avoid Fluent APIs**: Libraries with `.not.toBe()` or `.toHaveProperty()` chains add visual noise. Compare:

```typescript
// Fluent API (noisy)
expect(user.name).not.toBe(undefined);
expect(user.age).toEqual(30);
expect(user).toHaveProperty('id');

// Node assertions (signal)
eq(user.name, 'Alice');
eq(user.age, 30);
ok(user.id);
```

The Node approach removes ceremony, letting you focus on what's being tested rather than navigating API syntax.

## ObjectMother / TestDataFactory Pattern

### Overview

Test data is often where the readability of a test falls apart: sprawling, highly duplicated fixtures swamp what the test is really trying to say. Static, centralised fixtures help a little, but they soon become cumbersome and brittle as every new case needs yet more hard coded data. A better approach is to randomly generate realistic data, then explicitly override the few values you care about. The result is a cleaner test suite with far clearer intent.

**Note:** If your project already uses a random data generator (like faker, chance, or casual), prefer that library over adding falso as a new dependency.

### Key Benefits

1. **Focus on Intent**: Tests only specify values they actually assert, making the test's purpose clear
2. **Reduce Duplication**: Shared data generation logic in one place
3. **Avoid Brittleness**: Random data prevents tests from relying on specific values defined elsewhere

### Required Dependencies

```bash
npm install -D @ngneat/falso immer
```

### Implementation

```typescript
// test/TestData.ts
import { randCompanyName, randNumber, randEmail } from '@ngneat/falso';
import { produce } from 'immer';

interface Company {
  name: string;
  number: number;
  email: string;
}

export default class TestData {
  static company(recipe?: (draft: Company) => void): Company {
    return produce({
      name: randCompanyName(),
      number: randNumber({ min: 1000, max: 9999 }),
      email: randEmail()
    }, recipe);
  }
}
```

### Usage Example

```typescript
import { equal as eq } from 'node:assert/strict';
import TestData from './TestData';

it("creates company with specific details", async () => {
  const data = TestData.company((draft) => {
    draft.name = "Stark Industries";
    draft.number = 1234;
    draft.email = "tony@stark.com";
  });

  const created = await companyService.createCompany(data);

  eq(created.name, "Stark Industries");
  eq(created.number, 1234);
  eq(created.email, "tony@stark.com");
});
```

This pattern makes tests self-documenting: the values you set are the values you care about.

## Test Client Pattern

### Overview

The most effective tests make their intent unmistakable. Yet low level details such as HTTP calls, payload construction and plumbing are often where that clarity disappears. When every test is cluttered with request setup and wiring, it becomes hard to see what you are actually trying to prove. The Test Client pattern fixes this by wrapping common interactions in clear, expressive methods that speak the language of your domain. Each test then reads as a sequence of meaningful actions rather than a technical recipe, giving you a suite that is easier to read, easier to change and far closer to how you actually think about the system.

### Key Benefits

1. **Domain Language**: Methods express business operations, not HTTP mechanics
2. **Centralized Logic**: Authentication, headers, and payload construction in one place
3. **Easy Maintenance**: API changes require updating only the client

### Implementation

```typescript
// test/TestClient.ts
export default class TestClient {
  constructor(private baseUrl: string) {}

  async createCompany(data: any) {
    const response = await fetch(`${this.baseUrl}/api/companies`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(data)
    });
    const body = await response.json();
    return { status: response.status, body };
  }
}
```

### Usage Example

```typescript
import { equal as eq, ok } from 'node:assert/strict';
import TestClient from './TestClient';
import TestData from './TestData';

it("creates company", async () => {
  const client = new TestClient('http://localhost:3000');
  const data = TestData.company();

  const { status, body: created } = await client.createCompany(data);

  eq(status, 201);
  ok(created.id);
});
```

The test client abstracts HTTP complexity while tests focus on behaviour and assertions.

## Controlling Time in Tests

### Overview

Testing time-based behaviour requires control over what "now" means. Without it you're left with fragile tests that depend on real system time or arbitrary sleeps. A better approach is to use a library that allows you to override the current time in tests.

**Note:** If your project already uses date-fns or another date library, prefer that library's time control mechanism over adding luxon as a new dependency.

### Key Benefits

1. **Deterministic Tests**: No flaky tests due to timing issues
2. **Fast Time Travel**: Instantly test expiry and scheduled events
3. **Clear Assertions**: Test exact time values without approximations

### Required Dependencies

```bash
npm install luxon
```

### Implementation

```typescript
import { DateTime, Settings } from 'luxon';

function calculateExpiry(minutes: number): Date {
  const now = Settings.now();
  return DateTime.fromMillis(now).plus({ minutes }).toJSDate();
}
```

### Usage Example

```typescript
import { equal as eq } from 'node:assert/strict';
import { DateTime, Settings } from "luxon";

it("calculates expiry time", () => {
  const now = new Date("1997-08-29T02:14:00Z").getTime();
  Settings.now = () => now;

  const expiry = calculateExpiry(15);

  const expected = DateTime.fromMillis(now).plus({ minutes: 15 }).toJSDate();
  eq(expiry.toISOString(), expected.toISOString());
});
```

This pattern eliminates timing flakiness and makes time-dependent tests fast and reliable.

## Testing Async Rejection

### Overview

Good tests cover both success and failure paths. When testing code that should reject with specific errors, Node's built-in `assert.rejects()` provides clear, expressive assertions without try-catch clutter.

**Note:** For synchronous errors, use `assert.throws()` instead of `assert.rejects()`.

### Key Benefits

1. **Clear Intent**: Explicitly shows this test expects rejection
2. **Flexible Validation**: Assert exact messages or match patterns for unpredictable errors
3. **Clean Syntax**: No manual promise handling or try-catch blocks

### Usage Examples

#### Assert Exact Error Message

```typescript
import { rejects } from 'node:assert/strict';

it("rejects underage users", async () => {
  await rejects(() => validateAge(16), { message: 'Must be 18 or older' });
});
```

#### Match Message and Stack Trace

```typescript
import { rejects, match } from 'node:assert/strict';

it("rejects invalid email from validator", async () => {
  await rejects(() => userService.createUser(data), (err: Error) => {
    match(err.message, /invalid email/i);
    match(err.stack!, /at EmailValidator\.validate/);
    return true;
  });
});
```

This pattern makes error validation tests clear and maintainable.

## Stubbing HTTP with [Nock](https://github.com/nock/nock)

### Overview

Testing code that makes HTTP requests without hitting real servers requires stubbing those requests. Nock intercepts HTTP calls at the Node.js level, allowing you to simulate successful responses, HTTP errors, timeouts, and connection failures without any network activity.

### Key Benefits

1. **Fast Tests**: No real network calls means instant test execution
2. **Reliable Tests**: No dependency on external services or network conditions
3. **Comprehensive Coverage**: Test error scenarios that are hard to reproduce in real environments

### Required Dependencies

```bash
npm install -D nock
```

### Setup

```typescript
import { describe, it, before, afterEach } from 'node:test';
import { ok } from 'node:assert/strict';
import nock from 'nock';

describe("UserService", () => {

  before(() => {
    nock.disableNetConnect();
  });

  afterEach(() => {
    ok(nock.isDone(), 'pending mocks: %j', nock.pendingMocks());
    nock.cleanAll();
  });
});
```

Always call `nock.disableNetConnect()` to prevent accidental real HTTP requests, check `nock.isDone()` with `nock.pendingMocks()` to ensure all stubbed calls were made, and `nock.cleanAll()` to clear interceptors.

### Usage Examples

#### Stub Successful HTTP Response

```typescript
import { equal as eq } from 'node:assert/strict';
import nock from 'nock';

it("fetches user data", async () => {
  nock('https://api.example.com')
    .get('/users/123')
    .reply(200, { id: 123, name: 'Alice' });

  const user = await userService.fetchUser(123);

  eq(user.name, 'Alice');
});
```

#### Stub HTTP Error Response

```typescript
import { equal as eq } from 'node:assert/strict';
import nock from 'nock';

it("handles 404 not found", async () => {
  nock('https://api.example.com')
    .get('/users/999')
    .reply(404, { error: 'User not found' });

  const result = await userService.fetchUser(999);

  eq(result, null);
});
```

#### Simulate Network Timeout

```typescript
import { rejects, match } from 'node:assert/strict';
import nock from 'nock';

it("handles request timeout", async () => {
  nock('https://api.example.com')
    .get('/users/123')
    .delay(5000)
    .reply(200, { id: 123 });

  await rejects(() => userService.fetchUser(123), (err: Error) => {
    match(err.message, /timeout/i);
    return true;
  });
});
```

#### Simulate Connection Error

```typescript
import { rejects, match } from 'node:assert/strict';
import nock from 'nock';

it("handles connection failure", async () => {
  nock('https://api.example.com')
    .get('/users/123')
    .replyWithError('ECONNREFUSED');

  await rejects(() => userService.fetchUser(123), (err: Error) => {
    match(err.message, /ECONNREFUSED/);
    return true;
  });
});
```

This pattern gives you complete control over HTTP behaviour in tests without external dependencies.

### Tips

#### Debugging Unmatched Interceptors

If interceptors aren't matching requests as expected, enable nock's debug mode to see exactly what requests are being made:

```bash
DEBUG=nock.* npm test
```

This will log all HTTP requests and show why interceptors aren't matching, helping you identify URL mismatches, missing headers, or incorrect HTTP methods.

#### Persistent Stubs for Third-Party Polling

Use `.persist()` to keep interceptors active across multiple requests, useful for third-party code that polls external services:

```typescript
before(() => {
  nock('https://unleash.example.com')
    .get('/api/client/features')
    .reply(200, { features: [] })
    .persist();
});
```

This prevents tests from failing when libraries like Unleash poll for feature flags or configuration updates.

## Database Cleanup with Nuke

### Overview

Integration tests that write to databases need clean state between runs. Rather than manually truncating specific tables or recreating the database, a dynamic nuke function discovers and clears all tables automatically. This approach scales effortlessly as your schema evolves without maintaining cleanup code.

**Note:** The implementation below is PostgreSQL-specific. If using MySQL, SQLite, or other databases, you'll need to adapt the SQL syntax and system catalog queries accordingly.

### Key Benefits

1. **Zero Maintenance**: Automatically finds and clears all tables without hardcoding names
2. **Fast Cleanup**: Truncate is faster than dropping and recreating databases
3. **Safe by Design**: Built-in safeguards prevent running on production databases

### Implementation

#### Database Migration (PostgreSQL)

```sql
DO $$
BEGIN
  -- Only create nuke function if explicitly enabled
  IF current_setting('app.allow_nuke', true) = 'true' THEN
    CREATE OR REPLACE FUNCTION nuke_data()
    RETURNS void
    LANGUAGE plpgsql
    AS $func$
    DECLARE
      table_names text;
    BEGIN
      -- Disable foreign key constraints and triggers
      SET session_replication_role = replica;

      -- Get all user tables except migrations (you may need to change this for your project)
      SELECT string_agg(tablename, ', ')
      INTO table_names
      FROM pg_tables
      WHERE schemaname = 'public'
      AND tablename != 'migrations';

      -- Truncate all tables if any exist
      IF table_names IS NOT NULL THEN
        RAISE NOTICE 'Nuking tables: %', table_names;      
        EXECUTE 'TRUNCATE ' || table_names || ' RESTART IDENTITY CASCADE';
      END IF;

      -- Restore normal constraint checking
      SET session_replication_role = DEFAULT;
    END;
    $func$;
  END IF;
END;
$$;
```

#### Database Classes

First, your production Database class:

```typescript
// src/Database.ts
import { Pool } from 'pg';

export default class Database {
  protected pool: Pool;

  constructor(config: any) {
    this.pool = new Pool(config);
  }

  async start(): Promise<void> {
    // Run migrations or other startup tasks
  }

  async stop(): Promise<void> {
    await this.pool.end();
  }
}
```

Then create a TestDatabase subclass that adds the `nuke()` method:

```typescript
// test/TestDatabase.ts
import Database from '../src/Database';

export default class TestDatabase extends Database {
  async nuke(): Promise<void> {
    await this.pool.query('SELECT nuke_data()');
  }
}
```

### Usage Example

```typescript
import { describe, it, before, beforeEach, after } from 'node:test';
import TestDatabase from './TestDatabase';

describe("UserService", () => {
  let database: TestDatabase;

  before(async () => {
    database = new TestDatabase({
      host: 'localhost',
      database: 'myapp_test',
      user: 'test_user',
      password: 'test_password',
      options: '-c app.allow_nuke=true'
    });
    await database.start();
  });

  beforeEach(async () => {
    await database.nuke();
  });

  after(async () => {
    await database.stop();
  });
});
```

This pattern ensures clean test isolation without manually tracking which tables need cleanup.

## Testing Event Emitters

### Overview

Event emitters are common in Node.js applications, but testing them requires care to avoid listener leaks and race conditions. The key challenges are ensuring listeners are cleaned up after tests, waiting for events to fire before making assertions, and preventing the test from completing before assertions run. Using `emitter.once()` automatically removes listeners, the `done` callback ensures the test waits for the event, and wrapping the entire test in an IIFE lets you use async/await for setup while working with Node's callback-based test API.

### Key Benefits

1. **Automatic Cleanup**: `once()` removes listeners after firing, preventing cross-test contamination
2. **Async Setup**: IIFE pattern allows async operations for test setup
3. **Clear Intent**: Tests explicitly show what event is expected and what it should contain

### Usage Example

```typescript
import { it } from 'node:test';
import { equal as eq } from 'node:assert/strict';

it("emits user-created event", (done) => {
  (async () => {
    const userService = new UserService();

    userService.once('user-created', (user) => {
      eq(user.name, 'Alice');
      eq(user.role, 'admin');
      done();
    });

    await userService.createUser({ name: 'Alice', role: 'admin' });
  })();
});
```

This pattern prevents listener leaks and handles async setup cleanly.

## Suppressing Expected Error Logs

### Overview

When testing unhappy paths—invalid inputs, authorisation failures, missing resources—your code often logs errors to help debug production issues. But in tests, these expected errors pollute console output, confusing developers who can't distinguish between real failures and intentional test behaviour. This becomes especially problematic when joining a new codebase: running tests and seeing error logs destroys confidence. You can't know if those errors indicate genuine bugs or are just testing error paths. The solution is to suppress expected error logs during tests whilst preserving unexpected ones.

### Key Benefits

1. **Clear Test Output**: Console shows only genuine failures, not expected error paths
2. **Developer Confidence**: New team members can trust a green test run
3. **Signal vs Noise**: Real bugs stand out instead of being buried in expected errors

### Implementation Approach

Most structured logging libraries provide built-in mechanisms to control log output dynamically without modifying production code. Rather than adding conditional statements to your logger or creating mocks and stubs, leverage your logger's native capabilities to adjust log levels during tests.

For example, with **pino** you can temporarily silence logging by changing the log level:

```typescript
logger.level('silent');
```

With **winston**, you can similarly adjust the log level:

```typescript
logger.level = 'silent';
```

Other common loggers like **bunyan**, **log4js**, and **consola** offer equivalent functionality. The key principle is to use your logger's existing API to control output rather than wrapping it in test-specific conditionals or replacing it with test doubles.

This approach keeps your production code clean and free from test-specific logic. You simply configure the logger instance in your test setup to suppress expected errors, then restore the original level afterwards.

### Guidelines

- **Use logger's built-in level control**: Prefer `logger.level('silent')` over mocking or conditional logic
- **Suppress temporarily**: Set silent level before testing error paths, restore afterwards
- **Restore reliably**: Use try-finally or test framework hooks to guarantee restoration
- **Avoid production code changes**: Don't add `if (NODE_ENV === 'test')` conditionals to your logger
- **Test the error**: Don't just suppress and ignore—assert the error occurred

## Fast Tests and Low Timeouts

### Overview

Fast tests are essential for maintaining developer cadence. When tests run in seconds, developers run them constantly, catching bugs immediately. Slow tests break this feedback loop—developers run them less frequently, wait longer for results, and lose focus during the delay. Tests that run quickly after every small change enable the tight feedback cycle that makes TDD effective.

### The Principle

Tests should complete in seconds, not minutes. A healthy test suite runs hundreds of tests in under a minute. Individual tests normally complete in milliseconds to a few seconds. If a test approaches 10 seconds, something is fundamentally wrong—the test is likely waiting for something that never occurs rather than exercising real system behaviour.

### Default Timeout Configuration

Set a low default timeout (e.g., 3 seconds) to catch hanging tests immediately. This forces you to write fast, focused tests and quickly surfaces tests that are blocked or waiting indefinitely.

Configure globally via CLI:

```bash
node --test --test-timeout=3000
```

Or in package.json:

```json
{
  "scripts": {
    "test": "node --test --test-timeout=3000"
  }
}
```

You can override the timeout for individual tests when needed:

```typescript
import { it } from 'node:test';

it("long-running integration test", { timeout: 10000 }, async () => {
  // This specific test gets 10 seconds
});
```

### When Tests Time Out

If a test times out:

1. **Look at what it's doing**: Is it waiting for something that never happens?
2. **Identify blocking operations**: Database queries, HTTP requests, event listeners, message queues
3. **Speed it up reliably**: Fix the root cause without making the test flaky

### Common Timeout Causes

#### Waiting for Events That Never Fire

```typescript
// BAD: Test hangs if event never fires
it("handles user created event", (done) => {
  emitter.once('user-created', (user) => {
    eq(user.name, 'Alice');
    done();
  });
  // Bug: forgot to actually trigger the event
});
```

**Fix:** Ensure the code that triggers the event is actually called, or check your event name matches.

#### Polling for Database Changes

```typescript
// BAD: Test polls database repeatedly
it("processes background job", async () => {
  await jobQueue.enqueue(job);

  // Polls every 100ms, waiting for job to complete
  while (!(await job.isComplete())) {
    await sleep(100);
  }

  eq(await job.status(), 'complete');
});
```

**Fix:** Use a callback, promise, or event to signal completion rather than polling.

#### Waiting for Locks That Never Release

```typescript
// BAD: Test waits for lock that's held by another test
it("acquires exclusive lock", async () => {
  await lockService.acquire('resource-123');
  // Test times out if lock is still held from previous test
});
```

**Fix:** Ensure locks are released in `afterEach()` or use test isolation.

#### Unresolved Promises

```typescript
// BAD: Promise never resolves
it("calls external service", async () => {
  // Service is down but no timeout configured
  await externalService.call();
});
```

**Fix:** Configure timeouts on external calls or use stubs (nock) to avoid real network calls.

### Speeding Up Tests Without Compromising Behaviour

#### Use Test Doubles

Replace slow external dependencies with fast in-memory alternatives:

```typescript
// Instead of real HTTP calls
nock('https://api.example.com')
  .get('/users/123')
  .reply(200, { id: 123, name: 'Alice' });

// Instead of real databases
const testDatabase = new InMemoryDatabase();

// Instead of real message queues
const testQueue = new InMemoryQueue();
```

#### Use Docker Compose with tmpfs for Test Databases

When you need real databases, use containers with in-memory storage:

```yaml
# docker-compose.yml
services:
  postgres-test:
    image: postgres:16-alpine
    tmpfs:
      - /var/lib/postgresql/data  # In-memory, much faster
```

#### Reduce Unnecessary Setup

```typescript
// BAD: Creates elaborate fixture for simple test
beforeEach(async () => {
  await createCompany();
  await createUsers(100);
  await seedProducts(500);
});

it("validates email format", () => {
  eq(isValidEmail('test@example.com'), true);
});

// GOOD: Only create what you need
it("validates email format", () => {
  eq(isValidEmail('test@example.com'), true);
});
```

### Warning Signs

Tests that take more than 10 seconds are almost always waiting for something that never occurs:

- **Event listeners**: Listening for events that are never emitted
- **Message queues**: Waiting for messages that never arrive
- **Database changes**: Polling for updates that never happen
- **Network timeouts**: Waiting for responses from dead services
- **Locks or semaphores**: Waiting for resources that are never released

When you see a 10+ second test timeout, investigate immediately—it's revealing a bug in the test or the code under test.

### Guidelines

- **Set low default timeouts**: 3 seconds catches most problems quickly
- **Investigate timeouts immediately**: They reveal bugs or poor test design
- **Use test doubles**: Stub external dependencies to keep tests fast
- **Fix root causes**: Don't just increase timeouts to mask problems
- **Run tests constantly**: Fast tests enable tight feedback loops
- **Tests taking > 10 seconds are wrong**: They're waiting for something that never happens
