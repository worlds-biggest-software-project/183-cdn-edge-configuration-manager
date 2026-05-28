# CDN & Edge Configuration Manager — Phased Development Plan

> Project: 183-cdn-edge-configuration-manager · Created: 2026-05-25
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language | TypeScript (Node.js 22+) | API-heavy platform with extensive JSON/JSONB manipulation; strong OpenAPI tooling ecosystem; native async/await for concurrent CDN provider API calls; broad CDN provider SDK availability (Cloudflare, AWS, Akamai all publish TypeScript/JS SDKs) |
| API Framework | Fastify 5 | High-performance HTTP framework with built-in OpenAPI 3.1 schema generation via `@fastify/swagger`; TypeBox for runtime type validation matching TypeScript types; plugin architecture suits modular provider integration |
| Database | PostgreSQL 16 + ltree extension | Relational backbone for configuration integrity with CHECK constraints and foreign keys; ltree extension for hierarchical rule trees (mirrors Akamai's rule tree model); JSONB for provider-native config storage; TimescaleDB extension for time-series edge metrics |
| ORM / Query Builder | Drizzle ORM | Type-safe SQL with zero-overhead abstractions; native PostgreSQL JSONB operators; migration generation from schema definitions; works with ltree via custom column types |
| Task Queue | BullMQ (Redis-backed) | Provider sync operations, purge propagation, certificate renewal, and ML pipeline jobs are async workloads; BullMQ provides priority queues, retries, rate limiting, and job dependencies — essential for orchestrating multi-provider operations |
| Cache | Redis 7 | Session cache, rate limiting, provider API response caching, and real-time traffic weight storage for the ML routing model; also serves as BullMQ backend |
| Frontend | Next.js 15 (App Router) | Dashboard for configuration management, analytics visualization, and rule editing; React Server Components for fast initial loads; API routes colocated with UI; Tailwind CSS + shadcn/ui for consistent UI components |
| AI / ML | OpenAI API (GPT-4o) + custom scoring models | Natural-language configuration interface uses LLM; cache-rule recommender and anomaly detection use lightweight custom models trained on telemetry data; OpenAI SDK for TypeScript |
| Containerisation | Docker + Docker Compose | Self-hosted deployment target; multi-service architecture (API, worker, frontend, PostgreSQL, Redis); Dockerfile per service with multi-stage builds |
| Testing | Vitest + Supertest + Playwright | Vitest for unit/integration tests (fast, native TypeScript, ESM-compatible); Supertest for HTTP API testing; Playwright for E2E dashboard tests |
| Code Quality | ESLint 9 (flat config) + Prettier + typescript-eslint | Standard TypeScript linting; strict mode enabled; import sorting via eslint-plugin-import |
| Package Manager | pnpm 9 | Workspace support for monorepo; strict dependency resolution; disk-efficient |
| Monorepo | pnpm workspaces + Turborepo | Shared types between API, frontend, and worker packages; Turborepo for cached parallel builds |
| Secrets Management | HashiCorp Vault (production) / dotenv (development) | CDN provider credentials must never be stored in the database; Vault references (`vault://cdn/cloudflare/token`) in credential records; dotenv fallback for local development |
| OpenAPI | OpenAPI 3.1 via @fastify/swagger | Auto-generated from route schemas; published as `/api/docs`; used for SDK generation and provider schema validation |
| Observability | OpenTelemetry SDK + Prometheus + Grafana | OTEL traces for distributed request tracking through provider API calls; Prometheus metrics for system health; Grafana dashboards for edge analytics |

### Project Structure

```
cdn-edge-config-manager/
├── package.json                          # Root workspace config
├── pnpm-workspace.yaml
├── turbo.json
├── docker-compose.yml
├── .env.example
├── packages/
│   ├── api/                              # Fastify API server
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   ├── src/
│   │   │   ├── index.ts                  # Server bootstrap
│   │   │   ├── config.ts                 # Environment config with validation
│   │   │   ├── app.ts                    # Fastify app factory
│   │   │   ├── db/
│   │   │   │   ├── schema.ts             # Drizzle schema definitions
│   │   │   │   ├── migrations/           # Generated SQL migrations
│   │   │   │   └── seed.ts               # Seed data (provider registry)
│   │   │   ├── routes/
│   │   │   │   ├── properties.ts
│   │   │   │   ├── domains.ts
│   │   │   │   ├── origins.ts
│   │   │   │   ├── config-versions.ts
│   │   │   │   ├── edge-rules.ts
│   │   │   │   ├── traffic-policies.ts
│   │   │   │   ├── purge.ts
│   │   │   │   ├── certificates.ts
│   │   │   │   ├── providers.ts
│   │   │   │   ├── metrics.ts
│   │   │   │   ├── cost.ts
│   │   │   │   ├── ai.ts                 # AI-powered endpoints
│   │   │   │   └── auth.ts
│   │   │   ├── services/
│   │   │   │   ├── config-version.service.ts
│   │   │   │   ├── edge-rule.service.ts
│   │   │   │   ├── traffic-policy.service.ts
│   │   │   │   ├── purge.service.ts
│   │   │   │   ├── certificate.service.ts
│   │   │   │   ├── drift-detection.service.ts
│   │   │   │   ├── validation.service.ts
│   │   │   │   └── cost.service.ts
│   │   │   ├── providers/                # CDN provider adapters
│   │   │   │   ├── base.adapter.ts       # Abstract adapter interface
│   │   │   │   ├── cloudflare.adapter.ts
│   │   │   │   ├── fastly.adapter.ts
│   │   │   │   ├── akamai.adapter.ts
│   │   │   │   ├── cloudfront.adapter.ts
│   │   │   │   ├── gcore.adapter.ts
│   │   │   │   ├── bunnycdn.adapter.ts
│   │   │   │   └── registry.ts           # Provider adapter registry
│   │   │   ├── translators/              # Abstract rule → provider-native
│   │   │   │   ├── base.translator.ts
│   │   │   │   ├── cloudflare.translator.ts
│   │   │   │   ├── fastly.translator.ts
│   │   │   │   ├── akamai.translator.ts
│   │   │   │   └── registry.ts
│   │   │   ├── ai/
│   │   │   │   ├── cache-recommender.ts
│   │   │   │   ├── nl-config.ts          # Natural-language config interface
│   │   │   │   ├── anomaly-detector.ts
│   │   │   │   └── cost-predictor.ts
│   │   │   └── middleware/
│   │   │       ├── auth.ts
│   │   │       ├── org-context.ts
│   │   │       ├── rate-limit.ts
│   │   │       └── audit.ts
│   │   └── tests/
│   │       ├── unit/
│   │       ├── integration/
│   │       └── fixtures/
│   ├── worker/                           # BullMQ job processors
│   │   ├── package.json
│   │   ├── src/
│   │   │   ├── index.ts
│   │   │   ├── jobs/
│   │   │   │   ├── provider-sync.job.ts
│   │   │   │   ├── purge-propagation.job.ts
│   │   │   │   ├── certificate-renewal.job.ts
│   │   │   │   ├── drift-check.job.ts
│   │   │   │   ├── metrics-collection.job.ts
│   │   │   │   └── cost-aggregation.job.ts
│   │   │   └── queues.ts
│   │   └── tests/
│   ├── web/                              # Next.js dashboard
│   │   ├── package.json
│   │   ├── next.config.ts
│   │   ├── src/
│   │   │   ├── app/
│   │   │   │   ├── layout.tsx
│   │   │   │   ├── page.tsx              # Dashboard home
│   │   │   │   ├── properties/
│   │   │   │   ├── rules/
│   │   │   │   ├── traffic/
│   │   │   │   ├── analytics/
│   │   │   │   ├── certificates/
│   │   │   │   ├── settings/
│   │   │   │   └── api/                  # BFF routes
│   │   │   ├── components/
│   │   │   └── lib/
│   │   └── tests/
│   └── shared/                           # Shared types & utilities
│       ├── package.json
│       ├── src/
│       │   ├── types/
│       │   │   ├── property.ts
│       │   │   ├── edge-rule.ts
│       │   │   ├── traffic-policy.ts
│       │   │   ├── provider.ts
│       │   │   ├── certificate.ts
│       │   │   └── metrics.ts
│       │   ├── schemas/                  # JSON Schema definitions (Draft 2020-12)
│       │   │   ├── cache-directives.schema.json
│       │   │   ├── match-criteria.schema.json
│       │   │   ├── provider-native/
│       │   │   │   ├── cloudflare.schema.json
│       │   │   │   ├── fastly.schema.json
│       │   │   │   └── akamai.schema.json
│       │   │   └── traffic-target.schema.json
│       │   ├── validators/
│       │   │   ├── cache-rule.validator.ts
│       │   │   └── config.validator.ts
│       │   └── constants/
│       │       ├── http-methods.ts
│       │       ├── cache-directives.ts
│       │       └── provider-slugs.ts
│       └── tests/
```

---

## Phase 1: Foundation & Data Model

### Purpose

Establish the project skeleton, database schema, and development tooling. After this phase, the team has a running API server with database connectivity, migrations, seed data for the CDN provider registry, and CRUD operations for the core organizational entities (organizations, users, properties, domains, origins). No CDN-specific logic yet — just the relational backbone.

### Tasks

#### 1.1 — Monorepo Scaffold & Tooling

**What**: Initialize the pnpm workspace with all four packages (api, worker, web, shared), configure TypeScript, ESLint, Prettier, Vitest, and Turborepo.

**Design**:

Root `package.json`:
```json
{
  "name": "cdn-edge-config-manager",
  "private": true,
  "scripts": {
    "dev": "turbo run dev",
    "build": "turbo run build",
    "test": "turbo run test",
    "lint": "turbo run lint",
    "db:migrate": "pnpm --filter api db:migrate",
    "db:seed": "pnpm --filter api db:seed"
  },
  "devDependencies": {
    "turbo": "^2.4",
    "typescript": "^5.7",
    "eslint": "^9.0",
    "prettier": "^3.5"
  }
}
```

`turbo.json`:
```json
{
  "$schema": "https://turbo.build/schema.json",
  "tasks": {
    "build": { "dependsOn": ["^build"], "outputs": ["dist/**"] },
    "dev": { "cache": false, "persistent": true },
    "test": { "dependsOn": ["build"] },
    "lint": {}
  }
}
```

Shared `tsconfig.base.json`:
```json
{
  "compilerOptions": {
    "target": "ES2024",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "outDir": "./dist",
    "rootDir": "./src"
  }
}
```

**Testing**:
- `Unit: pnpm install completes without errors in all workspace packages`
- `Unit: turbo build succeeds with dependency ordering (shared → api, shared → worker, shared → web)`
- `Unit: eslint runs without errors on initial scaffold`
- `Unit: vitest discovers and runs a placeholder test in each package`

---

#### 1.2 — Database Schema & Migrations

**What**: Define the Drizzle ORM schema for core platform tables and CDN provider registry, generate and run initial migrations.

**Design**:

This phase adopts the **Hybrid Relational + JSONB** model (Data Model Suggestion 3) as the primary schema, enhanced with elements from the Entity-Centric model (Suggestion 1) for cache rule directives. The hybrid model best supports rapid multi-provider integration without schema migrations for each new provider.

Core schema definitions (`packages/api/src/db/schema.ts`):

```typescript
import { pgTable, uuid, varchar, text, boolean, integer, timestamp, jsonb, uniqueIndex, index, pgEnum, numeric, date, inet } from 'drizzle-orm/pg-core';

// --- Enums ---
export const planTierEnum = pgEnum('plan_tier', ['free', 'pro', 'business', 'enterprise']);
export const memberRoleEnum = pgEnum('member_role', ['owner', 'admin', 'editor', 'viewer']);
export const configStatusEnum = pgEnum('config_status', ['draft', 'validating', 'staged', 'active', 'superseded', 'rollback']);
export const ruleTypeEnum = pgEnum('rule_type', ['cache', 'header', 'redirect', 'rewrite', 'rate_limit', 'waf', 'geo_block', 'esi']);
export const syncStatusEnum = pgEnum('sync_status', ['pending', 'syncing', 'synced', 'drifted', 'failed', 'not_supported']);
export const purgeTypeEnum = pgEnum('purge_type', ['url', 'surrogate_key', 'prefix', 'all']);
export const acmeStatusEnum = pgEnum('acme_status', ['pending', 'ready', 'processing', 'valid', 'invalid']);

// --- Organizations & Users ---
export const organizations = pgTable('organizations', {
  id: uuid('id').primaryKey().defaultRandom(),
  name: varchar('name', { length: 255 }).notNull(),
  slug: varchar('slug', { length: 100 }).notNull().unique(),
  planTier: planTierEnum('plan_tier').notNull().default('free'),
  settings: jsonb('settings').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});

export const users = pgTable('users', {
  id: uuid('id').primaryKey().defaultRandom(),
  email: varchar('email', { length: 255 }).notNull().unique(),
  displayName: varchar('display_name', { length: 255 }),
  passwordHash: varchar('password_hash', { length: 255 }),
  mfaEnabled: boolean('mfa_enabled').notNull().default(false),
  preferences: jsonb('preferences').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});

export const organizationMembers = pgTable('organization_members', {
  id: uuid('id').primaryKey().defaultRandom(),
  organizationId: uuid('organization_id').notNull().references(() => organizations.id, { onDelete: 'cascade' }),
  userId: uuid('user_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  role: memberRoleEnum('role').notNull().default('member'),
  permissions: jsonb('permissions').notNull().default([]),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  uniqueMember: uniqueIndex('idx_org_members_unique').on(table.organizationId, table.userId),
  orgIdx: index('idx_org_members_org').on(table.organizationId),
}));

// --- Provider Connections ---
export const providerConnections = pgTable('provider_connections', {
  id: uuid('id').primaryKey().defaultRandom(),
  organizationId: uuid('organization_id').notNull().references(() => organizations.id, { onDelete: 'cascade' }),
  providerSlug: varchar('provider_slug', { length: 50 }).notNull(),
  displayName: varchar('display_name', { length: 255 }).notNull(),
  authConfig: jsonb('auth_config').notNull(),
  providerCapabilities: jsonb('provider_capabilities').notNull().default({}),
  isActive: boolean('is_active').notNull().default(true),
  lastValidatedAt: timestamp('last_validated_at', { withTimezone: true }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  orgIdx: index('idx_provider_conn_org').on(table.organizationId),
  slugIdx: index('idx_provider_conn_slug').on(table.providerSlug),
}));

// --- Properties, Domains, Origins ---
export const properties = pgTable('properties', {
  id: uuid('id').primaryKey().defaultRandom(),
  organizationId: uuid('organization_id').notNull().references(() => organizations.id, { onDelete: 'cascade' }),
  name: varchar('name', { length: 255 }).notNull(),
  slug: varchar('slug', { length: 100 }).notNull(),
  description: text('description'),
  tags: jsonb('tags').notNull().default([]),
  metadata: jsonb('metadata').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  uniqueSlug: uniqueIndex('idx_properties_org_slug').on(table.organizationId, table.slug),
  orgIdx: index('idx_properties_org').on(table.organizationId),
}));

export const domains = pgTable('domains', {
  id: uuid('id').primaryKey().defaultRandom(),
  propertyId: uuid('property_id').notNull().references(() => properties.id, { onDelete: 'cascade' }),
  hostname: varchar('hostname', { length: 500 }).notNull().unique(),
  isPrimary: boolean('is_primary').notNull().default(false),
  protocolConfig: jsonb('protocol_config').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  propertyIdx: index('idx_domains_property').on(table.propertyId),
}));

export const origins = pgTable('origins', {
  id: uuid('id').primaryKey().defaultRandom(),
  propertyId: uuid('property_id').notNull().references(() => properties.id, { onDelete: 'cascade' }),
  name: varchar('name', { length: 255 }).notNull(),
  hostname: varchar('hostname', { length: 500 }).notNull(),
  port: integer('port').notNull().default(443),
  protocol: varchar('protocol', { length: 10 }).notNull().default('https'),
  healthCheck: jsonb('health_check').notNull().default({}),
  connectionConfig: jsonb('connection_config').notNull().default({}),
  weight: integer('weight').notNull().default(100),
  isBackup: boolean('is_backup').notNull().default(false),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  propertyIdx: index('idx_origins_property').on(table.propertyId),
}));
```

Seed data for provider registry (`packages/api/src/db/seed.ts`):

```typescript
const CDN_PROVIDERS = [
  { slug: 'cloudflare', displayName: 'Cloudflare', authMethod: 'bearer_token', apiBaseUrl: 'https://api.cloudflare.com/client/v4' },
  { slug: 'fastly', displayName: 'Fastly', authMethod: 'api_key', apiBaseUrl: 'https://api.fastly.com' },
  { slug: 'akamai', displayName: 'Akamai', authMethod: 'edgegrid', apiBaseUrl: 'https://{host}' },
  { slug: 'aws_cloudfront', displayName: 'AWS CloudFront', authMethod: 'sigv4', apiBaseUrl: 'https://cloudfront.amazonaws.com' },
  { slug: 'gcore', displayName: 'Gcore', authMethod: 'bearer_token', apiBaseUrl: 'https://api.gcore.com' },
  { slug: 'bunnycdn', displayName: 'Bunny.net', authMethod: 'api_key', apiBaseUrl: 'https://api.bunny.net' },
] as const;
```

**Testing**:
- `Unit: Drizzle schema compiles with strict TypeScript — all table definitions produce correct InferSelectModel types`
- `Integration (real DB): migration applies cleanly to empty PostgreSQL database`
- `Integration (real DB): migration rollback removes all tables without errors`
- `Integration (real DB): seed data inserts all 6 CDN providers; SELECT count confirms 6 rows`
- `Integration (real DB): unique constraints enforced — duplicate organization slug raises constraint violation`
- `Unit: JSONB default values produce correct empty objects/arrays when no explicit value provided`

---

#### 1.3 — Fastify API Server Bootstrap

**What**: Configure the Fastify application with plugins for CORS, Swagger/OpenAPI, request validation, error handling, and health checks.

**Design**:

Application config (`packages/api/src/config.ts`):

```typescript
import { Type, Static } from '@sinclair/typebox';

const ConfigSchema = Type.Object({
  PORT: Type.Number({ default: 3001 }),
  HOST: Type.String({ default: '0.0.0.0' }),
  DATABASE_URL: Type.String(),
  REDIS_URL: Type.String({ default: 'redis://localhost:6379' }),
  JWT_SECRET: Type.String(),
  VAULT_ADDR: Type.Optional(Type.String()),
  VAULT_TOKEN: Type.Optional(Type.String()),
  LOG_LEVEL: Type.Union([
    Type.Literal('fatal'), Type.Literal('error'), Type.Literal('warn'),
    Type.Literal('info'), Type.Literal('debug'), Type.Literal('trace'),
  ], { default: 'info' }),
  NODE_ENV: Type.Union([
    Type.Literal('development'), Type.Literal('test'), Type.Literal('production'),
  ], { default: 'development' }),
});

export type AppConfig = Static<typeof ConfigSchema>;
```

App factory (`packages/api/src/app.ts`):

```typescript
import Fastify, { FastifyInstance } from 'fastify';
import fastifySwagger from '@fastify/swagger';
import fastifySwaggerUi from '@fastify/swagger-ui';
import fastifyCors from '@fastify/cors';

export async function buildApp(config: AppConfig): Promise<FastifyInstance> {
  const app = Fastify({ logger: { level: config.LOG_LEVEL } });

  await app.register(fastifyCors, { origin: true });
  await app.register(fastifySwagger, {
    openapi: {
      openapi: '3.1.0',
      info: { title: 'CDN Edge Configuration Manager API', version: '1.0.0' },
      servers: [{ url: `http://${config.HOST}:${config.PORT}` }],
    },
  });
  await app.register(fastifySwaggerUi, { routePrefix: '/api/docs' });

  // Health check
  app.get('/health', async () => ({ status: 'ok', timestamp: new Date().toISOString() }));

  return app;
}
```

**Testing**:
- `Integration: GET /health returns { status: "ok" } with 200`
- `Integration: GET /api/docs returns Swagger UI HTML page`
- `Integration: GET /api/docs/json returns valid OpenAPI 3.1 JSON document`
- `Unit: buildApp with missing DATABASE_URL throws validation error with field name`
- `Unit: buildApp accepts all valid LOG_LEVEL values without error`

---

#### 1.4 — Organization & User CRUD Routes

**What**: Implement RESTful CRUD endpoints for organizations, users, and organization membership.

**Design**:

Route schemas using TypeBox (type-safe request/response):

```typescript
import { Type, Static } from '@sinclair/typebox';

// --- Request/Response Schemas ---
export const CreateOrganizationSchema = Type.Object({
  name: Type.String({ minLength: 1, maxLength: 255 }),
  slug: Type.String({ minLength: 1, maxLength: 100, pattern: '^[a-z0-9-]+$' }),
  billingEmail: Type.Optional(Type.String({ format: 'email' })),
  planTier: Type.Optional(Type.Union([
    Type.Literal('free'), Type.Literal('pro'),
    Type.Literal('business'), Type.Literal('enterprise'),
  ])),
});

export const OrganizationResponseSchema = Type.Object({
  id: Type.String({ format: 'uuid' }),
  name: Type.String(),
  slug: Type.String(),
  planTier: Type.String(),
  settings: Type.Any(),
  createdAt: Type.String({ format: 'date-time' }),
  updatedAt: Type.String({ format: 'date-time' }),
});
```

API endpoints:

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/organizations` | Create organization |
| GET | `/api/v1/organizations` | List organizations for current user |
| GET | `/api/v1/organizations/:orgSlug` | Get organization by slug |
| PATCH | `/api/v1/organizations/:orgSlug` | Update organization |
| DELETE | `/api/v1/organizations/:orgSlug` | Delete organization |
| POST | `/api/v1/organizations/:orgSlug/members` | Add member to org |
| GET | `/api/v1/organizations/:orgSlug/members` | List org members |
| PATCH | `/api/v1/organizations/:orgSlug/members/:userId` | Update member role |
| DELETE | `/api/v1/organizations/:orgSlug/members/:userId` | Remove member |

**Testing**:
- `Integration: POST /api/v1/organizations with valid body → 201 with created org`
- `Integration: POST /api/v1/organizations with duplicate slug → 409 Conflict`
- `Integration: POST /api/v1/organizations with invalid slug format → 400 with validation details`
- `Integration: GET /api/v1/organizations/:orgSlug → 200 with org details`
- `Integration: GET /api/v1/organizations/nonexistent → 404`
- `Integration: DELETE /api/v1/organizations/:orgSlug → 204; cascades delete members`
- `Integration: POST /api/v1/organizations/:orgSlug/members adds user with default 'member' role`
- `Integration: POST /api/v1/organizations/:orgSlug/members with duplicate user → 409`

---

#### 1.5 — Property, Domain, and Origin CRUD Routes

**What**: Implement RESTful CRUD for properties (scoped to organization), domains (scoped to property), and origins (scoped to property).

**Design**:

```typescript
export const CreatePropertySchema = Type.Object({
  name: Type.String({ minLength: 1, maxLength: 255 }),
  slug: Type.String({ minLength: 1, maxLength: 100, pattern: '^[a-z0-9-]+$' }),
  description: Type.Optional(Type.String()),
  tags: Type.Optional(Type.Array(Type.String())),
  metadata: Type.Optional(Type.Record(Type.String(), Type.Any())),
});

export const CreateDomainSchema = Type.Object({
  hostname: Type.String({ minLength: 1, maxLength: 500 }),
  isPrimary: Type.Optional(Type.Boolean()),
  protocolConfig: Type.Optional(Type.Object({
    tls: Type.Optional(Type.Object({
      enabled: Type.Boolean({ default: true }),
      minVersion: Type.Optional(Type.Union([
        Type.Literal('1.0'), Type.Literal('1.1'),
        Type.Literal('1.2'), Type.Literal('1.3'),
      ])),
    })),
    http3: Type.Optional(Type.Object({
      enabled: Type.Boolean({ default: false }),
      quic: Type.Optional(Type.Object({
        connectionMigration: Type.Boolean({ default: true }),
        zeroRtt: Type.Boolean({ default: false }),
      })),
    })),
  })),
});

export const CreateOriginSchema = Type.Object({
  name: Type.String({ minLength: 1, maxLength: 255 }),
  hostname: Type.String({ minLength: 1, maxLength: 500 }),
  port: Type.Optional(Type.Integer({ minimum: 1, maximum: 65535, default: 443 })),
  protocol: Type.Optional(Type.Union([Type.Literal('http'), Type.Literal('https')])),
  weight: Type.Optional(Type.Integer({ minimum: 0, maximum: 1000, default: 100 })),
  isBackup: Type.Optional(Type.Boolean()),
  healthCheck: Type.Optional(Type.Object({
    path: Type.String({ default: '/health' }),
    intervalSec: Type.Integer({ default: 30 }),
    timeoutMs: Type.Integer({ default: 5000 }),
    healthyThreshold: Type.Integer({ default: 3 }),
    unhealthyThreshold: Type.Integer({ default: 2 }),
    expectedStatus: Type.Array(Type.Integer(), { default: [200] }),
  })),
  connectionConfig: Type.Optional(Type.Object({
    connectTimeoutMs: Type.Integer({ default: 5000 }),
    readTimeoutMs: Type.Integer({ default: 30000 }),
    keepaliveConnections: Type.Integer({ default: 64 }),
    retryCount: Type.Integer({ default: 2 }),
  })),
});
```

API endpoints (all scoped under `/:orgSlug`):

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/v1/organizations/:orgSlug/properties` | Create property |
| GET | `/api/v1/organizations/:orgSlug/properties` | List properties |
| GET | `/api/v1/organizations/:orgSlug/properties/:propertySlug` | Get property |
| PATCH | `/api/v1/organizations/:orgSlug/properties/:propertySlug` | Update property |
| DELETE | `/api/v1/organizations/:orgSlug/properties/:propertySlug` | Delete property |
| POST | `/api/v1/organizations/:orgSlug/properties/:propertySlug/domains` | Add domain |
| GET | `/api/v1/organizations/:orgSlug/properties/:propertySlug/domains` | List domains |
| DELETE | `/api/v1/organizations/:orgSlug/properties/:propertySlug/domains/:domainId` | Remove domain |
| POST | `/api/v1/organizations/:orgSlug/properties/:propertySlug/origins` | Add origin |
| GET | `/api/v1/organizations/:orgSlug/properties/:propertySlug/origins` | List origins |
| PATCH | `/api/v1/organizations/:orgSlug/properties/:propertySlug/origins/:originId` | Update origin |
| DELETE | `/api/v1/organizations/:orgSlug/properties/:propertySlug/origins/:originId` | Remove origin |

**Testing**:
- `Integration: POST property with valid body → 201; GET returns it`
- `Integration: POST property with duplicate slug within same org → 409`
- `Integration: POST domain with hostname already in use (globally) → 409`
- `Integration: POST domain with HTTP/3 QUIC config → stored correctly in protocolConfig JSONB`
- `Integration: POST origin with default health check values → defaults populated in response`
- `Integration: DELETE property cascades to domains and origins`
- `Unit: CreateDomainSchema rejects invalid TLS min version "1.4"`
- `Unit: CreateOriginSchema rejects port value 0 and 70000`

---

#### 1.6 — Docker Compose Development Environment

**What**: Create Docker Compose configuration for local development with PostgreSQL 16, Redis 7, and application services.

**Design**:

```yaml
# docker-compose.yml
services:
  postgres:
    image: postgres:16-alpine
    ports:
      - "5432:5432"
    environment:
      POSTGRES_DB: cdn_edge_config
      POSTGRES_USER: cdn_admin
      POSTGRES_PASSWORD: dev_password
    volumes:
      - postgres_data:/var/lib/postgresql/data
    command: >
      postgres
        -c shared_preload_libraries='pg_stat_statements'
        -c log_min_duration_statement=100

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    command: redis-server --maxmemory 256mb --maxmemory-policy allkeys-lru

  api:
    build:
      context: .
      dockerfile: packages/api/Dockerfile
      target: development
    ports:
      - "3001:3001"
    environment:
      DATABASE_URL: postgresql://cdn_admin:dev_password@postgres:5432/cdn_edge_config
      REDIS_URL: redis://redis:6379
      JWT_SECRET: dev-jwt-secret-not-for-production
      LOG_LEVEL: debug
    depends_on:
      - postgres
      - redis
    volumes:
      - ./packages:/app/packages
    command: pnpm --filter api dev

volumes:
  postgres_data:
```

Multi-stage Dockerfile (`packages/api/Dockerfile`):

```dockerfile
FROM node:22-alpine AS base
RUN corepack enable && corepack prepare pnpm@9 --activate
WORKDIR /app

FROM base AS development
COPY pnpm-workspace.yaml pnpm-lock.yaml ./
COPY packages/api/package.json packages/api/
COPY packages/shared/package.json packages/shared/
RUN pnpm install --frozen-lockfile
COPY . .
CMD ["pnpm", "--filter", "api", "dev"]

FROM base AS production
COPY pnpm-workspace.yaml pnpm-lock.yaml ./
COPY packages/api/package.json packages/api/
COPY packages/shared/package.json packages/shared/
RUN pnpm install --frozen-lockfile --prod
COPY packages/api/dist packages/api/dist
COPY packages/shared/dist packages/shared/dist
CMD ["node", "packages/api/dist/index.js"]
```

**Testing**:
- `E2E: docker compose up starts all services; API responds to GET /health within 30 seconds`
- `E2E: docker compose down --volumes cleans up; re-up from scratch succeeds`
- `Integration: API container can reach PostgreSQL and run migrations`
- `Integration: API container can reach Redis and set/get a key`

---

## Phase 2: Configuration Versioning & Edge Rules

### Purpose

Implement the configuration versioning system (draft → validate → stage → activate lifecycle) and the edge rule engine with abstract match criteria and cache directives aligned to RFC 9111/9213. After this phase, users can create versioned configurations, add cache/header/WAF rules with portable match criteria, validate configurations, and activate them. No provider sync yet.

### Tasks

#### 2.1 — Config Version Lifecycle

**What**: Implement the configuration version state machine with immutable activated versions and version lineage tracking.

**Design**:

State machine for config versions:

```
  draft → validating → staged → active → superseded
    ↑         ↓                            ↓
    └── (new version) ←── rollback ←──────┘
```

```typescript
// packages/shared/src/types/config-version.ts
export interface ConfigVersion {
  id: string;
  propertyId: string;
  versionNumber: number;
  status: 'draft' | 'validating' | 'staged' | 'active' | 'superseded' | 'rollback';
  description: string | null;
  createdBy: string | null;
  activatedAt: string | null;
  activatedBy: string | null;
  parentVersionId: string | null;
  validationResults: ValidationResults | null;
  createdAt: string;
  updatedAt: string;
}

export interface ValidationResults {
  valid: boolean;
  errors: ValidationIssue[];
  warnings: ValidationIssue[];
}

export interface ValidationIssue {
  ruleId?: string;
  code: string;
  message: string;
  severity: 'error' | 'warning';
}
```

Service methods:

```typescript
// packages/api/src/services/config-version.service.ts
export class ConfigVersionService {
  async createVersion(propertyId: string, opts: { description?: string; basedOn?: number }): Promise<ConfigVersion>;
  async getVersion(propertyId: string, versionNumber: number): Promise<ConfigVersion>;
  async listVersions(propertyId: string, opts?: { status?: string; limit?: number }): Promise<ConfigVersion[]>;
  async validateVersion(versionId: string): Promise<ValidationResults>;
  async stageVersion(versionId: string): Promise<ConfigVersion>;
  async activateVersion(versionId: string, activatedBy: string): Promise<ConfigVersion>;
  async rollbackVersion(propertyId: string, targetVersion: number): Promise<ConfigVersion>;
  async diffVersions(versionA: string, versionB: string): Promise<VersionDiff>;
}
```

API endpoints:

| Method | Path | Description |
|--------|------|-------------|
| POST | `.../:propertySlug/versions` | Create new version |
| GET | `.../:propertySlug/versions` | List versions |
| GET | `.../:propertySlug/versions/:versionNumber` | Get version detail |
| POST | `.../:propertySlug/versions/:versionNumber/validate` | Validate version |
| POST | `.../:propertySlug/versions/:versionNumber/stage` | Stage version |
| POST | `.../:propertySlug/versions/:versionNumber/activate` | Activate version |
| POST | `.../:propertySlug/versions/rollback` | Rollback to version |
| GET | `.../:propertySlug/versions/diff?a=N&b=M` | Diff two versions |

**Testing**:
- `Integration: create version → status is 'draft', versionNumber auto-increments`
- `Integration: activate draft version without validating → 400 error`
- `Integration: validate → stage → activate lifecycle completes successfully`
- `Integration: activating version N sets previous active version to 'superseded'`
- `Integration: modifying an activated version → 400 "immutable" error`
- `Integration: rollback creates a new version based on target, status 'rollback'`
- `Integration: diff two versions → returns added/removed/modified rules`
- `Unit: version numbering starts at 1, increments sequentially per property`

---

#### 2.2 — Edge Rule CRUD with Abstract Match Criteria

**What**: Implement the edge rules table with JSONB match criteria and action payloads, plus CRUD routes with JSON Schema validation.

**Design**:

```typescript
// packages/shared/src/types/edge-rule.ts
export interface EdgeRule {
  id: string;
  configVersionId: string;
  ruleType: 'cache' | 'header' | 'redirect' | 'rewrite' | 'rate_limit' | 'waf' | 'geo_block' | 'esi';
  name: string;
  priority: number;
  isEnabled: boolean;
  matchCriteria: MatchCriteria;
  action: RuleAction;
  aiAnalysis: AiAnalysis | null;
  createdAt: string;
  updatedAt: string;
}

// RFC 9111 aligned cache directives
export interface CacheDirectives {
  maxAge?: number;          // Cache-Control: max-age (seconds)
  sMaxage?: number;         // Cache-Control: s-maxage (seconds)
  staleWhileRevalidate?: number;  // RFC 5861
  staleIfError?: number;    // RFC 5861
  mustRevalidate?: boolean;
  noCache?: boolean;
  noStore?: boolean;
  isPrivate?: boolean;
  noTransform?: boolean;
  cdnCacheControl?: {       // RFC 9213
    maxAge?: number;
  };
  surrogateKey?: string;    // for tag-based purging
  serveStaleOnError?: boolean;
  respectOriginHeaders?: boolean;
}

export interface CacheAction {
  type: 'cache';
  cacheAction: 'cache' | 'bypass' | 'revalidate' | 'custom';
  directives: CacheDirectives;
}

export interface HeaderAction {
  type: 'set_header';
  headers: Array<{
    name: string;
    value: string;
    target: 'request' | 'response';
    action: 'set' | 'append' | 'remove';
  }>;
}

export interface WafAction {
  type: 'waf';
  mode: 'block' | 'simulate' | 'challenge' | 'disabled';
  owaspCategories: string[];  // e.g., ['A01', 'A03']
  sensitivity: 'low' | 'medium' | 'high';
}

export type RuleAction = CacheAction | HeaderAction | WafAction;

// Match criteria support composable boolean logic
export type MatchCriteria =
  | { path: { pattern: string; type: 'glob' | 'regex' | 'exact' } }
  | { methods: string[] }
  | { contentTypes: string[] }
  | { header: { name: string; contains?: string; equals?: string; exists?: boolean } }
  | { geo: { countries: string[]; exclude?: boolean } }
  | { queryString: { contains?: string; key?: string; value?: string } }
  | { and: MatchCriteria[] }
  | { or: MatchCriteria[] }
  | { not: MatchCriteria };
```

JSON Schema for match criteria validation (`packages/shared/src/schemas/match-criteria.schema.json`):

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://cdn-edge-config.dev/schemas/match-criteria",
  "title": "MatchCriteria",
  "oneOf": [
    { "type": "object", "properties": { "path": { "type": "object", "properties": { "pattern": { "type": "string" }, "type": { "enum": ["glob", "regex", "exact"] } }, "required": ["pattern", "type"] } }, "required": ["path"] },
    { "type": "object", "properties": { "methods": { "type": "array", "items": { "enum": ["GET", "HEAD", "POST", "PUT", "PATCH", "DELETE", "OPTIONS"] } } }, "required": ["methods"] },
    { "type": "object", "properties": { "and": { "type": "array", "items": { "$ref": "#" }, "minItems": 2 } }, "required": ["and"] },
    { "type": "object", "properties": { "or": { "type": "array", "items": { "$ref": "#" }, "minItems": 2 } }, "required": ["or"] },
    { "type": "object", "properties": { "not": { "$ref": "#" } }, "required": ["not"] }
  ]
}
```

**Testing**:
- `Integration: POST edge rule with cache action and RFC 9111 directives → 201, directives stored correctly`
- `Integration: POST edge rule with nested AND/OR/NOT match criteria → stored and retrieved correctly`
- `Integration: POST edge rule with invalid match criteria (missing path.type) → 400 with JSON Schema validation error`
- `Integration: POST cache rule with negative max_age → 400 validation error`
- `Integration: GET rules for version → ordered by priority ascending`
- `Integration: PATCH rule priority reorders correctly`
- `Integration: DELETE rule on activated version → 400 immutable`
- `Unit: CacheDirectives validates stale-while-revalidate must be positive integer`
- `Unit: MatchCriteria rejects geo.countries with invalid ISO 3166-1 codes`
- `Fixture: Load complex_rules.json fixture with 20 rules of all types → all validate and insert`

---

#### 2.3 — Configuration Validation Engine

**What**: Implement a validation engine that checks a config version's rules for conflicts, coverage gaps, and standards compliance.

**Design**:

```typescript
// packages/api/src/services/validation.service.ts
export class ValidationService {
  async validateConfigVersion(versionId: string): Promise<ValidationResults> {
    const rules = await this.getRulesForVersion(versionId);
    const errors: ValidationIssue[] = [];
    const warnings: ValidationIssue[] = [];

    // 1. Check for overlapping path patterns at the same priority
    errors.push(...this.detectPathConflicts(rules));

    // 2. Check for unreachable rules (shadowed by higher priority)
    warnings.push(...this.detectShadowedRules(rules));

    // 3. RFC 9111 compliance checks
    errors.push(...this.validateCacheDirectives(rules));

    // 4. Security header coverage
    warnings.push(...this.checkSecurityHeaders(rules));

    // 5. Overly broad bypass detection
    warnings.push(...this.detectBroadBypasses(rules));

    return { valid: errors.length === 0, errors, warnings };
  }

  private detectPathConflicts(rules: EdgeRule[]): ValidationIssue[];
  private detectShadowedRules(rules: EdgeRule[]): ValidationIssue[];
  private validateCacheDirectives(rules: EdgeRule[]): ValidationIssue[];
  private checkSecurityHeaders(rules: EdgeRule[]): ValidationIssue[];
  private detectBroadBypasses(rules: EdgeRule[]): ValidationIssue[];
}
```

Validation rules:

| Code | Severity | Description |
|------|----------|-------------|
| `PATH_CONFLICT` | error | Two rules at same priority match overlapping paths |
| `SHADOWED_RULE` | warning | Rule never fires because a higher-priority rule matches all its paths |
| `INVALID_CACHE_COMBO` | error | `no-store` combined with `max-age` (RFC 9111 violation) |
| `STALE_TTL_EXCESSIVE` | warning | `stale-while-revalidate` exceeds 30 days |
| `MISSING_SECURITY_HEADERS` | warning | No rules set X-Frame-Options, X-Content-Type-Options, or CSP |
| `BROAD_BYPASS` | warning | Cache bypass rule matches `/*` without additional conditions |
| `SMAX_WITHOUT_PUBLIC` | warning | `s-maxage` set but response may be `private` |

**Testing**:
- `Unit: two cache rules at priority 0 both matching "/static/*" → PATH_CONFLICT error`
- `Unit: rule matching "/*" at priority 0 shadows rule matching "/static/*" at priority 1 → SHADOWED_RULE warning`
- `Unit: rule with no_store=true and max_age=3600 → INVALID_CACHE_COMBO error`
- `Unit: stale_while_revalidate = 5184000 (60 days) → STALE_TTL_EXCESSIVE warning`
- `Unit: no header rules setting X-Frame-Options → MISSING_SECURITY_HEADERS warning`
- `Unit: bypass rule matching "/*" with no conditions → BROAD_BYPASS warning`
- `Integration: POST validate version → returns results; version status changes to 'validating' then back to 'draft' (with results stored)`
- `Fixture: valid_complete_config.json with 15 rules → validates with 0 errors, 0 warnings`

---

#### 2.4 — Audit Logging

**What**: Implement an append-only audit log that records all configuration changes with user context, IP address, and change diffs.

**Design**:

```typescript
// packages/api/src/db/schema.ts (addition)
export const auditLog = pgTable('audit_log', {
  id: uuid('id').primaryKey().defaultRandom(),
  organizationId: uuid('organization_id').notNull().references(() => organizations.id),
  userId: uuid('user_id').references(() => users.id),
  action: varchar('action', { length: 100 }).notNull(),
  resourceType: varchar('resource_type', { length: 100 }).notNull(),
  resourceId: uuid('resource_id'),
  changes: jsonb('changes'),
  requestContext: jsonb('request_context'),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  orgTimeIdx: index('idx_audit_org_time').on(table.organizationId, table.createdAt),
  resourceIdx: index('idx_audit_resource').on(table.resourceType, table.resourceId),
}));

// Audit middleware
export async function auditMiddleware(request: FastifyRequest, reply: FastifyReply) {
  reply.then(() => {
    if (['POST', 'PATCH', 'PUT', 'DELETE'].includes(request.method)) {
      // Enqueue audit log write (non-blocking)
      auditQueue.add('log', {
        organizationId: request.orgContext.id,
        userId: request.user?.id,
        action: `${request.routeOptions.config.resourceType}.${methodToVerb(request.method)}`,
        resourceType: request.routeOptions.config.resourceType,
        resourceId: reply.getHeader('x-resource-id'),
        changes: request.routeOptions.config.computeDiff?.(request),
        requestContext: {
          ip: request.ip,
          userAgent: request.headers['user-agent'],
          traceId: request.headers['traceparent']?.split('-')[1],
        },
      });
    }
  });
}
```

**Testing**:
- `Integration: POST property → audit log entry created with action 'property.create'`
- `Integration: PATCH edge rule → audit log entry includes before/after diff in changes JSONB`
- `Integration: DELETE domain → audit log entry includes request IP and user agent`
- `Integration: GET (read-only) requests → no audit log entry created`
- `Unit: audit log entries are immutable — no UPDATE or DELETE operations via service`
- `Integration: query audit log by organization + time range → returns filtered entries ordered by time desc`

---

## Phase 3: CDN Provider Adapter Framework

### Purpose

Build the abstraction layer for communicating with CDN provider APIs. Define the adapter interface, implement the rule translator framework (abstract rules to provider-native config), and implement two initial adapters (Cloudflare and Fastly) with credential management. After this phase, the platform can translate abstract edge rules into provider-specific formats and verify credentials.

### Tasks

#### 3.1 — Provider Adapter Interface

**What**: Define the abstract interface that all CDN provider adapters must implement.

**Design**:

```typescript
// packages/api/src/providers/base.adapter.ts
export interface ProviderCredentials {
  providerSlug: string;
  authConfig: Record<string, unknown>;
}

export interface ProviderResource {
  resourceId: string;
  resourceType: string;
  providerVersion?: string;
  rawConfig: Record<string, unknown>;
}

export interface PurgeResult {
  purgeId: string;
  status: 'submitted' | 'completed' | 'failed';
  propagationMs?: number;
  error?: string;
}

export interface ProviderHealthStatus {
  healthy: boolean;
  latencyMs: number;
  apiVersion?: string;
  quotaRemaining?: number;
}

export abstract class CdnProviderAdapter {
  abstract readonly slug: string;
  abstract readonly displayName: string;
  abstract readonly authMethod: 'bearer_token' | 'api_key' | 'edgegrid' | 'sigv4' | 'oauth2';

  // Credential lifecycle
  abstract validateCredentials(creds: ProviderCredentials): Promise<boolean>;
  abstract discoverCapabilities(creds: ProviderCredentials): Promise<Record<string, unknown>>;

  // Configuration sync
  abstract getRemoteConfig(creds: ProviderCredentials, resourceId: string): Promise<ProviderResource>;
  abstract applyConfig(creds: ProviderCredentials, config: Record<string, unknown>): Promise<ProviderResource>;
  abstract detectDrift(creds: ProviderCredentials, expected: Record<string, unknown>, resourceId: string): Promise<DriftResult>;

  // Cache operations
  abstract purgeByUrl(creds: ProviderCredentials, urls: string[]): Promise<PurgeResult>;
  abstract purgeBySurrogateKey(creds: ProviderCredentials, keys: string[]): Promise<PurgeResult>;
  abstract purgeAll(creds: ProviderCredentials): Promise<PurgeResult>;

  // Health & metrics
  abstract healthCheck(creds: ProviderCredentials): Promise<ProviderHealthStatus>;
  abstract fetchMetrics(creds: ProviderCredentials, opts: MetricsQuery): Promise<EdgeMetrics>;
}

export interface DriftResult {
  hasDrift: boolean;
  diffs: Array<{
    path: string;
    expected: unknown;
    actual: unknown;
  }>;
}

export interface MetricsQuery {
  resourceId: string;
  start: Date;
  end: Date;
  granularity: '1m' | '5m' | '1h' | '1d';
}

export interface EdgeMetrics {
  requests: number;
  cacheHits: number;
  cacheMisses: number;
  bytesSent: number;
  errors4xx: number;
  errors5xx: number;
  latencyP50Ms: number;
  latencyP95Ms: number;
  latencyP99Ms: number;
}
```

Provider adapter registry:

```typescript
// packages/api/src/providers/registry.ts
export class ProviderRegistry {
  private adapters = new Map<string, CdnProviderAdapter>();

  register(adapter: CdnProviderAdapter): void {
    this.adapters.set(adapter.slug, adapter);
  }

  get(slug: string): CdnProviderAdapter {
    const adapter = this.adapters.get(slug);
    if (!adapter) throw new ProviderNotFoundError(slug);
    return adapter;
  }

  list(): CdnProviderAdapter[] {
    return Array.from(this.adapters.values());
  }
}
```

**Testing**:
- `Unit: ProviderRegistry.register and get round-trip correctly`
- `Unit: ProviderRegistry.get with unknown slug throws ProviderNotFoundError`
- `Unit: ProviderRegistry.list returns all registered adapters`
- `Unit: CdnProviderAdapter abstract class enforces implementation of all required methods`

---

#### 3.2 — Rule Translator Framework

**What**: Build the framework that translates abstract edge rules (match criteria + action) into provider-native configuration and stores results in `provider_rule_configs`.

**Design**:

```typescript
// packages/api/src/translators/base.translator.ts
export interface TranslationResult {
  nativeConfig: Record<string, unknown>;
  warnings: string[];
  unsupportedFeatures: string[];
}

export abstract class RuleTranslator {
  abstract readonly providerSlug: string;

  abstract translateCacheRule(rule: EdgeRule & { action: CacheAction }): TranslationResult;
  abstract translateHeaderRule(rule: EdgeRule & { action: HeaderAction }): TranslationResult;
  abstract translateWafRule(rule: EdgeRule & { action: WafAction }): TranslationResult;

  translate(rule: EdgeRule): TranslationResult {
    switch (rule.action.type) {
      case 'cache': return this.translateCacheRule(rule as EdgeRule & { action: CacheAction });
      case 'set_header': return this.translateHeaderRule(rule as EdgeRule & { action: HeaderAction });
      case 'waf': return this.translateWafRule(rule as EdgeRule & { action: WafAction });
      default: return { nativeConfig: {}, warnings: [], unsupportedFeatures: [`Rule type ${rule.ruleType} not supported`] };
    }
  }

  protected translateMatchToExpression(criteria: MatchCriteria): string {
    // Subclasses override for provider-specific expression syntax
    throw new Error('Not implemented');
  }
}
```

Database schema addition:

```typescript
export const providerRuleConfigs = pgTable('provider_rule_configs', {
  id: uuid('id').primaryKey().defaultRandom(),
  edgeRuleId: uuid('edge_rule_id').notNull().references(() => edgeRules.id, { onDelete: 'cascade' }),
  providerSlug: varchar('provider_slug', { length: 50 }).notNull(),
  connectionId: uuid('connection_id').notNull().references(() => providerConnections.id),
  nativeConfig: jsonb('native_config').notNull(),
  syncStatus: syncStatusEnum('sync_status').notNull().default('pending'),
  providerResourceId: varchar('provider_resource_id', { length: 500 }),
  lastSyncedAt: timestamp('last_synced_at', { withTimezone: true }),
  driftDiff: jsonb('drift_diff'),
  errorMessage: text('error_message'),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  uniqueRuleProvider: uniqueIndex('idx_rule_provider_unique').on(table.edgeRuleId, table.providerSlug),
  ruleIdx: index('idx_provider_rules_rule').on(table.edgeRuleId),
  syncIdx: index('idx_provider_rules_sync').on(table.syncStatus),
}));
```

**Testing**:
- `Unit: base translator dispatches cache rules to translateCacheRule method`
- `Unit: translator returns unsupportedFeatures for rule types the provider does not handle`
- `Integration: translating a rule stores result in provider_rule_configs with sync_status 'pending'`
- `Unit: translating all rules for a version populates one provider_rule_configs row per rule per connected provider`

---

#### 3.3 — Cloudflare Adapter & Translator

**What**: Implement the Cloudflare provider adapter using the Cloudflare API v4 and the rule translator that converts abstract rules to Cloudflare ruleset expressions.

**Design**:

```typescript
// packages/api/src/providers/cloudflare.adapter.ts
export class CloudflareAdapter extends CdnProviderAdapter {
  readonly slug = 'cloudflare';
  readonly displayName = 'Cloudflare';
  readonly authMethod = 'bearer_token' as const;

  private client(creds: ProviderCredentials): CloudflareClient {
    const token = creds.authConfig.tokenRef; // Resolve from vault
    return new Cloudflare({ apiToken: token as string });
  }

  async validateCredentials(creds: ProviderCredentials): Promise<boolean> {
    const cf = this.client(creds);
    const result = await cf.user.tokens.verify();
    return result.status === 'active';
  }

  async purgeByUrl(creds: ProviderCredentials, urls: string[]): Promise<PurgeResult> {
    const cf = this.client(creds);
    const zoneId = creds.authConfig.zoneId as string;
    const result = await cf.cache.purge({ zone_id: zoneId, files: urls });
    return { purgeId: result.id, status: 'submitted' };
  }
  // ... other methods
}

// packages/api/src/translators/cloudflare.translator.ts
export class CloudflareTranslator extends RuleTranslator {
  readonly providerSlug = 'cloudflare';

  translateCacheRule(rule: EdgeRule & { action: CacheAction }): TranslationResult {
    const expression = this.matchToCloudflareExpression(rule.matchCriteria);
    return {
      nativeConfig: {
        ruleset_phase: 'http_request_cache_settings',
        expression,
        action: 'set_cache_settings',
        action_parameters: {
          cache: rule.action.cacheAction !== 'bypass',
          edge_ttl: rule.action.directives.sMaxage
            ? { mode: 'override_origin', default: rule.action.directives.sMaxage }
            : undefined,
          browser_ttl: rule.action.directives.maxAge
            ? { mode: 'override_origin', default: rule.action.directives.maxAge }
            : undefined,
        },
      },
      warnings: [],
      unsupportedFeatures: [],
    };
  }

  private matchToCloudflareExpression(criteria: MatchCriteria): string {
    if ('path' in criteria) {
      const op = criteria.path.type === 'glob' ? 'wildcard' : 'matches';
      return `(http.request.uri.path ${op} "${criteria.path.pattern}")`;
    }
    if ('and' in criteria) {
      return criteria.and.map(c => this.matchToCloudflareExpression(c)).join(' and ');
    }
    if ('or' in criteria) {
      return criteria.or.map(c => this.matchToCloudflareExpression(c)).join(' or ');
    }
    if ('not' in criteria) {
      return `not ${this.matchToCloudflareExpression(criteria.not)}`;
    }
    return 'true'; // fallback
  }
}
```

**Testing**:
- `Unit: CloudflareTranslator translates glob path "/static/*" to Cloudflare expression "(http.request.uri.path wildcard "/static/*")"`
- `Unit: CloudflareTranslator translates AND(path, method) to compound Cloudflare expression`
- `Unit: CloudflareTranslator translates NOT(geo) to negated expression`
- `Unit: CloudflareTranslator translates cache directives to edge_ttl/browser_ttl parameters`
- `Integration (mocked API): CloudflareAdapter.validateCredentials with valid token → true`
- `Integration (mocked API): CloudflareAdapter.validateCredentials with expired token → false`
- `Integration (mocked API): CloudflareAdapter.purgeByUrl sends correct POST to /zones/:zoneId/purge_cache`
- `Fixture: cloudflare_translation_cases.json — 15 rule/expected-native pairs validated`

---

#### 3.4 — Fastly Adapter & Translator

**What**: Implement the Fastly provider adapter and the rule translator that converts abstract rules to Fastly VCL snippets and condition objects.

**Design**:

```typescript
// packages/api/src/translators/fastly.translator.ts
export class FastlyTranslator extends RuleTranslator {
  readonly providerSlug = 'fastly';

  translateCacheRule(rule: EdgeRule & { action: CacheAction }): TranslationResult {
    const condition = this.matchToFastlyCondition(rule.matchCriteria);
    const vclSnippet = this.generateCacheVcl(rule.action.directives, condition);

    return {
      nativeConfig: {
        condition: {
          type: 'REQUEST',
          statement: condition,
          priority: rule.priority,
        },
        cache_setting: {
          action: rule.action.cacheAction === 'bypass' ? 'pass' : 'cache',
          ttl: rule.action.directives.sMaxage ?? rule.action.directives.maxAge,
          stale_ttl: rule.action.directives.staleWhileRevalidate,
        },
        vcl_snippet: vclSnippet,
      },
      warnings: [],
      unsupportedFeatures: [],
    };
  }

  private matchToFastlyCondition(criteria: MatchCriteria): string {
    if ('path' in criteria) {
      return `req.url ~ "^${criteria.path.pattern.replace('*', '.*')}"`;
    }
    // ... other criteria
    return 'true';
  }

  private generateCacheVcl(directives: CacheDirectives, condition: string): string {
    const lines: string[] = [];
    lines.push(`if (${condition}) {`);
    if (directives.sMaxage) lines.push(`  set beresp.ttl = ${directives.sMaxage}s;`);
    if (directives.staleWhileRevalidate) lines.push(`  set beresp.stale_while_revalidate = ${directives.staleWhileRevalidate}s;`);
    if (directives.staleIfError) lines.push(`  set beresp.stale_if_error = ${directives.staleIfError}s;`);
    if (directives.surrogateKey) lines.push(`  set beresp.http.Surrogate-Key = "${directives.surrogateKey}";`);
    lines.push('}');
    return lines.join('\n');
  }
}
```

**Testing**:
- `Unit: FastlyTranslator converts glob path to VCL regex condition`
- `Unit: FastlyTranslator generates valid VCL snippet with TTL, stale-while-revalidate, and surrogate key`
- `Unit: FastlyTranslator sets cache_setting.action to "pass" for bypass rules`
- `Integration (mocked API): FastlyAdapter.validateCredentials with valid Fastly-Key → true`
- `Integration (mocked API): FastlyAdapter.purgeBySurrogateKey calls POST /service/:id/purge/:key`
- `Integration (mocked API): FastlyAdapter.fetchMetrics retrieves and normalizes stats response`
- `Fixture: fastly_translation_cases.json — 12 rule/expected-VCL pairs validated`

---

#### 3.5 — Provider Connection Management Routes

**What**: Implement CRUD and validation endpoints for CDN provider credentials.

**Design**:

```typescript
export const CreateProviderConnectionSchema = Type.Object({
  providerSlug: Type.String({ enum: ['cloudflare', 'fastly', 'akamai', 'aws_cloudfront', 'gcore', 'bunnycdn'] }),
  displayName: Type.String({ minLength: 1, maxLength: 255 }),
  authConfig: Type.Record(Type.String(), Type.Any()),
});
```

| Method | Path | Description |
|--------|------|-------------|
| POST | `.../:orgSlug/connections` | Add provider connection |
| GET | `.../:orgSlug/connections` | List connections |
| POST | `.../:orgSlug/connections/:connId/validate` | Validate credentials against provider API |
| DELETE | `.../:orgSlug/connections/:connId` | Remove connection |

**Testing**:
- `Integration: POST connection → 201; authConfig stored (vault refs, not raw secrets)`
- `Integration (mocked provider): POST validate → calls adapter.validateCredentials, returns { valid: true, capabilities: {...} }`
- `Integration: POST validate with bad credentials → returns { valid: false, error: "..." }`
- `Integration: DELETE connection → 204; referenced provider_rule_configs cleaned up`

---

## Phase 4: Provider Sync & Purge Operations

### Purpose

Implement the asynchronous machinery for syncing configurations to CDN providers and propagating cache purge requests. After this phase, activating a config version triggers background jobs that push the translated configuration to each connected provider, and users can issue purge requests that fan out to all active providers.

### Tasks

#### 4.1 — BullMQ Worker Infrastructure

**What**: Set up the BullMQ worker process with named queues for provider sync, purge propagation, and drift detection.

**Design**:

```typescript
// packages/worker/src/queues.ts
import { Queue, Worker, QueueEvents } from 'bullmq';
import { Redis } from 'ioredis';

const connection = new Redis(process.env.REDIS_URL!);

export const providerSyncQueue = new Queue('provider-sync', { connection });
export const purgeQueue = new Queue('purge-propagation', { connection });
export const driftCheckQueue = new Queue('drift-check', { connection });
export const metricsQueue = new Queue('metrics-collection', { connection });
export const certRenewalQueue = new Queue('cert-renewal', { connection });

// Rate limiting per provider to respect API quotas
export const PROVIDER_RATE_LIMITS: Record<string, { max: number; duration: number }> = {
  cloudflare: { max: 1200, duration: 300_000 },  // 1200 req/5min
  fastly: { max: 1000, duration: 60_000 },
  akamai: { max: 100, duration: 60_000 },
  aws_cloudfront: { max: 50, duration: 1_000 },
  gcore: { max: 500, duration: 60_000 },
  bunnycdn: { max: 500, duration: 60_000 },
};
```

**Testing**:
- `Integration: enqueue a job → worker picks it up and processes it`
- `Integration: failed job with retries → retried up to maxAttempts`
- `Integration: rate-limited job → delayed appropriately`
- `Unit: all 5 queues are registered and named correctly`

---

#### 4.2 — Provider Sync Job

**What**: Implement the job that pushes translated configuration to a CDN provider when a config version is activated.

**Design**:

```typescript
// packages/worker/src/jobs/provider-sync.job.ts
export interface ProviderSyncJobData {
  configVersionId: string;
  providerSlug: string;
  connectionId: string;
  ruleConfigs: Array<{
    edgeRuleId: string;
    nativeConfig: Record<string, unknown>;
  }>;
}

export async function processProviderSync(job: Job<ProviderSyncJobData>): Promise<void> {
  const { configVersionId, providerSlug, connectionId, ruleConfigs } = job.data;

  // 1. Update sync status to 'syncing'
  await updateSyncStatus(configVersionId, providerSlug, 'syncing');

  // 2. Get adapter and credentials
  const adapter = providerRegistry.get(providerSlug);
  const creds = await resolveCredentials(connectionId);

  // 3. Apply configuration
  try {
    const result = await adapter.applyConfig(creds, {
      rules: ruleConfigs.map(rc => rc.nativeConfig),
    });

    // 4. Update sync status and resource IDs
    await updateSyncStatus(configVersionId, providerSlug, 'synced', {
      providerResourceId: result.resourceId,
      providerVersion: result.providerVersion,
    });
  } catch (error) {
    await updateSyncStatus(configVersionId, providerSlug, 'failed', {
      errorMessage: error.message,
    });
    throw error; // trigger BullMQ retry
  }
}
```

Sync status table:

```typescript
export const providerSyncState = pgTable('provider_sync_state', {
  id: uuid('id').primaryKey().defaultRandom(),
  configVersionId: uuid('config_version_id').notNull().references(() => configVersions.id),
  providerSlug: varchar('provider_slug', { length: 50 }).notNull(),
  connectionId: uuid('connection_id').notNull().references(() => providerConnections.id),
  syncStatus: syncStatusEnum('sync_status').notNull().default('pending'),
  providerResourceId: varchar('provider_resource_id', { length: 500 }),
  providerVersionId: varchar('provider_version_id', { length: 255 }),
  lastSyncedAt: timestamp('last_synced_at', { withTimezone: true }),
  errorMessage: text('error_message'),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  uniqueVersionProvider: uniqueIndex('idx_sync_version_provider').on(table.configVersionId, table.providerSlug),
}));
```

**Testing**:
- `Integration (mocked provider): activate version → sync jobs enqueued for each connected provider`
- `Integration (mocked provider): sync job succeeds → sync_status set to 'synced', providerResourceId stored`
- `Integration (mocked provider): sync job fails → sync_status set to 'failed', error_message stored, job retried`
- `Integration (mocked provider): provider API rate limit hit → job delayed and retried`
- `Unit: sync job correctly resolves credentials from vault reference`

---

#### 4.3 — Purge Propagation

**What**: Implement cache purge request creation and fan-out to all connected providers.

**Design**:

```typescript
// packages/api/src/db/schema.ts (addition)
export const purgeOperations = pgTable('purge_operations', {
  id: uuid('id').primaryKey().defaultRandom(),
  propertyId: uuid('property_id').notNull().references(() => properties.id),
  initiatedBy: uuid('initiated_by').references(() => users.id),
  purgeType: purgeTypeEnum('purge_type').notNull(),
  purgeTarget: text('purge_target').notNull(),
  status: varchar('status', { length: 50 }).notNull().default('pending'),
  providerResults: jsonb('provider_results').notNull().default([]),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  completedAt: timestamp('completed_at', { withTimezone: true }),
});

// API route
export const PurgeRequestSchema = Type.Object({
  purgeType: Type.Union([
    Type.Literal('url'), Type.Literal('surrogate_key'),
    Type.Literal('prefix'), Type.Literal('all'),
  ]),
  targets: Type.Array(Type.String(), { minItems: 1 }),
});
```

| Method | Path | Description |
|--------|------|-------------|
| POST | `.../:propertySlug/purge` | Create purge request |
| GET | `.../:propertySlug/purge` | List recent purge operations |
| GET | `.../:propertySlug/purge/:purgeId` | Get purge status with per-provider results |

**Testing**:
- `Integration (mocked providers): POST purge by URL → purge jobs enqueued per provider; purge operation created with status 'pending'`
- `Integration (mocked providers): purge job completes → providerResults JSONB updated with propagation_ms`
- `Integration (mocked providers): all provider purges complete → overall status set to 'completed'`
- `Integration (mocked providers): one provider fails → overall status remains 'propagating', failed provider recorded`
- `Integration: POST purge by surrogate_key with provider that doesn't support it → PurgeResult with status 'not_supported'`
- `Integration: GET /purge/:id → returns full per-provider breakdown`

---

#### 4.4 — Drift Detection

**What**: Implement a scheduled job that compares the expected provider configuration against the actual remote state and flags drift.

**Design**:

```typescript
// packages/worker/src/jobs/drift-check.job.ts
export interface DriftCheckJobData {
  propertyId: string;
  providerSlug: string;
  connectionId: string;
  expectedConfig: Record<string, unknown>;
  providerResourceId: string;
}

export async function processDriftCheck(job: Job<DriftCheckJobData>): Promise<DriftResult> {
  const adapter = providerRegistry.get(job.data.providerSlug);
  const creds = await resolveCredentials(job.data.connectionId);

  const result = await adapter.detectDrift(creds, job.data.expectedConfig, job.data.providerResourceId);

  if (result.hasDrift) {
    await updateSyncStatus(job.data.propertyId, job.data.providerSlug, 'drifted', {
      driftDiff: result.diffs,
    });
    // Emit drift event for alerting
    await emitEvent('drift.detected', { propertyId: job.data.propertyId, provider: job.data.providerSlug, diffs: result.diffs });
  }

  return result;
}
```

Drift check scheduled via BullMQ repeatable job (every 15 minutes by default).

**Testing**:
- `Integration (mocked provider): remote config matches expected → sync_status remains 'synced'`
- `Integration (mocked provider): remote config differs → sync_status set to 'drifted', driftDiff populated`
- `Integration: drift detected → event emitted with diff details`
- `Unit: drift detection compares nested JSONB structures and reports path-level diffs`

---

## Phase 5: Traffic Steering & Multi-CDN Orchestration

### Purpose

Implement traffic policies for distributing requests across multiple CDN providers using weighted, geographic, latency-based, failover, and cost-optimized strategies. After this phase, users can define how traffic splits across their connected CDN providers with geographic overrides.

### Tasks

#### 5.1 — Traffic Policy CRUD

**What**: Implement traffic policy creation and management with strategy-specific configuration validation.

**Design**:

```typescript
// packages/api/src/db/schema.ts (addition)
export const trafficPolicies = pgTable('traffic_policies', {
  id: uuid('id').primaryKey().defaultRandom(),
  propertyId: uuid('property_id').notNull().references(() => properties.id, { onDelete: 'cascade' }),
  configVersionId: uuid('config_version_id').notNull().references(() => configVersions.id, { onDelete: 'cascade' }),
  name: varchar('name', { length: 255 }).notNull(),
  strategy: varchar('strategy', { length: 50 }).notNull().default('weighted'),
  isEnabled: boolean('is_enabled').notNull().default(true),
  strategyConfig: jsonb('strategy_config').notNull().default({}),
  targets: jsonb('targets').notNull(),
  geoOverrides: jsonb('geo_overrides').notNull().default([]),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});

export const CreateTrafficPolicySchema = Type.Object({
  name: Type.String({ minLength: 1 }),
  strategy: Type.Union([
    Type.Literal('weighted'), Type.Literal('geolocation'),
    Type.Literal('latency'), Type.Literal('failover'),
    Type.Literal('cost'), Type.Literal('ml_optimized'),
  ]),
  strategyConfig: Type.Optional(Type.Record(Type.String(), Type.Any())),
  targets: Type.Array(Type.Object({
    providerSlug: Type.String(),
    connectionId: Type.String({ format: 'uuid' }),
    weight: Type.Integer({ minimum: 0, maximum: 1000, default: 100 }),
    maxCostPerGbUsd: Type.Optional(Type.Number({ minimum: 0 })),
    enabled: Type.Boolean({ default: true }),
  }), { minItems: 1 }),
  geoOverrides: Type.Optional(Type.Array(Type.Object({
    countries: Type.Optional(Type.Array(Type.String({ minLength: 2, maxLength: 2 }))),
    continents: Type.Optional(Type.Array(Type.String({ minLength: 2, maxLength: 2 }))),
    providerSlug: Type.String(),
    weight: Type.Integer({ minimum: 0, maximum: 1000 }),
    fallback: Type.Optional(Type.String()),
  }))),
});
```

| Method | Path | Description |
|--------|------|-------------|
| POST | `.../:propertySlug/versions/:vn/traffic-policies` | Create traffic policy |
| GET | `.../:propertySlug/versions/:vn/traffic-policies` | List policies |
| GET | `.../:propertySlug/versions/:vn/traffic-policies/:policyId` | Get policy |
| PATCH | `.../:propertySlug/versions/:vn/traffic-policies/:policyId` | Update policy |
| DELETE | `.../:propertySlug/versions/:vn/traffic-policies/:policyId` | Delete policy |

**Testing**:
- `Integration: create weighted policy with 3 targets summing to 100% → 201`
- `Integration: create policy with geo overrides for JP, KR → overrides stored with ISO 3166-1 codes`
- `Integration: create policy referencing nonexistent connectionId → 400`
- `Integration: update target weights → new weights stored`
- `Integration: delete policy on activated version → 400 immutable`
- `Unit: weighted strategy validates that at least one target has weight > 0`
- `Unit: failover strategy validates that targets have distinct priorities`
- `Unit: geo override with invalid country code "XX" → validation error`

---

#### 5.2 — Traffic Decision Engine

**What**: Implement the server-side logic that evaluates traffic policies and returns the optimal provider for a given request context (used by DNS/proxy integrations and the ML training pipeline).

**Design**:

```typescript
// packages/api/src/services/traffic-decision.service.ts
export interface RequestContext {
  clientCountry: string;      // ISO 3166-1 alpha-2
  clientContinent?: string;
  clientAsn?: number;
  contentType?: string;
  requestPath?: string;
}

export interface TrafficDecision {
  selectedProvider: string;
  connectionId: string;
  reason: string;
  alternativeProviders: Array<{ provider: string; weight: number }>;
}

export class TrafficDecisionService {
  async decide(propertyId: string, context: RequestContext): Promise<TrafficDecision> {
    const policy = await this.getActivePolicy(propertyId);

    // 1. Check geo overrides first
    const geoMatch = this.findGeoOverride(policy.geoOverrides, context);
    if (geoMatch) {
      return {
        selectedProvider: geoMatch.providerSlug,
        connectionId: this.resolveConnectionId(geoMatch.providerSlug, policy.targets),
        reason: `geo_override:${context.clientCountry}`,
        alternativeProviders: geoMatch.fallback ? [{ provider: geoMatch.fallback, weight: 100 }] : [],
      };
    }

    // 2. Apply strategy
    switch (policy.strategy) {
      case 'weighted': return this.weightedSelection(policy.targets);
      case 'failover': return this.failoverSelection(policy.targets);
      case 'cost': return this.costOptimizedSelection(policy.targets, context);
      case 'latency': return this.latencyBasedSelection(policy.targets, context);
      case 'ml_optimized': return this.mlOptimizedSelection(policy.targets, context);
      default: return this.weightedSelection(policy.targets);
    }
  }

  private weightedSelection(targets: TrafficTarget[]): TrafficDecision;
  private failoverSelection(targets: TrafficTarget[]): TrafficDecision;
  private costOptimizedSelection(targets: TrafficTarget[], ctx: RequestContext): TrafficDecision;
  private latencyBasedSelection(targets: TrafficTarget[], ctx: RequestContext): TrafficDecision;
  private mlOptimizedSelection(targets: TrafficTarget[], ctx: RequestContext): TrafficDecision;
}
```

**Testing**:
- `Unit: weighted selection distributes requests proportionally to weights over 10000 iterations`
- `Unit: geo override for JP → returns JP-specific provider regardless of weights`
- `Unit: failover selection returns highest-priority enabled target`
- `Unit: cost selection returns target with lowest maxCostPerGbUsd`
- `Unit: all targets disabled → throws NoAvailableProviderError`
- `Integration: decide with real policy from DB → returns valid TrafficDecision`

---

## Phase 6: Certificate Lifecycle & TLS Management

### Purpose

Implement automated TLS certificate provisioning and renewal via ACME (RFC 8555) integration. After this phase, users can request certificates for their domains, and the platform automatically renews them before expiry.

### Tasks

#### 6.1 — Certificate Management CRUD & ACME Integration

**What**: Implement certificate request, storage, and ACME order lifecycle management.

**Design**:

```typescript
// packages/shared/src/types/certificate.ts
export interface Certificate {
  id: string;
  propertyId: string;
  domainNames: string[];
  issuer: 'letsencrypt' | 'zerossl' | 'custom';
  acmeState: AcmeState | null;
  certificateRef: string | null;  // vault ref
  privateKeyRef: string | null;   // vault ref
  issuedAt: string | null;
  expiresAt: string | null;
  autoRenew: boolean;
  renewalConfig: RenewalConfig;
  createdAt: string;
  updatedAt: string;
}

export interface AcmeState {
  orderUrl: string;
  status: 'pending' | 'ready' | 'processing' | 'valid' | 'invalid';
  challenges: Array<{
    type: 'http-01' | 'dns-01';
    status: string;
    token?: string;
    validatedAt?: string;
  }>;
  finalizeUrl?: string;
  certificateUrl?: string;
}

export interface RenewalConfig {
  daysBeforeExpiry: number;
  preferredChallenge: 'http-01' | 'dns-01';
  notifyEmails: string[];
}
```

| Method | Path | Description |
|--------|------|-------------|
| POST | `.../:propertySlug/certificates` | Request new certificate |
| GET | `.../:propertySlug/certificates` | List certificates |
| GET | `.../:propertySlug/certificates/:certId` | Get certificate details |
| POST | `.../:propertySlug/certificates/:certId/renew` | Force renewal |
| DELETE | `.../:propertySlug/certificates/:certId` | Remove certificate |

**Testing**:
- `Integration (mocked ACME): POST certificate → ACME order created, status 'pending'`
- `Integration (mocked ACME): ACME challenge completed → status transitions to 'valid', cert stored in vault ref`
- `Integration: GET certificate → returns expiry date and ACME state`
- `Unit: RenewalConfig defaults daysBeforeExpiry to 30`

---

#### 6.2 — Certificate Auto-Renewal Worker

**What**: Implement a scheduled job that checks for expiring certificates and triggers ACME renewal.

**Design**:

```typescript
// packages/worker/src/jobs/certificate-renewal.job.ts
export async function processCertRenewal(): Promise<void> {
  const expiringCerts = await db.select().from(certificates)
    .where(and(
      eq(certificates.autoRenew, true),
      lte(certificates.expiresAt,
        sql`now() + interval '1 day' * ${certificates.renewalConfig}->>'daysBeforeExpiry'`),
    ));

  for (const cert of expiringCerts) {
    await certRenewalQueue.add('renew', { certificateId: cert.id });
  }
}

// Scheduled as repeatable job: every 6 hours
certRenewalQueue.add('check-expiry', {}, {
  repeat: { pattern: '0 */6 * * *' },
});
```

**Testing**:
- `Integration (mocked ACME): certificate expiring in 25 days (threshold 30) → renewal job enqueued`
- `Integration (mocked ACME): certificate expiring in 35 days (threshold 30) → no renewal job`
- `Integration (mocked ACME): renewal completes → new cert stored, expiresAt updated`
- `Integration: certificate with autoRenew=false → skipped even if near expiry`

---

## Phase 7: Observability, Metrics & Cost Tracking

### Purpose

Implement the metrics collection pipeline, real-time analytics aggregation, and cost attribution across CDN providers. After this phase, users can view performance dashboards with per-provider, per-geography breakdowns and track CDN costs with predictive forecasting.

### Tasks

#### 7.1 — Metrics Collection Worker

**What**: Implement a scheduled job that fetches performance metrics from each connected CDN provider and stores them in the edge_metrics table.

**Design**:

```typescript
// packages/api/src/db/schema.ts (addition)
export const edgeMetrics = pgTable('edge_metrics', {
  id: uuid('id').primaryKey().defaultRandom(),
  propertyId: uuid('property_id').notNull().references(() => properties.id),
  providerSlug: varchar('provider_slug', { length: 50 }).notNull(),
  collectedAt: timestamp('collected_at', { withTimezone: true }).notNull(),
  granularity: varchar('granularity', { length: 20 }).notNull().default('1m'),
  metrics: jsonb('metrics').notNull(),
  geoBreakdown: jsonb('geo_breakdown'),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  propertyTimeIdx: index('idx_metrics_property_time').on(table.propertyId, table.collectedAt),
  providerTimeIdx: index('idx_metrics_provider_time').on(table.providerSlug, table.collectedAt),
}));
```

**Testing**:
- `Integration (mocked provider): metrics collection job → fetches from provider, stores normalized data`
- `Integration: GET /metrics?start=...&end=...&granularity=1h → returns aggregated metrics`
- `Unit: metrics normalization handles different provider response formats`

---

#### 7.2 — Cost Tracking & Attribution

**What**: Implement per-provider cost recording and cross-provider cost comparison.

**Design**:

```typescript
export const costTracking = pgTable('cost_tracking', {
  id: uuid('id').primaryKey().defaultRandom(),
  organizationId: uuid('organization_id').notNull().references(() => organizations.id),
  providerSlug: varchar('provider_slug', { length: 50 }).notNull(),
  propertyId: uuid('property_id').references(() => properties.id),
  periodStart: date('period_start').notNull(),
  periodEnd: date('period_end').notNull(),
  costDetails: jsonb('cost_details').notNull(),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
});

// API endpoints
// GET /:orgSlug/costs?start=...&end=...&groupBy=provider|property
// GET /:orgSlug/costs/forecast?months=3
```

**Testing**:
- `Integration: cost records created → GET /costs returns per-provider breakdown`
- `Integration: GET /costs?groupBy=property → returns cost per property per provider`
- `Unit: cost aggregation correctly sums line_items from JSONB costDetails`

---

#### 7.3 — Metrics & Cost API Routes

**What**: Implement the analytics API endpoints that power the dashboard.

**Design**:

| Method | Path | Description |
|--------|------|-------------|
| GET | `.../:propertySlug/metrics` | Get performance metrics (with time range, granularity, provider filters) |
| GET | `.../:propertySlug/metrics/cache-hit-ratio` | Cache hit ratio over time |
| GET | `.../:propertySlug/metrics/latency` | Latency percentiles over time |
| GET | `.../:orgSlug/costs` | Organization cost summary |
| GET | `.../:orgSlug/costs/by-property` | Cost breakdown by property |
| GET | `.../:orgSlug/costs/by-provider` | Cost breakdown by provider |

**Testing**:
- `Integration: GET metrics with time range → returns data points within range`
- `Integration: GET cache-hit-ratio → returns ratio per provider per time bucket`
- `Integration: GET costs/by-provider with date range → correct provider totals`

---

## Phase 8: AI-Powered Features

### Purpose

Implement the AI-native differentiators: cache rule recommender, natural-language configuration interface, anomaly detection, and predictive cost modeling. These features address the underserved areas identified in the competitive analysis.

### Tasks

#### 8.1 — Cache Rule Recommender

**What**: Analyse origin response headers, URL patterns, and cache-miss rates to auto-generate optimal cache-control rules.

**Design**:

```typescript
// packages/api/src/ai/cache-recommender.ts
export interface CacheRecommendation {
  ruleId?: string;           // existing rule being improved, or null for new rule
  name: string;
  matchCriteria: MatchCriteria;
  suggestedDirectives: CacheDirectives;
  rationale: string;
  estimatedHitRateImprovement: number;  // 0.0 to 1.0
  confidence: number;                    // 0.0 to 1.0
}

export class CacheRecommenderService {
  async generateRecommendations(propertyId: string): Promise<CacheRecommendation[]> {
    // 1. Fetch recent edge metrics (cache hits, misses by path pattern)
    const metrics = await this.fetchCacheMetrics(propertyId);

    // 2. Analyse origin response headers from recent cache misses
    const originPatterns = await this.analyseOriginHeaders(propertyId);

    // 3. Cluster URLs by response characteristics
    const clusters = this.clusterUrlPatterns(metrics, originPatterns);

    // 4. Generate rule recommendations per cluster
    const recommendations = clusters.map(cluster =>
      this.generateRuleForCluster(cluster)
    );

    // 5. Score and rank by estimated hit-rate improvement
    return recommendations
      .filter(r => r.confidence > 0.5)
      .sort((a, b) => b.estimatedHitRateImprovement - a.estimatedHitRateImprovement);
  }
}
```

| Method | Path | Description |
|--------|------|-------------|
| GET | `.../:propertySlug/ai/cache-recommendations` | Get AI-generated cache rule recommendations |
| POST | `.../:propertySlug/ai/cache-recommendations/:recId/apply` | Apply a recommendation as a new edge rule |

**Testing**:
- `Unit: URL pattern clustering groups "/static/img/*.png" and "/static/img/*.jpg" together`
- `Unit: recommendation for high-miss-rate static content suggests max_age > 86400`
- `Integration (mocked metrics): property with 60% cache hit rate → recommendations that could raise it to 85%+`
- `Integration: POST apply recommendation → creates new edge rule with recommended directives`

---

#### 8.2 — Natural-Language Configuration Interface

**What**: Translate plain-English rules into provider-specific edge configurations using an LLM.

**Design**:

```typescript
// packages/api/src/ai/nl-config.ts
export interface NlConfigRequest {
  instruction: string;  // e.g., "Cache images for 7 days except for logged-in users"
  propertyId: string;
}

export interface NlConfigResult {
  interpretedRule: EdgeRule;
  providerPreviews: Record<string, TranslationResult>;
  confidence: number;
  clarificationNeeded?: string;
}

export class NlConfigService {
  private readonly systemPrompt = `You are an expert CDN configuration assistant.
Given a plain-English rule description, output a JSON edge rule object with:
- ruleType (cache, header, waf, redirect, rewrite, rate_limit, geo_block)
- matchCriteria (using the MatchCriteria schema)
- action (using the appropriate action schema for the rule type)

Cache directives must align with RFC 9111 and RFC 9213.
Use ISO 3166-1 alpha-2 codes for geographic rules.
Always prefer s-maxage over max-age for CDN caching.`;

  async interpret(request: NlConfigRequest): Promise<NlConfigResult> {
    const response = await this.openai.chat.completions.create({
      model: 'gpt-4o',
      messages: [
        { role: 'system', content: this.systemPrompt },
        { role: 'user', content: request.instruction },
      ],
      response_format: { type: 'json_object' },
      temperature: 0.1,
    });

    const parsed = JSON.parse(response.choices[0].message.content!);
    const rule = this.validateAndNormalize(parsed);
    const previews = await this.previewTranslations(rule, request.propertyId);

    return { interpretedRule: rule, providerPreviews: previews, confidence: 0.9 };
  }
}
```

| Method | Path | Description |
|--------|------|-------------|
| POST | `.../:propertySlug/ai/interpret` | Interpret natural-language rule |
| POST | `.../:propertySlug/ai/interpret/:interpretationId/confirm` | Confirm and create rule |

**Testing**:
- `Integration (mocked LLM): "cache images for 7 days" → cache rule with path "/.*\\.(png|jpg|gif|webp|svg)" and max_age=604800`
- `Integration (mocked LLM): "block traffic from China and Russia" → geo_block rule with countries ["CN", "RU"]`
- `Integration (mocked LLM): "set X-Frame-Options to DENY on all responses" → header rule with correct action`
- `Unit: LLM output validated against EdgeRule JSON Schema before returning`
- `Unit: invalid LLM output triggers re-prompt with error details`

---

#### 8.3 — Anomaly Detection for Edge Configurations

**What**: ML-driven detection of misconfigurations, security gaps, and unusual traffic patterns.

**Design**:

```typescript
// packages/api/src/ai/anomaly-detector.ts
export interface AnomalyReport {
  anomalies: Anomaly[];
  overallRiskScore: number;  // 0.0 to 1.0
  generatedAt: string;
}

export interface Anomaly {
  type: 'misconfiguration' | 'security_gap' | 'performance' | 'cost';
  severity: 'critical' | 'high' | 'medium' | 'low';
  title: string;
  description: string;
  affectedResources: Array<{ type: string; id: string; name: string }>;
  recommendation: string;
}

export class AnomalyDetectorService {
  async scan(propertyId: string): Promise<AnomalyReport> {
    const rules = await this.getActiveRules(propertyId);
    const metrics = await this.getRecentMetrics(propertyId);
    const certs = await this.getCertificates(propertyId);

    const anomalies: Anomaly[] = [];

    // Security checks
    anomalies.push(...this.checkMissingSecurityHeaders(rules));
    anomalies.push(...this.checkBroadCacheBypasses(rules));
    anomalies.push(...this.checkExpiringCertificates(certs));
    anomalies.push(...this.checkTlsVersions(propertyId));

    // Performance checks
    anomalies.push(...this.checkLowCacheHitRate(metrics));
    anomalies.push(...this.checkHighErrorRate(metrics));

    // Cost checks
    anomalies.push(...this.checkCostSpikes(propertyId));

    const overallRiskScore = this.calculateRiskScore(anomalies);
    return { anomalies, overallRiskScore, generatedAt: new Date().toISOString() };
  }
}
```

| Method | Path | Description |
|--------|------|-------------|
| GET | `.../:propertySlug/ai/anomalies` | Run anomaly scan |
| GET | `.../:orgSlug/ai/anomalies/summary` | Organization-wide anomaly summary |

**Testing**:
- `Unit: property with no X-Frame-Options rule → 'security_gap' anomaly flagged`
- `Unit: certificate expiring in 5 days → 'critical' severity anomaly`
- `Unit: cache hit rate below 50% → 'performance' anomaly with recommendation`
- `Unit: bypass rule matching "/*" → 'misconfiguration' anomaly`
- `Integration: GET /ai/anomalies → returns sorted anomalies with risk score`

---

#### 8.4 — Predictive Cost Modeling

**What**: Forecast CDN spend across providers given planned traffic growth.

**Design**:

```typescript
// packages/api/src/ai/cost-predictor.ts
export interface CostForecast {
  forecasts: Array<{
    providerSlug: string;
    currentMonthly: number;
    projectedMonthly: number[];   // next N months
    confidenceInterval: { lower: number[]; upper: number[] };
    recommendations: string[];
  }>;
  totalCurrentMonthly: number;
  totalProjectedMonthly: number[];
}

export class CostPredictorService {
  async forecast(organizationId: string, months: number = 3): Promise<CostForecast> {
    const historicalCosts = await this.getHistoricalCosts(organizationId, 12);
    const trafficTrends = await this.getTrafficTrends(organizationId, 12);

    // Linear regression with seasonality adjustment per provider
    const forecasts = historicalCosts.map(providerCosts =>
      this.projectCosts(providerCosts, trafficTrends, months)
    );

    return {
      forecasts,
      totalCurrentMonthly: forecasts.reduce((s, f) => s + f.currentMonthly, 0),
      totalProjectedMonthly: this.sumArrays(forecasts.map(f => f.projectedMonthly)),
    };
  }
}
```

**Testing**:
- `Unit: linear cost growth of 10%/month → 3-month forecast within 5% accuracy`
- `Unit: cost forecast with insufficient historical data (< 3 months) → returns with wide confidence interval`
- `Integration: GET /costs/forecast?months=6 → returns per-provider projections`

---

## Phase 9: Web Dashboard

### Purpose

Build the Next.js dashboard for visual configuration management, analytics visualization, and operational workflows. After this phase, users can manage CDN configurations, view performance analytics, and operate traffic steering through a web interface.

### Tasks

#### 9.1 — Dashboard Layout & Authentication

**What**: Implement the dashboard shell with navigation, organization switcher, and authentication (JWT-based).

**Design**:

```typescript
// packages/web/src/app/layout.tsx
// Main layout with sidebar navigation:
// - Dashboard (overview)
// - Properties (list → detail → versions → rules)
// - Traffic (policies, geo map)
// - Analytics (metrics, cache hit rates, latency)
// - Certificates (list, status, renewal)
// - Cost (breakdown, forecast)
// - Settings (connections, members, API keys)

// Authentication via JWT tokens stored in httpOnly cookies
// Middleware checks auth on all routes except /login
```

**Testing**:
- `E2E (Playwright): login with valid credentials → redirected to dashboard`
- `E2E (Playwright): unauthenticated request → redirected to /login`
- `E2E (Playwright): organization switcher shows all user's organizations`

---

#### 9.2 — Property & Rule Management UI

**What**: Build the property detail page with version management, rule editor, and validation display.

**Design**:

The rule editor uses a form-based approach for simple rules and a JSON editor mode for advanced match criteria. The validation results panel shows errors and warnings inline with the affected rules.

Key pages:
- `/properties` — property list with search and filter by tags
- `/properties/:slug` — property detail with domains, origins, and version timeline
- `/properties/:slug/versions/:vn` — version detail with rule list
- `/properties/:slug/versions/:vn/rules/new` — rule creation wizard
- `/properties/:slug/versions/:vn/rules/:ruleId` — rule editor

**Testing**:
- `E2E (Playwright): create property → appears in property list`
- `E2E (Playwright): create cache rule with form → rule appears in version's rule list`
- `E2E (Playwright): validate version → errors shown inline with affected rules`
- `E2E (Playwright): activate version → confirmation dialog, then status changes to 'active'`

---

#### 9.3 — Analytics Dashboard

**What**: Build the analytics pages with charts for requests, cache hit rates, latency percentiles, and error rates, with per-provider and per-geography breakdowns.

**Design**:

Charts built with Recharts or similar. Time range selector (1h, 6h, 24h, 7d, 30d). Provider comparison overlays.

Key views:
- Overview: total requests, cache hit ratio, p50/p95 latency, error rate (sparklines)
- Performance: time-series charts for requests, bandwidth, latency by provider
- Cache: hit/miss ratio over time, top cache-miss paths
- Geographic: world map with latency heatmap per provider

**Testing**:
- `E2E (Playwright): analytics page loads with charts for time range 24h`
- `E2E (Playwright): switching time range updates chart data`
- `E2E (Playwright): provider filter shows/hides provider-specific data series`

---

#### 9.4 — Cost & Forecasting Dashboard

**What**: Build the cost overview page with per-provider breakdown, historical trends, and forecast visualization.

**Design**:

Key views:
- Cost overview: total spend this month vs. last month by provider (stacked bar)
- Trend: 12-month rolling cost by provider (line chart)
- Forecast: projected 3-month spend with confidence bands
- Breakdown: cost per property per provider (table)

**Testing**:
- `E2E (Playwright): cost page loads with per-provider breakdown`
- `E2E (Playwright): forecast chart shows projected values with confidence bands`

---

## Phase 10: Additional Provider Adapters

### Purpose

Extend provider coverage to Akamai, AWS CloudFront, Gcore, and Bunny.net. After this phase, the platform supports all six target CDN providers.

### Tasks

#### 10.1 — Akamai Adapter & Translator

**What**: Implement the Akamai adapter using EdgeGrid authentication and the Property Manager API, plus translator for Akamai rule tree behaviors/criteria.

**Design**:

Akamai requires special EdgeGrid HMAC-SHA-256 signing distinct from OAuth/bearer patterns. The translator maps abstract rules to Akamai's Property Manager behavior/criteria objects.

```typescript
// packages/api/src/providers/akamai.adapter.ts
export class AkamaiAdapter extends CdnProviderAdapter {
  readonly slug = 'akamai';
  readonly authMethod = 'edgegrid' as const;

  private signRequest(request: HttpRequest, creds: ProviderCredentials): HttpRequest {
    // EdgeGrid HMAC-SHA-256 signature per techdocs.akamai.com
  }
}
```

**Testing**:
- `Integration (mocked API): EdgeGrid signature generated correctly for known test vector`
- `Unit: Akamai translator maps cache rule to Property Manager behavior with correct options`
- `Unit: Akamai translator handles 5-level deep rule tree paths`

---

#### 10.2 — AWS CloudFront Adapter & Translator

**What**: Implement the CloudFront adapter using AWS SigV4 authentication and the CloudFront API.

**Design**:

Uses the AWS SDK for JavaScript v3 (`@aws-sdk/client-cloudfront`). Translator maps abstract rules to CloudFront cache policy and cache behavior objects.

**Testing**:
- `Integration (mocked API): CloudFront adapter creates distribution with correct cache behavior`
- `Unit: translator maps cache directives to CloudFront cache policy`

---

#### 10.3 — Gcore & Bunny.net Adapters

**What**: Implement adapters and translators for Gcore and Bunny.net.

**Design**:

Both use bearer-token/API-key authentication. Gcore uses REST/JSON API; Bunny.net uses its rebuilt OpenAPI-spec-driven API.

**Testing**:
- `Integration (mocked API): Gcore adapter validates credentials via API`
- `Integration (mocked API): Bunny.net adapter creates pull zone with correct cache settings`
- `Unit: translators generate correct native config for each provider`

---

## Phase 11: GitOps & Infrastructure-as-Code Integration

### Purpose

Enable declarative, version-controlled edge configuration through Terraform/HCL export, import from existing Terraform state, and drift detection against IaC-defined configurations. After this phase, users can manage their CDN configurations through GitOps workflows.

### Tasks

#### 11.1 — Terraform HCL Export

**What**: Generate Terraform HCL files from the platform's configuration that can be committed to a Git repository.

**Design**:

```typescript
// packages/api/src/services/terraform-export.service.ts
export class TerraformExportService {
  async exportVersion(versionId: string): Promise<string> {
    const version = await this.getVersionWithRules(versionId);
    const connections = await this.getConnections(version.propertyId);

    let hcl = '';
    for (const conn of connections) {
      hcl += this.generateProviderBlock(conn);
      hcl += this.generateResourceBlocks(version.rules, conn);
    }
    return hcl;
  }
}
```

| Method | Path | Description |
|--------|------|-------------|
| GET | `.../:propertySlug/versions/:vn/export/terraform` | Export as Terraform HCL |
| POST | `.../:propertySlug/import/terraform` | Import from Terraform state |

**Testing**:
- `Unit: export generates valid HCL for Cloudflare zone + ruleset resources`
- `Unit: export generates valid HCL for Fastly service + cache_setting resources`
- `Integration: exported HCL can be parsed by hcl2json without errors`
- `Integration: import from Terraform state creates matching edge rules in platform`

---

#### 11.2 — GitOps Webhook Integration

**What**: Accept webhooks from Git providers (GitHub, GitLab) to detect configuration changes in committed HCL files and trigger platform sync.

**Design**:

```typescript
export const GitWebhookSchema = Type.Object({
  repository: Type.Object({ full_name: Type.String() }),
  ref: Type.String(),
  commits: Type.Array(Type.Object({
    modified: Type.Array(Type.String()),
    added: Type.Array(Type.String()),
  })),
});
```

**Testing**:
- `Integration (mocked webhook): push with modified cdn.tf → config version created from HCL`
- `Integration: webhook with invalid signature → 401`

---

## Phase 12: Hardening, Performance & Production Readiness

### Purpose

Prepare the platform for production deployment with rate limiting, comprehensive error handling, performance optimization, monitoring, and deployment automation.

### Tasks

#### 12.1 — Rate Limiting & API Security

**What**: Implement request rate limiting, input sanitization, and API key authentication for programmatic access.

**Design**:

```typescript
// Rate limiting per organization
const RATE_LIMITS = {
  free: { max: 100, window: '1m' },
  pro: { max: 1000, window: '1m' },
  business: { max: 5000, window: '1m' },
  enterprise: { max: 20000, window: '1m' },
};
```

**Testing**:
- `Integration: exceed rate limit → 429 Too Many Requests with Retry-After header`
- `Integration: API key auth → accepted for programmatic access`
- `Unit: request body sanitization strips HTML/script tags from string inputs`

---

#### 12.2 — OpenTelemetry Instrumentation

**What**: Add distributed tracing across API, worker, and provider API calls with OpenTelemetry SDK.

**Design**:

Traces span from incoming API request through database queries, queue operations, and outgoing provider API calls. Trace context (traceparent) propagated per W3C Trace Context spec.

**Testing**:
- `Integration: API request → trace ID present in response header and audit log`
- `Integration: provider sync job → child spans for each provider API call`

---

#### 12.3 — Production Docker & CI/CD

**What**: Create production-optimized Docker images and CI/CD pipeline configuration.

**Design**:

Multi-stage Docker builds for minimal production images. GitHub Actions workflow for test, build, and deploy.

**Testing**:
- `E2E: docker compose -f docker-compose.prod.yml up → all services healthy`
- `E2E: health check endpoints respond within 1 second`
- `Integration: CI pipeline runs tests, builds, and produces Docker images`

---

#### 12.4 — Database Performance Optimization

**What**: Add database indexes, query optimization, connection pooling, and data retention policies for metrics and audit logs.

**Design**:

- TimescaleDB hypertable for `edge_metrics` (time-partitioned)
- Connection pooling via PgBouncer
- Data retention: metrics older than 90 days compressed, audit log retained for 1 year
- Materialized views for common dashboard queries

**Testing**:
- `Integration: metrics query over 30 days completes in < 500ms with 1M rows`
- `Integration: audit log query by organization + time range uses index scan`
- `Integration: data retention job compresses old metrics without data loss`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation & Data Model            ─── required by everything
    │
Phase 2: Config Versioning & Edge Rules     ─── requires Phase 1
    │
Phase 3: Provider Adapter Framework         ─── requires Phase 2
    │
    ├── Phase 4: Provider Sync & Purge      ─── requires Phase 3
    │       │
    │       ├── Phase 5: Traffic Steering   ─── requires Phase 4
    │       │
    │       └── Phase 6: Certificate Mgmt   ─── requires Phase 3 (can parallel Phase 5)
    │
    ├── Phase 7: Observability & Cost       ─── requires Phase 3 (can parallel Phases 4-6)
    │
    └── Phase 8: AI-Powered Features        ─── requires Phases 2, 7
         │
Phase 9: Web Dashboard                      ─── requires Phases 2-8 for full functionality;
    │                                            can start after Phase 2 with stub data
    │
Phase 10: Additional Provider Adapters      ─── requires Phase 3 (can parallel Phases 5-9)
    │
Phase 11: GitOps & IaC Integration          ─── requires Phases 2, 3
    │
Phase 12: Hardening & Production            ─── requires all prior phases
```

**Parallelism opportunities:**
- Phases 5, 6, and 7 can be developed concurrently after Phase 3/4
- Phase 9 (dashboard) can start after Phase 2 using mock data, then integrate real APIs as backend phases complete
- Phase 10 (additional adapters) can proceed in parallel with Phases 5-9 since the adapter interface is stable after Phase 3
- Phase 11 (GitOps) can proceed after Phase 3 in parallel with Phases 5-9

---

## Definition of Done (per phase)

1. All tasks implemented and code compiles with `strict: true` TypeScript.
2. All unit tests pass (`vitest run`).
3. All integration tests pass (with mocked external dependencies).
4. ESLint passes with zero errors (`eslint .`).
5. Prettier formatting applied (`prettier --check .`).
6. Docker build succeeds for all affected packages.
7. New API endpoints appear in auto-generated OpenAPI spec at `/api/docs`.
8. Database migrations created for any schema changes and apply/rollback cleanly.
9. New environment variables documented in `.env.example`.
10. Audit log captures all write operations introduced in the phase.
11. No secrets or credentials stored in source code or database (vault references only).
12. Feature works end-to-end via API (curl/Postman) or UI as appropriate.
