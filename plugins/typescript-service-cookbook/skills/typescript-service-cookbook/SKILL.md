---
name: typescript-service-cookbook
description: TypeScript service design patterns and implementation cookbook. Use this skill when creating or refactoring TypeScript services to reference proven patterns for well-factored, maintainable service architectures.
allowed-tools: Read
---

# TypeScript Service Cookbook

## Separate Construction and Lifecycle Management

### Overview

Applications should separate construction from lifecycle management (starting/stopping). The entry point constructs all dependencies and injects them into an Application class, then calls `start()` as a separate step. This separation allows integration tests to construct the application with test configurations and fully control its lifecycle. Components should implement asynchronous `start()` methods for initialization and optional `stop()` methods for graceful shutdown.

### Key Benefits

1. **Full Test Control**: Integration tests can start/stop the application programmatically
2. **Explicit Dependencies**: Major dependencies (config, database, server) are injected via constructor
3. **No Hidden State**: Constructor does no I/O, only stores dependencies
4. **Clean Lifecycle**: Start and stop operations are explicit and reversible
5. **Graceful Shutdown**: Resources are properly closed and cleaned up

### Implementation

```typescript
// src/Database.ts
export default class Database {
  constructor(private readonly config: DatabaseConfig) {}

  async start(): Promise<void> {
    // Connect to database, run migrations, etc.
  }

  async stop(): Promise<void> {
    // Close database connections
  }
}

// src/WebServer.ts
export default class WebServer {
  constructor(private readonly config: ServerConfig) {}

  async start(): Promise<void> {
    // Start HTTP server
  }

  async stop(): Promise<void> {
    // Stop accepting new connections, finish in-flight requests
  }
}

// src/Application.ts
export default class Application {
  constructor(
    private readonly database: Database,
    private readonly server: WebServer
  ) {}

  async start(): Promise<void> {
    await this.database.start();
    await this.server.start();
  }

  async stop(): Promise<void> {
    await this.server.stop();
    await this.database.stop();
  }
}

// index.ts (production)
const config = loadConfig();
const database = new Database(config.postgres);
const server = new WebServer(config.server);
const application = new Application(database, server);

process.on("SIGINT", () => application.stop());
process.on("SIGTERM", () => application.stop());

await application.start();
```

**Important Guidelines:**

- Inject all major dependencies (config, database, server, etc.) via constructor
- Constructor should only assign dependencies, never perform I/O or async operations
- All I/O and async initialization happens in `start()`
- Components orchestrate startup in dependency order (database before server)
- Components orchestrate shutdown in reverse order (server before database)
- Stop methods are optional but recommended for graceful shutdown
- **Prefer manual construction** as shown above - it's explicit, simple, and maintainable for most applications
- For very large applications with many dependencies, consider a minimal DI framework like [systemic](https://github.com/guidesmiths/systemic), but avoid heavyweight frameworks like NestJS or magical decorator-based frameworks like tsyringe

## Hierarchical Configuration Management

### Overview

Configuration should be version-controlled with the code that depends on it (except secrets). While this contradicts 12-factor app wisdom, most applications have a small, known set of environments, so naming them explicitly is pragmatic. Environment variables are problematic: they only support strings and cannot represent structured data like arrays or objects. A hierarchical configuration approach uses a base `default.json` file that is progressively merged with environment-specific overrides.

### Key Benefits

1. **Version Control**: Configuration lives with code, making changes traceable and reviewable
2. **Rich Data Types**: JSON supports objects, arrays, numbers, booleans, not just strings
3. **Sensible Defaults**: Most configuration is in default.json, overrides are minimal and explicit
4. **Type Safety**: Configuration can be validated and typed

### Implementation

```typescript
// src/Configuration.ts
import { readFileSync as read, existsSync as exists } from 'node:fs';
import { mergeDeepRight as merge } from 'ramda';

export default class Configuration {
  static load(filenames: string[]): Record<string, any> {
    return Object.freeze(
      filenames
        .filter((filename) => exists(filename))
        .map((filename) => JSON.parse(read(filename, 'utf8')))
        .reduce((config, overrides) => merge(config, overrides), {})
    );
  }
}

// src/init/load-config.ts
import Configuration from "../Configuration.js";
import { hostname } from 'node:os';

export default function loadConfig() {
  return Configuration.load([
    "config/default.json",
    `config/${process.env.APP_ENV || "local"}.json`,
    `config/${hostname()}.json`,
    "config/runtime.json",
    process.env.APP_CONFIG,
  ].filter(Boolean));
}
```

### Configuration Hierarchy

Configuration files are loaded and merged in this order:

1. **`config/default.json`** - Base configuration with sensible defaults
2. **`config/${APP_ENV}.json`** - Environment-specific overrides (local, test, development, staging, production)
3. **`config/${hostname}.json`** - Machine-specific overrides (optional)
4. **`config/runtime.json`** - Runtime overrides (optional, typically gitignored)
5. **`${APP_CONFIG}`** - Path from environment variable for secrets (optional)

**Important Guidelines:**

- **Never use `NODE_ENV`**: Third-party libraries use it for their own purposes (e.g., Handlebars only caches templates when `NODE_ENV=production`), which causes surprising behaviour when set to "staging" or "test"
- **Use `APP_ENV` instead**: This is application-specific and won't conflict with libraries
- **Version control everything except secrets**: Config files (except runtime.json and secrets) should be committed
- **Deep merge**: Later config files merge deeply, not shallow replace, so you only specify what changes
- **Freeze configuration**: Use `Object.freeze()` to prevent accidental mutation
- **Named environments**: It's fine to name environments explicitly (local, test, staging, production) rather than trying to be environment-agnostic

## Best Practice Factory Modules

### Overview

Wrap third-party libraries in custom factory modules that enforce organizational standards and best practices. This creates a "pit of success" where correct usage becomes the default. As Jeff Bezos said: "Good intentions don't work, good mechanisms do." Rather than documenting best practices and hoping teams follow them, factory modules make best practices automatic and impossible to bypass.

### The Problem

In microservice architectures, standardizing on a single library isn't sufficient. When issues arise—such as unhandled errors, sensitive data leaks, or missed configuration—fixes must be manually applied to potentially hundreds of services, creating unsustainable overhead. Factory modules solve this by centralizing best practices in one place that all services consume.

### Key Benefits

1. **Automatic Best Practices**: Teams get correct behaviour by default without needing to remember or implement standards
2. **Centralized Updates**: Fix issues once in the factory, all consuming services benefit immediately
3. **Organizational Standards**: Enforce conventions consistently across all services
4. **Reduced Cognitive Load**: Developers don't need to learn every configuration option of the underlying library

### Examples

#### Database Connection Factory (PostgreSQL)

```typescript
// packages/database/src/index.ts
import pg from 'pg';

export function createDatabase(config: DatabaseConfig) {
  const pool = new pg.Pool(config);

  // Best practice: Always listen for errors to prevent unhandled exceptions
  pool.on('error', (err) => {
    console.error('Unexpected database error', err);
  });

  return pool;
}
```

**What this prevents**: Unhandled 'error' events that crash the application when database connections fail unexpectedly.

#### Redis Client Factory

```typescript
// packages/redis/src/index.ts
import { createClient } from 'redis';

export async function createRedis(config: RedisConfig) {
  const client = createClient(config);

  // Best practice: Always listen for errors
  client.on('error', (err) => {
    console.error('Redis error', err);
  });

  // Best practice: Wait for connection to be truly ready before returning
  await new Promise<void>((resolve) => {
    client.connect();
    client.once('ready', () => resolve());
  });

  return client;
}
```

**What this prevents**: Unhandled Redis errors that crash the application, forgetting to call `connect()`, and race conditions where the application starts before Redis is truly ready.

#### Logger Factory

```typescript
// packages/logger/src/index.ts
import pino from 'pino';

export function createLogger(options: LoggerOptions = {}) {
  return pino({
    level: options.level || 'info',

    // Best practice: Always redact sensitive fields
    redact: {
      paths: ['password', 'authorization', 'cookie'],
      censor: '[REDACTED]'
    },

    // Best practice: ISO timestamps
    timestamp: pino.stdTimeFunctions.isoTime,
  });
}
```

**What this prevents**: Sensitive data appearing in logs, inconsistent timestamp formats.

### Important Guidelines

- **Version as internal packages**: Publish factory modules to your internal npm registry or use monorepo packages
- **Minimize options**: Only expose configuration that genuinely needs to vary
- **Enforce best practices automatically**: Don't make correct usage optional

## Unit of Work Pattern with AsyncLocalStorage

### Overview

The Unit of Work pattern groups database operations into a single transaction, ensuring all operations succeed or fail together. Using Node's AsyncLocalStorage allows services to join an active transaction without explicitly passing transaction objects through every function call. This provides transactional guarantees while keeping service code clean and focused.

### Key Benefits

1. **Automatic Transaction Management**: Services automatically use the active transaction without manual plumbing
2. **Clear Boundaries**: Transaction spans are explicitly named and visible in logs
3. **Atomic Operations**: Multiple service calls within a span either all succeed or all fail
4. **Clean Service Code**: Services don't need transaction parameters in every method

### Implementation

```typescript
// src/Database.ts
import { AsyncLocalStorage } from 'async_hooks';
import type { PgTransaction } from 'drizzle-orm/pg-core';

export default class Database {
  private transactionStorage = new AsyncLocalStorage<PgTransaction<any, any, any>>();

  getDatabase() {
    const activeTransaction = this.transactionStorage.getStore();
    return activeTransaction || this.db;
  }

  async startTransaction<T>(callback: () => Promise<T>): Promise<T> {
    return this.db.transaction(async (newTransaction: PgTransaction<any, any, any>) => {
      return this.transactionStorage.run(newTransaction, callback);
    });
  }

  async joinTransaction<T>(callback: (transaction: PgTransaction<any, any, any>) => Promise<T>): Promise<T> {
    const existingTransaction = this.transactionStorage.getStore();

    if (!existingTransaction) {
      throw new Error('No active transaction');
    }

    return callback(existingTransaction);
  }
}

// src/UnitOfWork.ts
export default class UnitOfWork {
  constructor(private database: Database) {}

  async span<T>(name: string, fn: () => Promise<T>): Promise<T> {
    return logger.withContext({ unitOfWork: name }, async () => {
      logger.debug(`Starting unit of work: ${name}`);
      const result = await this.database.startTransaction(async () => fn());
      logger.debug(`Completed unit of work: ${name}`);
      return result;
    });
  }
}
```

### Usage in Routes

```typescript
// src/routes/organisation.ts
export default function createOrganisationRoutes(
  unitOfWork: UnitOfWork,
  organisationService: OrganisationService
) {
  async function createOrganisation(c: Context) {
    const { name } = await c.req.json();

    const organisation = await unitOfWork.span("Create Organisation", async () => {
      return await organisationService.createOrganisation({ name }, userId);
    });

    return c.json(organisation, 201);
  }
}
```

### Usage in Services

```typescript
// src/services/OrganisationService.ts
export default class OrganisationService {
  constructor(private database: Database) {}

  async createOrganisation({ name }: CreateOrganisationRequest, userId: number) {
    // Multiple database operations in the same transaction
    const organisation = await this._insertOrganisation(name);
    await this._assignAdmin(userId, organisation.id);
    return organisation;
  }

  private async _insertOrganisation(name: string) {
    return this.database.joinTransaction(async (transaction) => {
      const [organisation] = await transaction
        .insert(organisationSchema)
        .values({ name })
        .returning();
      return organisation;
    });
  }

  private async _assignAdmin(userId: number, organisationId: number) {
    return this.database.joinTransaction(async (transaction) => {
      await transaction
        .insert(organisationMemberSchema)
        .values({ userId, organisationId, isAdmin: true });
    });
  }
}
```

**Important Guidelines:**

- **Start transactions at boundaries**: Use `unitOfWork.span()` in route handlers, not in services
- **Join transactions in services**: Services use `joinTransaction()` to participate in the active transaction
- **Name your spans**: Use descriptive names for unit of work spans to aid debugging
- **Throw on missing transaction**: `joinTransaction()` should throw if no transaction is active
- **One transaction per request**: Typically each HTTP request has one unit of work span

## AsyncLocalStorage for Logging Context

### Overview

Use AsyncLocalStorage to automatically propagate context (like requestId, userId, etc.) through all async operations without manually passing it as parameters. This creates ambient context that flows through the entire request lifecycle, enriching all log statements automatically.

### Key Benefits

1. **Automatic Context Propagation**: Context flows through all async operations without explicit passing
2. **Cleaner Function Signatures**: No need for logger or context parameters everywhere
3. **Consistent Logging**: All logs within a context automatically include relevant metadata
4. **Framework Agnostic**: Works with any async code, not tied to specific frameworks

### Implementation

```typescript
// src/Logger.ts
import { AsyncLocalStorage } from "node:async_hooks";

const asyncLocalStorage = new AsyncLocalStorage<Record<string, any>>();

export const instance = {
  debug: (message: string, context?: any): void => {
    log(LogLevel.DEBUG, message, context);
  },
  info: (message: string, context?: any): void => {
    log(LogLevel.INFO, message, context);
  },
  error: (message: string, context?: any): void => {
    log(LogLevel.ERROR, message, context);
  },
  withContext: <T>(context: Record<string, any>, fn: () => T): T => {
    return asyncLocalStorage.run(context, fn);
  },
};

function log(level: LogLevel, message: string, context: any = {}): void {
  const store = asyncLocalStorage.getStore() || {};
  // Merge ambient context with explicitly provided context
  emit(ApplicationEvents.LOG, { level, message, context: { ...store, ...context } });
}
```

### Usage

```typescript
// In middleware - set context for entire request
export function requestLogger() {
  return async (c: Context, next: Next) => {
    const requestId = randomUUID();
    const requestContext = { requestId, method: c.req.method, path: c.req.path };

    return logger.withContext(requestContext, async () => {
      logger.info(`HTTP ${c.req.method} ${c.req.path} started`);
      await next();
      logger.info(`HTTP ${c.req.method} ${c.req.path} completed`);
    });
  };
}

// In services - logs automatically include requestId
export class UserService {
  async createUser(data: CreateUserInput) {
    logger.info('Creating user', { email: data.email });
    // Log output includes: { requestId: '...', email: '...' }
  }
}
```

**Important Guidelines:**

- **Set context at boundaries**: Use `withContext()` in middleware, not deep in business logic
- **Merge contexts**: Explicitly provided context should override ambient context
- **Nest contexts**: Contexts can be nested (e.g., request context, then unit of work context)
- **Keep context small**: Only include data useful for debugging and correlation

## Request Logging Middleware

### Overview

Automatically log all HTTP requests and responses with metadata like duration, status code, and request ID. This middleware wraps each request in a logging context and captures start/completion events without manual instrumentation.

### Key Benefits

1. **Automatic Request Tracking**: Every request is logged without manual effort
2. **Duration Tracking**: Automatic timing of request processing
3. **Correlation**: Request ID propagates through all logs for the request
4. **Consistent Format**: Standardized request/response logging across the application

### Implementation

```typescript
// src/middleware/RequestLogger.ts
import { Context, Next } from "hono";
import { randomUUID } from "node:crypto";

export function requestLogger() {
  return async (c: Context, next: Next) => {
    const requestId = randomUUID();
    const startTime = Date.now();
    const method = c.req.method;
    const path = c.req.path;

    const requestContext = { requestId, method, path };

    return logger.withContext(requestContext, async () => {
      logger.info(`HTTP ${method} ${path} started`, { method, path });

      await next();

      const duration = Date.now() - startTime;
      const status = c.res.status;

      logger.info(`HTTP ${method} ${path} ${status}`, { method, path, status, duration });
    });
  };
}
```

### Usage

```typescript
// src/routes/index.ts
export default function createRoutes(database: Database) {
  const app = new Hono();

  // Apply to all /api routes
  app.use("/api/*", requestLogger());

  app.route("/api/users", userRoutes);
  app.route("/api/organisations", organisationRoutes);

  return app;
}
```

**Important Guidelines:**

- **Place early in middleware chain**: Ensures all requests are logged, even if they fail in other middleware
- **Generate request ID**: Use UUID or similar for unique correlation across services
- **Log both start and completion**: Helps identify requests that never complete
- **Include duration**: Essential for performance monitoring and alerting

## Custom Error Classes with Centralized Handling

### Overview

Define domain-specific error classes that carry structured details, then handle them centrally with a single error handler. This separates error creation (throw anywhere) from error presentation (handle once), making error handling consistent across the application.

### Key Benefits

1. **Consistent Error Responses**: All errors of the same type produce identical response format
2. **Type-Safe Error Details**: TypeScript ensures error details are correctly structured
3. **Single Source of Truth**: One place to change error handling for entire application
4. **Clean Service Code**: Services throw errors, don't concern themselves with HTTP responses

### Implementation

```typescript
// src/errors/ValidationError.ts
export default class ValidationError extends Error {
  public code: string;

  constructor(public details: any) {
    super("Validation failed");
    this.name = "ValidationError";
    this.code = "TP_VALIDATION_ERROR";
  }
}

// src/errors/BadRequestError.ts
export default class BadRequestError extends Error {
  constructor(message: string) {
    super(message);
    this.name = 'BadRequestError';
  }
}

// src/routes/index.ts
export default function createRoutes(database: Database) {
  const app = new Hono();

  // Centralized error handler
  app.onError((error, c) => {
    if (error instanceof ValidationError) {
      return c.json({
        error: "Validation failed",
        details: error.details
      }, 400);
    }

    if (error instanceof BadRequestError) {
      return c.json({ error: error.message }, 400);
    }

    logger.error("Unhandled error", error);
    return c.json({ error: "Internal server error" }, 500);
  });

  return app;
}
```

### Usage

```typescript
// Services throw domain errors
export class UserService {
  async createUser(data: unknown) {
    const result = CreateUserSchema.safeParse(data);
    if (!result.success) {
      throw new ValidationError(result.error.issues);
    }

    if (await this.userExists(result.data.email)) {
      throw new BadRequestError('User already exists');
    }

    return this.database.users.create(result.data);
  }
}
```

**Important Guidelines:**

- **Include error codes**: Makes it easier for clients to handle errors programmatically
- **Structure validation details**: Use consistent format (Zod's error format works well)
- **Log unexpected errors**: Always log errors you don't recognize
- **Don't expose internals**: Generic message for unhandled errors to avoid leaking implementation details

## Event-Driven Logging Bridge

### Overview

Decouple your application's logging calls from the underlying logging framework by emitting events that a logging bridge listens to. This allows you to swap logging frameworks (pino, winston, logtape) without changing application code, and makes it easy to test logging behavior.

### Key Benefits

1. **Framework Independence**: Application code doesn't depend on specific logging library
2. **Easy Testing**: Tests can listen for log events without configuring full logging framework
3. **Centralized Configuration**: Logging framework configured once at startup
4. **Context Merging**: Logging bridge can merge AsyncLocalStorage context before emission

### Implementation

```typescript
// src/Logger.ts (application-facing)
import { AsyncLocalStorage } from "node:async_hooks";
import { Events as ApplicationEvents } from "./Application.js";

const asyncLocalStorage = new AsyncLocalStorage<Record<string, any>>();

export const instance = {
  debug: (message: string, context?: any): void => {
    log(LogLevel.DEBUG, message, context);
  },
  info: (message: string, context?: any): void => {
    log(LogLevel.INFO, message, context);
  },
  error: (message: string, context?: any): void => {
    log(LogLevel.ERROR, message, context);
  },
  withContext: <T>(context: Record<string, any>, fn: () => T): T => {
    return asyncLocalStorage.run(context, fn);
  },
};

function log(level: LogLevel, message: string, context: any = {}): void {
  const store = asyncLocalStorage.getStore() || {};
  // Emit event with merged context
  process.emit(ApplicationEvents.LOG, {
    level,
    message,
    context: { ...store, ...context }
  });
}

// src/init/init-logging.ts (logging framework bridge)
import { getLogger, configure } from "@logtape/logtape";
import { Events as ApplicationEvents } from "../Application.js";

export default async function initLogging(config: any) {
  await configure({
    sinks: { console: getConsoleSink() },
    loggers: [{ category: ["app"], lowestLevel: config.level, sinks: ["console"] }],
  });

  const logger = getLogger(["app"]);
  addLogListeners(logger);
}

function addLogListeners(logger: any) {
  process.on(ApplicationEvents.LOG, (data: { level: LogLevel; message: string; context: any }) => {
    const method = data.level.toLowerCase();
    logger[method](data.message, data.context);
  });
}
```

### Usage

```typescript
// Application startup
const config = loadConfig();
await initLogging(config.logging);

// Application code
logger.info('User created', { userId: 123 });
// Event emitted, bridge forwards to LogTape

// In tests
const logEvents: any[] = [];
process.on(ApplicationEvents.LOG, (data) => logEvents.push(data));
// No need to configure full logging framework
```

**Important Guidelines:**

- **Emit after context merge**: Merge AsyncLocalStorage context before emitting event
- **Use typed events**: Define event types to ensure type safety
- **Initialize bridge at startup**: Set up event listeners before application starts processing
- **Consider event namespacing**: Use application-specific event names to avoid conflicts

