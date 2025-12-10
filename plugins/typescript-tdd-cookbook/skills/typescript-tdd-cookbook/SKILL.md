---
name: typescript-tdd-cookbook
description: TypeScript TDD patterns and implementation cookbook. Use this skill when generating TypeScript code to reference proven test-driven development patterns.
allowed-tools: Read
---

# TypeScript TDD Cookbook

Great tests tell a simple, focused story. You see instantly what behaviour is under test and which values actually matter. That means pulling the noise out of sight and keeping only the essential details in view. The patterns in this cookbook exist to support that aim: they strip away clutter, clarify intent and help your tests express the behaviour you actually care about rather than the mechanics required to exercise it.

## Pattern 1: ObjectMother / TestDataFactory Pattern

### Overview

Great tests tell a simple story about behaviour and intent. Test data is often where that story falls apart: sprawling, highly duplicated fixtures swamp what the test is really trying to say. Static, centralised fixtures help a little, but they soon become cumbersome and brittle as every new case needs yet more hard coded data. A better approach is to randomly generate realistic data, then explicitly override the few values you care about. The result is a cleaner test suite with far clearer intent.Static, centralised fixtures help a little, but they soon become cumbersome and brittle as every new case needs yet more hard coded data. A better approach is to randomly generate realistic data, then explicitly override the few values you care about. The result is a cleaner test suite with far clearer intent.

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

## Pattern 2: Test Client Pattern

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

The test client abstracts HTTP complexity while tests focus on behavior and assertions.

## Pattern 3: Controlling Time in Tests

### Overview

Testing time-based behaviour requires control over what "now" means. Without it you're left with fragile tests that depend on real system time or arbitrary sleeps. A better approach is to use a library that allows you to override the current time in tests.

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
