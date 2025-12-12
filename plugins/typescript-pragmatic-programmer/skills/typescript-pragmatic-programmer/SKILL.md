---
name: typescript-pragmatic-programmer
description: Pragmatic programming principles and wisdom applied to TypeScript development. Use this skill when writing TypeScript code to follow pragmatic practices for maintainable, robust software.
allowed-tools: Read
---

# TypeScript Pragmatic Programmer

Practical wisdom for TypeScript developers, focusing on pragmatic approaches to building maintainable, robust software.

## Take Small Steps with Immediate Feedback

### Overview

Make small, incremental changes and verify them immediately. After every change, run TypeScript compilation and tests before moving forward. This tight feedback loop catches errors early when they're easiest to fix and prevents you from building on faulty foundations.

### The Principle

Large changes introduce many variables, making it hard to identify what went wrong when something breaks. Small changes with immediate verification create a safety net that catches problems at their source. The cost of fixing a bug grows exponentially with the time between introduction and detection.

### Practice in TypeScript

After every meaningful change, no matter how small:

1. **Compile**: Run `tsc` or your build tool to catch type errors
2. **Test**: Run your test suite to verify behavior
3. **Commit**: If both pass, commit immediately

```bash
# After each change
npm run build    # Verify TypeScript compilation
npm test         # Verify tests pass
git add .
git commit -m "Small, verified change"
```

### What Counts as a Small Change?

- Add a single function
- Rename a variable or function
- Extract a helper function
- Add a type annotation
- Refactor one method
- Add one test case

### Why This Works

- **Immediate feedback**: Type errors appear seconds after introduction
- **Easy debugging**: Only one change to investigate if something breaks
- **Confidence**: Each step is verified before the next
- **Clean history**: Commits represent working states, making bisect effective
- **Less stress**: No "big bang" integration of many unverified changes

### Guidelines

- **Compile after every change**: TypeScript is fast, use it constantly
- **Run affected tests**: Use watch mode or run specific test files
- **Commit working states**: Only commit when compilation and tests pass
- **Use watch mode**: `tsc --watch` and `npm test -- --watch` for instant feedback
- **Fix before moving on**: Don't pile up errors to "fix later"

## Tracer Bullets: Build End-to-End First

### Overview

Build a thin slice through all layers of your system first, then expand it. This ensures all parts integrate correctly before investing in features. A "tracer bullet" hits the target, showing you're aimed correctly.

### Practice in TypeScript

When building a new feature:

1. **Minimal end-to-end**: Request → route → service → database → response
2. **Verify integration**: All layers talk to each other
3. **Add functionality**: Expand each layer incrementally

```typescript
// Step 1: Minimal end-to-end (tracer bullet)
app.post('/api/users', async (c) => {
  const user = await db.users.create({ name: 'test' });
  return c.json(user);
});

// Step 2: Add validation
app.post('/api/users', async (c) => {
  const data = CreateUserSchema.parse(await c.req.json());
  const user = await db.users.create(data);
  return c.json(user);
});

// Step 3: Add service layer
app.post('/api/users', async (c) => {
  const data = CreateUserSchema.parse(await c.req.json());
  const user = await userService.createUser(data);
  return c.json(user);
});
```

### Guidelines

- **Thin slice first**: Get one simple case working end-to-end
- **Expand incrementally**: Add complexity layer by layer
- **Test integration early**: Don't build components in isolation then hope they integrate

## Use High-Quality Libraries: Don't Reinvent the Wheel

### Overview

Before implementing functionality yourself, check if the user has expressed a preference for a suitable library. If not, search for well-maintained, high-quality libraries on npm. Reusing proven solutions saves time, reduces bugs, and gives you battle-tested code that handles edge cases you might miss.

### The Principle

Writing custom implementations is expensive: you pay for development time, testing time, bug fixes, maintenance, and opportunity cost. Good libraries have already paid these costs and spread them across thousands of users. Your time is better spent on domain-specific problems that differentiate your application.

### Process

When you need common functionality:

1. **Use existing dependencies**: Is there already a library in package.json that can handle this?
2. **Check user preferences**: Has the user specified a library preference in their configuration, documentation, skills or previous prompts?
3. **Search npm**: Look for high-quality libraries if no preference exists
4. **Evaluate quality**: Check downloads, maintenance, TypeScript support, bundle size
5. **Install and use**: Add the dependency rather than implementing from scratch

### When to Write Your Own

Write custom implementations only when:

- **Domain-specific logic**: The functionality is unique to your business domain
- **No suitable library exists**: You've searched and found nothing appropriate
- **Library is abandoned**: Last update was years ago with open critical issues
- **Extreme simplicity**: The implementation is truly 2-3 lines (e.g., `Math.max(a, b)`)
- **Performance critical**: Profiling shows the library is a bottleneck (rare)

### How to Evaluate Libraries

When recommending libraries, consider:

- **Maintenance**: Number of long-standing / serious issues. Recent commits and releases
- **TypeScript support**: Native types or @types package
- **Downloads**: npm weekly downloads (higher suggests stability)
- **Size**: Bundle size impact (use bundlephobia.com)
- **Dependencies**: Fewer transitive dependencies reduces risk
- **Documentation**: Clear examples and API docs
- **Tests**: Well-tested with high coverage

### Guidelines

- **Check user preferences FIRST**: Look in package.json, project docs, or ask
- **Search before coding**: Spend 5 minutes searching npm before implementing
- **Prefer well-maintained**: Choose libraries with recent activity
- **TypeScript-first**: Prefer libraries with native TypeScript support
- **Bundle size matters**: Check impact on frontend applications
- **Read the docs**: Understand the API before dismissing as "too complex"
- **Domain code only**: Save your effort for problems unique to your application

## Fix Root Problems: Never Compromise on Quality

### Overview

When you encounter a problem while implementing the user's design, fix the root issue rather than abandoning the approach or disabling safeguards. If the user has specified a design pattern or architecture, honor it even when it creates challenges. Taking shortcuts—like disabling TypeScript, turning off tests, or ignoring lint rules—creates technical debt and undermines the quality foundation.

### The Principle

Quality tools like TypeScript, tests, and linters exist to catch problems. When they complain, they're doing their job. Disabling them is shooting the messenger. Similarly, if the user has chosen an architectural approach (events, dependency injection, etc.), they did so for good reasons. Abandoning it at the first sign of difficulty shows lack of discipline and understanding.

### When Something "Just Won't Work"

**Wrong Response:**
- "TypeScript won't let me do this, so I'll use `any`"
- "The tests keep failing, so I'll skip them for now"
- "The linter is annoying, so I'll turn it off"
- "This pattern is too hard, so I'll use a simpler approach"

**Right Response:**
- "TypeScript won't let me do this, so I'll fix my type definitions"
- "The tests keep failing, so I'll fix the bug they're catching"
- "The linter is complaining, so I'll write better code"
- "This pattern is challenging, so I'll learn how to implement it correctly"

### Guidelines

- **Honor the user's design choices**: They specified patterns for good reasons
- **Fix TypeScript errors properly**: Add types, don't use `any` or `@ts-ignore`
- **Fix failing tests**: Don't skip or disable them
- **Respect linter rules**: Write compliant code, don't disable checks
- **Solve root causes**: Don't work around problems with hacks
- **Ask for clarification**: If the design seems wrong, ask the user to explain
- **Learn the pattern**: If struggling with user's approach, understand it better
- **Quality is non-negotiable**: Tests, types, and linting are safeguards, not obstacles

## Docker Compose for Local Backing Services

### Overview

Use docker-compose to manage local backing services like databases, Redis, message queues, etc. This ensures consistent environments across developers and CI, eliminates "works on my machine" problems, and provides isolation between development and test databases.

### Key Benefits

1. **Consistent Environments**: All developers and CI have identical backing service versions
2. **Fast Setup**: New developers run `docker-compose up` instead of installing databases
3. **Isolation**: Development and test databases are completely separated
4. **Version Control**: Infrastructure configuration lives alongside code
5. **Easy Cleanup**: `docker-compose down -v` removes all data for fresh starts

### Implementation

Run separate database containers for development and tests. This provides complete isolation and allows both environments to run simultaneously.

```yaml
# docker-compose.yml
services:
  postgres-dev:
    image: postgres:16-alpine
    container_name: my-app-postgres-dev
    environment:
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"
    volumes:
      - postgres-dev-data:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

  postgres-test:
    image: postgres:16-alpine
    container_name: my-app-postgres-test
    environment:
      POSTGRES_PASSWORD: postgres
    ports:
      - "5433:5432"  # Different host port
    tmpfs:
      - /var/lib/postgresql/data  # In-memory for faster tests
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5

volumes:
  postgres-dev-data:
```

**Note**: Since each container is dedicated to a single application, it's fine to use the default `postgres` user and `postgres` database name. There's no need to create custom database names when you have complete container isolation.

### npm Scripts Integration

```json
{
  "scripts": {
    "docker:up": "docker-compose up -d",
    "docker:down": "docker-compose down",
    "docker:reset": "docker-compose down -v && docker-compose up -d",
    "docker:logs": "docker-compose logs -f"
  }
}
```

### Important Guidelines

- **Use specific image versions**: Never use `latest` tag (e.g., `postgres:16-alpine`, not `postgres:latest`)
- **Use tmpfs for test databases**: Speeds up tests significantly by keeping data in memory
- **Include healthchecks**: Ensures containers are truly ready before app starts
- **Map to different ports**: Development on 5432, test on 5433
- **Version control docker-compose.yml**: Infrastructure as code
- **Use volumes for development data**: Persists data across container restarts
- **Container names**: Prefix with project name to avoid conflicts with other projects
- **Complete isolation**: Dev and test can run simultaneously without interference
