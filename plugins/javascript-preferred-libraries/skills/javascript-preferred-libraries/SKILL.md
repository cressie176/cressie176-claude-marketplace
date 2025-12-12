---
name: javascript-preferred-libraries
description: Use this skill when generating JavaScript/TypeScript code, suggesting libraries or dependencies, or making architectural decisions about which packages to use. This skill provides guidance on preferred libraries and frameworks to use, acceptable alternatives, and libraries to avoid with specific reasons.
allowed-tools: Read, Grep, Glob
---

# JavaScript/TypeScript Preferred Libraries

This skill provides guidance on library selection for JavaScript and TypeScript projects. Use this when:
- Generating new code that requires external dependencies
- Suggesting libraries to solve a problem
- Making architectural decisions about technology choices
- Reviewing existing dependencies

## Library Preferences by Category

### HTTP Server

**Prefer:**
- **Hono** - Good TypeScript support, builds on Express patterns

**Acceptable:**
- **Express** - Established, widely used

**Avoid:**
- **Fastify** - TypeScript ends up being nasty

### ORM / Database Clients

**Prefer:**
- **Drizzle** - TypeScript-first, wide adoption

**Acceptable:**
- **Slonik** - Not an ORM, but acceptable for direct SQL queries

**Avoid:**
- **Prisma** - Long history of poor decision making
- **Sequelize** - Mature, but not TypeScript friendly
- **Raw PostgreSQL clients** - Only use for extremely trivial applications

### Testing Frameworks

**Prefer:**
- **Node test framework** (`node:test`) - Native, fast, no dependencies
- **Node assert** (`node:assert`) - Native assertion library

**Avoid:**
- **Jest** - Very slow to start, rewrites your code messing with the import order, lots of bugs
- **Mocha** - High false-positive CVE

### Date/Time

**Prefer:**
- **Luxon** - Written by a moment maintainer who understood the warts, immutable API

**Acceptable:**
- **date-fns** - OK alternative, but Luxon is preferred

**Avoid:**
- **moment** - Not immutable, surprising API

### HTTP Client

**Prefer:**
- **fetch** - Native, no dependencies
- **axios** - Widely used, acceptable alternative

### Validation

**Prefer:**
- **zod** - Best TypeScript integration, type inference

**Acceptable:**
- **yup** - OK alternative
- **ajv** - OK if JSON Schema validation specifically required

**Avoid:**
- **joi** - Draws in too many dependencies

### Linting/Formatting

**Prefer:**
- **Biome** - Fast, batteries included

**Avoid:**
- **ESLint** - Dependencies have false positives, nightmare to update

### Git Hooks

**Prefer:**
- **lefthook** - Simple, fast

**Avoid:**
- **husky** - Clunky setup

### Logging

**Prefer:**
- **logtape** - Simple, predictable

**Avoid:**
- **Pino** - Over complicated, workers result in unpredictable behaviour, non-standard API
- **winston** - Full of bugs

## Usage Guidelines

1. **Always prefer native Node.js APIs** when available (fetch, node:test, node:assert, etc.)
2. **Choose TypeScript-friendly libraries** - Libraries with good type definitions and inference
3. **Avoid unnecessary dependencies** - Prefer simpler, more focused libraries
4. **Consider bundle size** - Especially for libraries that might be included in client bundles
5. **Check for active maintenance** - Prefer libraries with active development and community support

When suggesting a library that's not in the preferred list, explain why it's being recommended and acknowledge any trade-offs.
