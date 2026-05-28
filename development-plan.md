# Email Deliverability Platform — Phased Development Plan

> Project: 130-email-deliverability-platform · Created: 2026-05-25
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language | TypeScript (Node.js) | API-heavy platform with dashboard frontend; TypeScript provides type safety across the full stack, strong ecosystem for email protocols (nodemailer, dns.promises), and shared types between API and frontend. |
| API framework | Fastify | Outperforms Express for webhook-heavy workloads (email event ingestion), native JSON schema validation for request/response, OpenAPI 3.1 generation via @fastify/swagger. |
| Database | PostgreSQL 16 | Required for JSONB columns (hybrid data model), GIN indexes, INET type for IP tracking, range partitioning for high-volume email_events table. Chosen data model: Hybrid Relational + JSONB (data-model-suggestion-3). |
| ORM / Query builder | Drizzle ORM | Type-safe SQL with full PostgreSQL feature support (JSONB, INET, generated columns). Lighter than Prisma, better JSONB support, native migration tooling. |
| Task queue | BullMQ (Redis-backed) | Warm-up scheduling, DNS checks, DMARC report parsing, and placement tests all require reliable async job processing with retries, rate limiting, and scheduled/recurring jobs. |
| Cache / Pub-Sub | Redis 7 | Shared infrastructure with BullMQ. Used for API rate limiting, session cache, real-time dashboard updates via pub/sub. |
| Frontend | Next.js 15 (App Router) | Dashboard-heavy application with server-side rendering for SEO on public pages, React Server Components for data-heavy dashboard views, shared TypeScript types with the API. |
| UI components | shadcn/ui + Tailwind CSS 4 | Accessible, composable components. Recharts for time-series reputation and placement charts. |
| DNS resolution | dns.promises (Node.js built-in) + dns2 | Built-in resolver handles SPF/DKIM/DMARC/BIMI TXT lookups. dns2 provides lower-level control for PTR records and custom resolver configuration. |
| Email protocol | nodemailer + imapflow | nodemailer for SMTP sending (warm-up engine), imapflow for IMAP mailbox monitoring (checking placement, reading replies). |
| Authentication | NextAuth.js v5 | OAuth2 support for Gmail/Outlook mailbox connection, JWT sessions, multi-tenant role-based access. |
| API authentication | API keys (SHA-256 hashed) + JWT | API keys for programmatic access (webhook ingestion, CLI), JWT for dashboard sessions. |
| Content analysis | SpamAssassin (via spamc protocol) + custom rules engine | SpamAssassin is the industry-standard open-source spam scorer. Custom rules engine for trigger word detection, link analysis, and HTML validation. |
| AI/ML | OpenAI API (GPT-4o) + custom time-series models | GPT-4o for content risk analysis and composition suggestions. Custom models (TensorFlow.js or Python microservice) for reputation forecasting and warm-up optimisation. |
| Containerisation | Docker + Docker Compose | Multi-service architecture (API, workers, frontend, Redis, PostgreSQL). Required for self-hosted deployment. |
| Testing | Vitest + Supertest + Playwright | Vitest for unit tests (fast, ESM-native), Supertest for API integration tests, Playwright for E2E dashboard tests. |
| Code quality | ESLint 9 + Prettier + TypeScript strict mode | Flat config ESLint, Prettier for formatting, strict TypeScript for type safety. |
| Package manager | pnpm | Workspace support for monorepo (api, web, shared packages), disk-efficient, strict dependency resolution. |
| Monorepo structure | pnpm workspaces + Turborepo | Shared types and utilities between API and frontend, parallel builds, dependency-aware task execution. |

---

### Project Structure

```
email-deliverability-platform/
├── package.json                          # Root workspace config
├── pnpm-workspace.yaml
├── turbo.json                            # Turborepo pipeline config
├── docker-compose.yml                    # PostgreSQL, Redis, API, workers, web
├── Dockerfile.api
├── Dockerfile.web
├── Dockerfile.worker
├── .env.example
├── packages/
│   └── shared/                           # Shared types, constants, utilities
│       ├── package.json
│       ├── tsconfig.json
│       └── src/
│           ├── types/
│           │   ├── tenant.ts
│           │   ├── domain.ts
│           │   ├── mailbox.ts
│           │   ├── auth-records.ts
│           │   ├── warmup.ts
│           │   ├── placement.ts
│           │   ├── reputation.ts
│           │   ├── email-events.ts
│           │   ├── alerts.ts
│           │   └── api.ts                # API request/response schemas
│           ├── constants/
│           │   ├── providers.ts          # ISP provider definitions
│           │   ├── bounce-classes.ts     # SendGrid 7-class taxonomy
│           │   ├── event-types.ts        # Email event type definitions
│           │   └── compliance.ts         # CAN-SPAM, GDPR, CCPA rules
│           └── validators/
│               ├── spf.ts
│               ├── dkim.ts
│               ├── dmarc.ts
│               └── bimi.ts
├── apps/
│   ├── api/                              # Fastify API server
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   └── src/
│   │       ├── server.ts                 # Fastify app bootstrap
│   │       ├── config.ts                 # Environment-based config
│   │       ├── db/
│   │       │   ├── schema.ts             # Drizzle schema (all tables)
│   │       │   ├── migrate.ts            # Migration runner
│   │       │   └── migrations/           # SQL migration files
│   │       ├── routes/
│   │       │   ├── auth.ts               # Login, register, API key management
│   │       │   ├── domains.ts            # Domain CRUD + verification
│   │       │   ├── mailboxes.ts          # Mailbox connection + management
│   │       │   ├── auth-records.ts       # SPF/DKIM/DMARC/BIMI checks
│   │       │   ├── warmup.ts             # Warm-up schedule management
│   │       │   ├── placement.ts          # Placement test triggers + results
│   │       │   ├── reputation.ts         # Reputation scores + history
│   │       │   ├── content.ts            # Content analysis
│   │       │   ├── dmarc-reports.ts      # DMARC report viewing
│   │       │   ├── alerts.ts             # Alert rule management
│   │       │   ├── webhooks.ts           # ESP webhook ingestion
│   │       │   └── health.ts             # Health check endpoint
│   │       ├── services/
│   │       │   ├── dns-checker.ts        # SPF/DKIM/DMARC/BIMI DNS lookups
│   │       │   ├── warmup-engine.ts      # Warm-up orchestration logic
│   │       │   ├── placement-tester.ts   # Inbox placement test execution
│   │       │   ├── reputation-scorer.ts  # Daily reputation computation
│   │       │   ├── content-analyzer.ts   # Spam content analysis
│   │       │   ├── dmarc-parser.ts       # DMARC aggregate report XML parser
│   │       │   ├── blacklist-checker.ts  # Blacklist/DNSBL lookups
│   │       │   ├── alert-evaluator.ts    # Alert rule evaluation engine
│   │       │   └── webhook-processor.ts  # ESP webhook normalisation
│   │       ├── middleware/
│   │       │   ├── auth.ts               # JWT + API key authentication
│   │       │   ├── tenant.ts             # Tenant context injection
│   │       │   └── rate-limit.ts         # Redis-based rate limiting
│   │       └── plugins/
│   │           ├── swagger.ts            # OpenAPI spec generation
│   │           └── redis.ts              # Redis client plugin
│   ├── web/                              # Next.js dashboard
│   │   ├── package.json
│   │   ├── tsconfig.json
│   │   ├── next.config.ts
│   │   └── src/
│   │       ├── app/
│   │       │   ├── layout.tsx
│   │       │   ├── page.tsx              # Landing / login
│   │       │   ├── (auth)/               # Login, register, forgot password
│   │       │   └── (dashboard)/          # Authenticated dashboard
│   │       │       ├── layout.tsx         # Sidebar + nav
│   │       │       ├── domains/          # Domain list, detail, auth health
│   │       │       ├── mailboxes/        # Mailbox connection + status
│   │       │       ├── warmup/           # Warm-up schedules + progress
│   │       │       ├── placement/        # Placement test results
│   │       │       ├── reputation/       # Reputation dashboard + trends
│   │       │       ├── content/          # Content analysis tool
│   │       │       ├── dmarc/            # DMARC report viewer
│   │       │       ├── alerts/           # Alert rules + history
│   │       │       └── settings/         # Tenant settings, API keys, users
│   │       ├── components/
│   │       │   ├── ui/                   # shadcn/ui components
│   │       │   ├── charts/              # Recharts wrappers
│   │       │   ├── domain-health-card.tsx
│   │       │   ├── reputation-chart.tsx
│   │       │   ├── placement-results.tsx
│   │       │   └── warmup-progress.tsx
│   │       └── lib/
│   │           ├── api-client.ts         # Typed API client
│   │           └── hooks/               # React Query hooks
│   └── worker/                           # BullMQ worker processes
│       ├── package.json
│       ├── tsconfig.json
│       └── src/
│           ├── index.ts                  # Worker bootstrap
│           ├── queues.ts                 # Queue definitions
│           └── jobs/
│               ├── dns-check.ts          # Scheduled DNS authentication checks
│               ├── warmup-cycle.ts       # Warm-up email send/receive cycle
│               ├── placement-collect.ts  # Collect placement test results
│               ├── reputation-compute.ts # Daily reputation score computation
│               ├── blacklist-scan.ts     # Scheduled blacklist scans
│               ├── dmarc-ingest.ts       # DMARC report email ingestion + parsing
│               ├── alert-evaluate.ts     # Periodic alert rule evaluation
│               └── webhook-process.ts    # Async webhook event processing
└── tests/
    ├── fixtures/
    │   ├── spf-records.json              # Sample SPF records for testing
    │   ├── dkim-records.json
    │   ├── dmarc-records.json
    │   ├── dmarc-report.xml              # Sample DMARC aggregate report
    │   ├── webhook-payloads/             # Sample ESP webhook payloads
    │   │   ├── sendgrid-bounce.json
    │   │   ├── sendgrid-delivered.json
    │   │   ├── mailgun-bounce.json
    │   │   └── postmark-bounce.json
    │   └── email-content/               # Sample emails for content analysis
    │       ├── clean-email.html
    │       ├── spam-trigger-email.html
    │       └── missing-unsubscribe.html
    └── e2e/
        ├── auth.spec.ts
        ├── domain-setup.spec.ts
        ├── warmup.spec.ts
        └── placement.spec.ts
```

---

## Phase 1: Foundation — Project Scaffolding, Database, and Authentication

### Purpose

Establish the monorepo structure, database schema, configuration management, and multi-tenant authentication system. After this phase, the platform can register tenants, authenticate users, and manage API keys. All subsequent phases build on this foundation.

### Tasks

#### 1.1 — Monorepo Scaffolding and Configuration

**What**: Initialise the pnpm workspace with Turborepo, configure TypeScript, ESLint, Prettier, and Docker Compose for local development.

**Design**:

Root `pnpm-workspace.yaml`:
```yaml
packages:
  - 'packages/*'
  - 'apps/*'
```

Root `turbo.json`:
```json
{
  "$schema": "https://turbo.build/schema.json",
  "globalDependencies": [".env"],
  "tasks": {
    "build": { "dependsOn": ["^build"], "outputs": ["dist/**"] },
    "dev": { "cache": false, "persistent": true },
    "test": { "dependsOn": ["^build"] },
    "lint": {},
    "typecheck": {}
  }
}
```

Environment configuration (`apps/api/src/config.ts`):
```typescript
import { z } from 'zod';

const envSchema = z.object({
  NODE_ENV: z.enum(['development', 'test', 'production']).default('development'),
  PORT: z.coerce.number().default(3001),
  DATABASE_URL: z.string().url(),
  REDIS_URL: z.string().url().default('redis://localhost:6379'),
  JWT_SECRET: z.string().min(32),
  JWT_EXPIRY: z.string().default('24h'),
  API_KEY_SALT: z.string().min(16),
  CORS_ORIGINS: z.string().default('http://localhost:3000'),
  LOG_LEVEL: z.enum(['debug', 'info', 'warn', 'error']).default('info'),
});

export type Env = z.infer<typeof envSchema>;
export const env = envSchema.parse(process.env);
```

Docker Compose (`docker-compose.yml`):
```yaml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: deliverability
      POSTGRES_USER: app
      POSTGRES_PASSWORD: devpassword
    ports: ["5432:5432"]
    volumes: ["pgdata:/var/lib/postgresql/data"]
  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]
    volumes: ["redisdata:/data"]
  api:
    build: { context: ., dockerfile: Dockerfile.api }
    depends_on: [postgres, redis]
    ports: ["3001:3001"]
    env_file: .env
  worker:
    build: { context: ., dockerfile: Dockerfile.worker }
    depends_on: [postgres, redis]
    env_file: .env
  web:
    build: { context: ., dockerfile: Dockerfile.web }
    depends_on: [api]
    ports: ["3000:3000"]
volumes:
  pgdata:
  redisdata:
```

**Testing**:
- `Unit: envSchema parses valid .env → Env object with correct types`
- `Unit: envSchema with missing DATABASE_URL → ZodError with path ["DATABASE_URL"]`
- `Unit: envSchema with invalid PORT (non-numeric) → ZodError`
- `Integration: pnpm build → all packages compile without errors`
- `Integration: docker-compose up → PostgreSQL and Redis accessible from API container`

---

#### 1.2 — Database Schema and Migrations

**What**: Implement the Hybrid Relational + JSONB schema from data-model-suggestion-3 using Drizzle ORM, with initial migration files.

**Design**:

Core Drizzle schema (`apps/api/src/db/schema.ts`) — key tables:

```typescript
import {
  pgTable, uuid, varchar, text, boolean, integer, decimal,
  timestamp, date, inet, jsonb, uniqueIndex, index, serial,
} from 'drizzle-orm/pg-core';

// --- Tenants ---
export const tenants = pgTable('tenants', {
  id: uuid('id').primaryKey().defaultRandom(),
  name: varchar('name', { length: 255 }).notNull(),
  slug: varchar('slug', { length: 100 }).notNull().unique(),
  planTier: varchar('plan_tier', { length: 50 }).notNull().default('free'),
  settings: jsonb('settings').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});

// --- Users ---
export const users = pgTable('users', {
  id: uuid('id').primaryKey().defaultRandom(),
  tenantId: uuid('tenant_id').notNull().references(() => tenants.id, { onDelete: 'cascade' }),
  email: varchar('email', { length: 320 }).notNull(),
  name: varchar('name', { length: 255 }),
  passwordHash: varchar('password_hash', { length: 255 }).notNull(),
  role: varchar('role', { length: 50 }).notNull().default('member'),
  profile: jsonb('profile').notNull().default({}),
  lastLoginAt: timestamp('last_login_at', { withTimezone: true }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (t) => [
  uniqueIndex('uq_users_tenant_email').on(t.tenantId, t.email),
  index('idx_users_tenant').on(t.tenantId),
]);

// --- API Keys ---
export const apiKeys = pgTable('api_keys', {
  id: uuid('id').primaryKey().defaultRandom(),
  tenantId: uuid('tenant_id').notNull().references(() => tenants.id, { onDelete: 'cascade' }),
  createdBy: uuid('created_by').notNull().references(() => users.id),
  keyHash: varchar('key_hash', { length: 255 }).notNull().unique(),
  name: varchar('name', { length: 255 }).notNull(),
  scopes: jsonb('scopes').notNull().default([]),
  expiresAt: timestamp('expires_at', { withTimezone: true }),
  lastUsedAt: timestamp('last_used_at', { withTimezone: true }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
}, (t) => [
  index('idx_api_keys_tenant').on(t.tenantId),
]);

// --- Domains ---
export const domains = pgTable('domains', {
  id: uuid('id').primaryKey().defaultRandom(),
  tenantId: uuid('tenant_id').notNull().references(() => tenants.id, { onDelete: 'cascade' }),
  domainName: varchar('domain_name', { length: 253 }).notNull(),
  status: varchar('status', { length: 50 }).notNull().default('pending_verification'),
  verifiedAt: timestamp('verified_at', { withTimezone: true }),
  authHealth: jsonb('auth_health').notNull().default({}),
  complianceStatus: jsonb('compliance_status').notNull().default({}),
  reputation: jsonb('reputation').notNull().default({}),
  settings: jsonb('settings').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (t) => [
  uniqueIndex('uq_domains_tenant_name').on(t.tenantId, t.domainName),
  index('idx_domains_tenant').on(t.tenantId),
  index('idx_domains_name').on(t.domainName),
]);

// --- Mailboxes ---
export const mailboxes = pgTable('mailboxes', {
  id: uuid('id').primaryKey().defaultRandom(),
  domainId: uuid('domain_id').notNull().references(() => domains.id, { onDelete: 'cascade' }),
  tenantId: uuid('tenant_id').notNull().references(() => tenants.id, { onDelete: 'cascade' }),
  emailAddress: varchar('email_address', { length: 320 }).notNull().unique(),
  provider: varchar('provider', { length: 50 }).notNull(),
  displayName: varchar('display_name', { length: 255 }),
  status: varchar('status', { length: 50 }).notNull().default('active'),
  connection: jsonb('connection').notNull().default({}),
  health: jsonb('health').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (t) => [
  index('idx_mailboxes_domain').on(t.domainId),
  index('idx_mailboxes_tenant').on(t.tenantId),
  index('idx_mailboxes_provider').on(t.provider),
]);

// Remaining tables (auth_records, dmarc_reports, warmup_schedules,
// warmup_daily_log, email_events, placement_tests, reputation_scores,
// content_analyses, alert_rules, alert_instances, blacklist_checks,
// warmup_peers) follow the same pattern from data-model-suggestion-3.
```

**Testing**:
- `Unit: Drizzle schema compiles with TypeScript strict mode → no type errors`
- `Integration: drizzle-kit push → all tables created in PostgreSQL with correct columns, types, and indexes`
- `Integration: insert tenant → UUID generated, timestamps set, settings default to {}`
- `Integration: insert user with duplicate (tenant_id, email) → unique constraint violation`
- `Integration: delete tenant → CASCADE deletes all users, api_keys, domains, mailboxes`
- `Integration: JSONB column accepts valid JSON object → stored and retrievable`
- `Integration: GIN index on domains.auth_health → EXPLAIN shows GIN index used for @> queries`

---

#### 1.3 — Multi-Tenant Authentication and Authorization

**What**: Implement user registration, login (JWT), API key management, and tenant-scoped middleware.

**Design**:

Shared types (`packages/shared/src/types/tenant.ts`):
```typescript
export interface Tenant {
  id: string;
  name: string;
  slug: string;
  planTier: 'free' | 'starter' | 'growth' | 'enterprise';
  settings: TenantSettings;
  createdAt: string;
  updatedAt: string;
}

export interface TenantSettings {
  maxDomains: number;
  maxMailboxes: number;
  timezone?: string;
  jurisdiction?: string;
  notificationPreferences?: {
    email: boolean;
    slackWebhook?: string;
    dailyDigest: boolean;
  };
}

export interface User {
  id: string;
  tenantId: string;
  email: string;
  name?: string;
  role: 'owner' | 'admin' | 'member' | 'viewer';
  lastLoginAt?: string;
  createdAt: string;
}

export interface ApiKey {
  id: string;
  tenantId: string;
  name: string;
  scopes: string[];
  expiresAt?: string;
  lastUsedAt?: string;
  createdAt: string;
}
```

Auth middleware (`apps/api/src/middleware/auth.ts`):
```typescript
import { FastifyRequest, FastifyReply } from 'fastify';
import { verifyJwt } from '../services/jwt';
import { verifyApiKey } from '../services/api-key';

export interface AuthContext {
  tenantId: string;
  userId: string;
  role: 'owner' | 'admin' | 'member' | 'viewer';
  scopes: string[];
  authMethod: 'jwt' | 'api_key';
}

declare module 'fastify' {
  interface FastifyRequest {
    auth: AuthContext;
  }
}

export async function authMiddleware(
  request: FastifyRequest,
  reply: FastifyReply
): Promise<void> {
  const authHeader = request.headers.authorization;
  if (!authHeader) {
    return reply.status(401).send({ error: 'Missing authorization header' });
  }

  if (authHeader.startsWith('Bearer ')) {
    const token = authHeader.slice(7);
    request.auth = await verifyJwt(token);
  } else if (authHeader.startsWith('ApiKey ')) {
    const key = authHeader.slice(7);
    request.auth = await verifyApiKey(key);
  } else {
    return reply.status(401).send({ error: 'Invalid authorization scheme' });
  }
}

export function requireRole(...roles: AuthContext['role'][]) {
  return async (request: FastifyRequest, reply: FastifyReply) => {
    if (!roles.includes(request.auth.role)) {
      return reply.status(403).send({ error: 'Insufficient permissions' });
    }
  };
}

export function requireScope(scope: string) {
  return async (request: FastifyRequest, reply: FastifyReply) => {
    if (request.auth.authMethod === 'api_key' && !request.auth.scopes.includes(scope)) {
      return reply.status(403).send({ error: `Missing scope: ${scope}` });
    }
  };
}
```

API routes (`apps/api/src/routes/auth.ts`):
```typescript
// POST /api/v1/auth/register
// Body: { tenantName: string, email: string, password: string, name?: string }
// Response: { tenant: Tenant, user: User, token: string }

// POST /api/v1/auth/login
// Body: { email: string, password: string }
// Response: { user: User, token: string }

// POST /api/v1/auth/api-keys
// Body: { name: string, scopes: string[], expiresAt?: string }
// Response: { apiKey: ApiKey, rawKey: string }  // rawKey only returned once

// GET /api/v1/auth/api-keys
// Response: { apiKeys: ApiKey[] }

// DELETE /api/v1/auth/api-keys/:id
// Response: { success: true }
```

Password hashing uses bcrypt with cost factor 12. API key generation produces a 32-byte random hex string; the SHA-256 hash is stored. JWT payload includes `{ sub: userId, tenantId, role }`.

**Testing**:
- `Unit: hashPassword("test123") → bcrypt hash, verifyPassword("test123", hash) → true`
- `Unit: verifyPassword("wrong", hash) → false`
- `Unit: generateApiKey() → 64-char hex string`
- `Unit: hashApiKey(key) → SHA-256 hex digest, deterministic`
- `Unit: signJwt({ sub, tenantId, role }) → valid JWT, decodes to same payload`
- `Unit: verifyJwt(expired token) → throws TokenExpiredError`
- `Integration: POST /api/v1/auth/register → 201, tenant created, user created with role=owner, JWT returned`
- `Integration: POST /api/v1/auth/register with duplicate email → 409 conflict`
- `Integration: POST /api/v1/auth/login with valid credentials → 200, JWT returned`
- `Integration: POST /api/v1/auth/login with wrong password → 401`
- `Integration: POST /api/v1/auth/api-keys → 201, key returned once, hash stored in DB`
- `Integration: GET /api/v1/domains with valid JWT → 200, returns tenant-scoped data`
- `Integration: GET /api/v1/domains with expired JWT → 401`
- `Integration: GET /api/v1/domains with ApiKey header → 200 if scope includes read:domains`
- `Integration: GET /api/v1/domains with ApiKey missing read:domains scope → 403`
- `Integration: authMiddleware with no Authorization header → 401`
- `Integration: requireRole('owner') with role=member → 403`

---

## Phase 2: Domain Management and DNS Authentication Checking

### Purpose

Enable users to register domains, verify ownership via DNS TXT records, and run SPF/DKIM/DMARC/BIMI validation checks. After this phase, the platform can monitor authentication health for registered domains — the core compliance requirement driven by Google/Yahoo/Microsoft enforcement.

### Tasks

#### 2.1 — Domain Registration and Verification

**What**: CRUD operations for domains with DNS-based ownership verification.

**Design**:

Domain verification flow:
1. User adds a domain (e.g., `acme.com`)
2. Platform generates a unique TXT verification token: `edp-verify=<uuid>`
3. User adds the TXT record to their DNS
4. Platform queries DNS for the TXT record and marks the domain as verified

API routes:
```typescript
// POST /api/v1/domains
// Body: { domainName: string }
// Response: { domain: Domain, verificationToken: string }

// POST /api/v1/domains/:id/verify
// Response: { domain: Domain }  // status changes to 'verified' or error

// GET /api/v1/domains
// Query: { status?: string, page?: number, limit?: number }
// Response: { domains: Domain[], total: number }

// GET /api/v1/domains/:id
// Response: { domain: Domain }

// DELETE /api/v1/domains/:id
// Response: { success: true }
```

Domain type (`packages/shared/src/types/domain.ts`):
```typescript
export interface Domain {
  id: string;
  tenantId: string;
  domainName: string;
  status: 'pending_verification' | 'verified' | 'suspended' | 'archived';
  verifiedAt?: string;
  authHealth: AuthHealth;
  complianceStatus: ComplianceStatus;
  reputation: ReputationSnapshot;
  settings: DomainSettings;
  createdAt: string;
  updatedAt: string;
}

export interface AuthHealth {
  spf?: { valid: boolean; lastChecked: string; qualifier: string; lookupCount: number };
  dkim?: { valid: boolean; lastChecked: string; selectors: string[]; keyLength: number };
  dmarc?: { valid: boolean; lastChecked: string; policy: string; adkim: string; aspf: string };
  bimi?: { valid: boolean; lastChecked: string; reason?: string };
  arc?: { supported: boolean };
  ptr?: { valid: boolean; lastChecked: string };
}

export interface ComplianceStatus {
  oneClickUnsubscribe?: { compliant: boolean; postUrl?: string; lastChecked: string };
  canSpam?: { physicalAddress: boolean; unsubscribeMechanism: boolean };
  gdpr?: { consentBasis: string; suppressionListActive: boolean };
  ccpa?: { optOutAvailable: boolean };
}

export interface ReputationSnapshot {
  overallScore?: number;
  inboxRate?: number;
  spamRate?: number;
  bounceRate?: number;
  complaintRate?: number;
  computedAt?: string;
  trend?: 'improving' | 'stable' | 'declining';
}

export interface DomainSettings {
  checkIntervalHours: number;
  alertThresholds: {
    reputationDrop: number;
    bounceRate: number;
    complaintRate: number;
  };
}
```

**Testing**:
- `Unit: generateVerificationToken(domainId) → "edp-verify=<valid-uuid>"`
- `Integration: POST /api/v1/domains { domainName: "acme.com" } → 201, domain with status=pending_verification, verificationToken returned`
- `Integration: POST /api/v1/domains with duplicate domain for same tenant → 409`
- `Integration: POST /api/v1/domains with invalid domain (no TLD) → 400`
- `Integration: POST /api/v1/domains/:id/verify when TXT record exists → 200, status=verified`
- `Integration: POST /api/v1/domains/:id/verify when TXT record missing → 400, status remains pending`
- `Integration: GET /api/v1/domains → returns only domains for authenticated tenant`
- `Integration: DELETE /api/v1/domains/:id → 200, domain deleted, associated mailboxes cascade-deleted`
- `Integration: tenant A cannot access tenant B's domains → 404`

---

#### 2.2 — DNS Authentication Checker Service

**What**: Service that performs SPF, DKIM, DMARC, BIMI, and PTR record validation against DNS, stores results in the unified `auth_records` table, and updates `domains.auth_health`.

**Design**:

DNS checker service (`apps/api/src/services/dns-checker.ts`):
```typescript
import { promises as dns } from 'node:dns';

export interface SpfCheckResult {
  recordType: 'spf';
  rawRecord: string;
  isValid: boolean;
  details: {
    version: string;
    mechanisms: string[];
    qualifier: string;   // '-all', '~all', '?all', '+all'
    dnsLookupCount: number;  // Must be <= 10 per RFC 7208
    errors: string[];
  };
}

export interface DkimCheckResult {
  recordType: 'dkim';
  rawRecord: string;
  isValid: boolean;
  details: {
    selector: string;
    keyType: 'rsa' | 'ed25519';
    keyLength: number;       // Must be >= 1024, recommended 2048 per RFC 6376
    canonicalisation: string;
    publicKeyHash: string;
    errors: string[];
  };
}

export interface DmarcCheckResult {
  recordType: 'dmarc';
  rawRecord: string;
  isValid: boolean;
  details: {
    version: string;
    policy: 'none' | 'quarantine' | 'reject';
    subdomainPolicy?: string;
    adkim: 'r' | 's';
    aspf: 'r' | 's';
    pct: number;
    rua: string[];
    ruf: string[];
    fo: string;
    // DMARCbis forward-compatibility (draft-ietf-dmarc-dmarcbis)
    np?: string;
    psd?: boolean;
    t?: boolean;
    errors: string[];
  };
}

export interface BimiCheckResult {
  recordType: 'bimi';
  rawRecord: string | null;
  isValid: boolean;
  details: {
    logoUrl?: string;
    certificateUrl?: string;
    certificateType?: 'vmc' | 'cmc';
    certificateValid?: boolean;
    dmarcReady: boolean;     // Requires p=quarantine or p=reject
    errors: string[];
  };
}

export type AuthCheckResult = SpfCheckResult | DkimCheckResult | DmarcCheckResult | BimiCheckResult;

export async function checkSpf(domain: string): Promise<SpfCheckResult>;
export async function checkDkim(domain: string, selectors: string[]): Promise<DkimCheckResult[]>;
export async function checkDmarc(domain: string): Promise<DmarcCheckResult>;
export async function checkBimi(domain: string, dmarcPolicy: string): Promise<BimiCheckResult>;
export async function runFullAuthCheck(domain: string): Promise<AuthCheckResult[]>;
```

SPF validation rules per RFC 7208:
- Record must start with `v=spf1`
- DNS lookup count must not exceed 10 (including includes, a, mx, ptr, exists)
- Must end with an `all` mechanism
- Warn if qualifier is `+all` (permits all senders)
- Warn if `~all` (softfail) rather than `-all` (hardfail)

DKIM validation rules per RFC 6376:
- Key length must be at least 1024 bits (warn if not 2048+)
- Selector record must exist at `<selector>._domainkey.<domain>`
- Key type must be `rsa` or `ed25519`

DMARC validation rules per RFC 7489:
- Record must exist at `_dmarc.<domain>`
- Must start with `v=DMARC1`
- Policy must be `none`, `quarantine`, or `reject`
- `rua` tag must contain valid `mailto:` URIs
- Warn if policy is `none` (not at enforcement)

BIMI validation rules:
- Record at `default._bimi.<domain>`
- DMARC policy must be at `quarantine` or `reject`
- Logo URL must point to SVG Tiny P/S format
- Certificate (VMC or CMC) chain must be valid

Worker job (`apps/worker/src/jobs/dns-check.ts`):
```typescript
// Scheduled job: runs every 6 hours (configurable per domain)
// 1. Fetch all verified domains due for a check
// 2. For each domain, run runFullAuthCheck()
// 3. Insert results into auth_records table
// 4. Update domains.auth_health JSONB with latest results
// 5. If any check changed from valid→invalid, evaluate alert rules
```

**Testing**:
- `Unit: parseSpfRecord("v=spf1 include:_spf.google.com -all") → { version: "spf1", mechanisms: ["include:_spf.google.com"], qualifier: "-all", dnsLookupCount: 1 }`
- `Unit: parseSpfRecord("v=spf1 +all") → valid with warning "Permits all senders"`
- `Unit: parseSpfRecord with 11 DNS lookups → isValid: false, errors: ["Exceeds 10 DNS lookup limit"]`
- `Unit: parseDmarcRecord("v=DMARC1; p=reject; rua=mailto:dmarc@acme.com") → policy: "reject", rua: ["mailto:dmarc@acme.com"]`
- `Unit: parseDmarcRecord with DMARCbis np tag → np field populated`
- `Unit: parseDmarcRecord missing v=DMARC1 → isValid: false`
- `Unit: parseDkimRecord with 1024-bit key → isValid: true with warning "2048-bit recommended"`
- `Unit: parseBimiRecord with DMARC policy=none → dmarcReady: false`
- `Integration (mocked DNS): checkSpf("example.com") → resolves TXT records, returns SpfCheckResult`
- `Integration (mocked DNS): checkDmarc for domain with no _dmarc record → isValid: false, error: "No DMARC record found"`
- `Integration: POST /api/v1/domains/:id/check-auth → triggers full auth check, returns results, auth_health updated`
- `Integration: scheduled dns-check job → processes due domains, inserts auth_records, updates auth_health`

---

#### 2.3 — Authentication Records API and History

**What**: API endpoints to retrieve current and historical authentication check results.

**Design**:

```typescript
// GET /api/v1/domains/:id/auth
// Response: { authHealth: AuthHealth, records: AuthRecord[] }

// GET /api/v1/domains/:id/auth/history
// Query: { recordType?: 'spf' | 'dkim' | 'dmarc' | 'bimi', from?: string, to?: string, limit?: number }
// Response: { records: AuthRecord[], total: number }

// POST /api/v1/domains/:id/auth/check
// Triggers an immediate auth check (rate-limited to 1/minute per domain)
// Response: { results: AuthCheckResult[] }

export interface AuthRecord {
  id: string;
  domainId: string;
  recordType: 'spf' | 'dkim' | 'dmarc' | 'bimi' | 'arc' | 'ptr';
  rawRecord: string | null;
  isValid: boolean;
  details: Record<string, unknown>;
  checkedAt: string;
  createdAt: string;
}
```

**Testing**:
- `Integration: GET /api/v1/domains/:id/auth → returns current auth_health and latest auth_records`
- `Integration: GET /api/v1/domains/:id/auth/history?recordType=dmarc → returns only DMARC records`
- `Integration: GET /api/v1/domains/:id/auth/history?from=2026-01-01&to=2026-05-01 → returns records in range`
- `Integration: POST /api/v1/domains/:id/auth/check → triggers check, returns fresh results`
- `Integration: POST /api/v1/domains/:id/auth/check twice within 1 minute → 429 rate limited`

---

## Phase 3: Mailbox Connection and Email Infrastructure

### Purpose

Enable users to connect email accounts (Gmail via OAuth2, Outlook via OAuth2, custom SMTP/IMAP) and verify connectivity. This is required before warm-up or placement testing can occur.

### Tasks

#### 3.1 — Mailbox Connection Service

**What**: Connect mailboxes via OAuth2 (Gmail, Outlook) or SMTP/IMAP credentials, test connectivity, and store encrypted connection details.

**Design**:

```typescript
export interface MailboxConnectionRequest {
  emailAddress: string;
  provider: 'gmail' | 'outlook' | 'yahoo' | 'custom_smtp';
  // For OAuth providers:
  oauthCode?: string;
  // For custom SMTP:
  smtpHost?: string;
  smtpPort?: number;
  imapHost?: string;
  imapPort?: number;
  username?: string;
  password?: string;  // Encrypted before storage
}

export interface MailboxConnectionTest {
  smtp: { success: boolean; error?: string; responseTime: number };
  imap: { success: boolean; error?: string; responseTime: number };
  sendTest: { success: boolean; error?: string };
}
```

API routes:
```typescript
// POST /api/v1/mailboxes
// Body: MailboxConnectionRequest
// Response: { mailbox: Mailbox, connectionTest: MailboxConnectionTest }

// POST /api/v1/mailboxes/:id/test
// Response: { connectionTest: MailboxConnectionTest }

// GET /api/v1/mailboxes
// Query: { domainId?: string, status?: string, page?: number }
// Response: { mailboxes: Mailbox[], total: number }

// PATCH /api/v1/mailboxes/:id
// Body: { displayName?: string, status?: 'active' | 'paused' }
// Response: { mailbox: Mailbox }

// DELETE /api/v1/mailboxes/:id
// Response: { success: true }
```

OAuth2 flow for Gmail:
1. Frontend redirects to Google OAuth consent screen with scopes: `gmail.send`, `gmail.readonly`
2. Google redirects back with authorization code
3. API exchanges code for refresh token via Google's token endpoint
4. Refresh token is AES-256-GCM encrypted before storage in `mailboxes.connection` JSONB
5. Access tokens are obtained on-demand from refresh token; never stored

Connection JSONB structure per provider:
```typescript
// Gmail / Outlook (OAuth2)
interface OAuthConnection {
  authMethod: 'oauth2';
  oauthTokenEnc: string;       // AES-256-GCM encrypted refresh token
  scopes: string[];
  tokenExpiresAt: string;
}

// Custom SMTP
interface SmtpConnection {
  authMethod: 'smtp_credentials';
  smtpHost: string;
  smtpPort: number;
  imapHost: string;
  imapPort: number;
  usernameEnc: string;         // AES-256-GCM encrypted
  passwordEnc: string;         // AES-256-GCM encrypted
}
```

**Testing**:
- `Unit: encryptCredential(plaintext, key) → ciphertext, decryptCredential(ciphertext, key) → plaintext`
- `Unit: encryptCredential with wrong key → DecryptionError`
- `Integration (mocked OAuth): POST /api/v1/mailboxes with Gmail oauthCode → exchanges for token, stores encrypted, 201`
- `Integration (mocked SMTP): POST /api/v1/mailboxes with custom SMTP → tests SMTP connection, 201`
- `Integration: POST /api/v1/mailboxes with invalid SMTP host → connectionTest.smtp.success=false, 400`
- `Integration: POST /api/v1/mailboxes/:id/test → re-tests connectivity, returns fresh results`
- `Integration: PATCH /api/v1/mailboxes/:id { status: "paused" } → status updated`
- `Integration: DELETE /api/v1/mailboxes/:id → mailbox removed`
- `Integration: plan limit enforcement → tenant on free plan with 10 mailboxes cannot add 11th → 403`

---

#### 3.2 — Email Sending and Receiving Abstraction

**What**: Unified interface for sending and receiving emails across all supported providers, used by the warm-up engine and placement tester.

**Design**:

```typescript
export interface EmailMessage {
  from: string;
  to: string;
  subject: string;
  textBody: string;
  htmlBody?: string;
  headers?: Record<string, string>;
  replyTo?: string;
}

export interface ReceivedEmail {
  messageId: string;
  from: string;
  to: string;
  subject: string;
  textBody: string;
  htmlBody?: string;
  headers: Record<string, string>;
  folder: 'inbox' | 'spam' | 'promotions' | 'social' | 'other';
  receivedAt: string;
}

export interface EmailTransport {
  send(mailboxId: string, message: EmailMessage): Promise<{ messageId: string }>;
  checkInbox(mailboxId: string, since: Date, folder?: string): Promise<ReceivedEmail[]>;
  moveToFolder(mailboxId: string, messageId: string, folder: string): Promise<void>;
  markAsImportant(mailboxId: string, messageId: string): Promise<void>;
  markAsNotSpam(mailboxId: string, messageId: string): Promise<void>;
}

// Implementations: GmailTransport, OutlookTransport, SmtpImapTransport
```

**Testing**:
- `Unit: GmailTransport.send() constructs correct MIME message with required headers`
- `Unit: SmtpImapTransport.send() connects to SMTP host with decrypted credentials`
- `Integration (mocked SMTP/IMAP): send email → message appears in IMAP mailbox`
- `Integration (mocked Gmail API): checkInbox returns emails with correct folder classification`
- `Integration (mocked): markAsNotSpam moves message from Spam to Inbox`

---

## Phase 4: Warm-Up Engine

### Purpose

Implement the email warm-up system that builds sender reputation by exchanging emails with a warm-up peer network, tracking daily metrics, and supporting both linear and adaptive warm-up strategies. This is one of the platform's core value propositions.

### Tasks

#### 4.1 — Warm-Up Schedule Management

**What**: CRUD for warm-up schedules with configurable strategy, daily targets, and ramp-up settings.

**Design**:

```typescript
export interface WarmupSchedule {
  id: string;
  mailboxId: string;
  status: 'active' | 'paused' | 'completed' | 'failed';
  config: WarmupConfig;
  aiState: WarmupAiState;
  startedAt: string;
  pausedAt?: string;
  completedAt?: string;
  createdAt: string;
  updatedAt: string;
}

export interface WarmupConfig {
  strategy: 'linear' | 'adaptive' | 'aggressive';
  dailyTarget: number;      // Current daily email target (e.g., 30)
  dailyMax: number;          // Maximum daily volume (e.g., 200)
  rampIncrement: number;     // Emails to add per day (e.g., 5)
  providerDistribution: Record<string, number>;  // e.g., { gmail: 0.5, outlook: 0.3 }
  warmTopics: string[];      // Topics for generated warm-up content
  language: string;          // ISO 639-1 code
  pauseOnSpamRate: number;   // Auto-pause if spam rate exceeds this (e.g., 0.10)
}

export interface WarmupAiState {
  modelVersion?: string;
  engagementMomentum?: number;   // 0.0 - 1.0
  recommendedDaily?: number;
  riskScore?: number;            // 0.0 - 1.0
  lastAdjustment?: string;
  adjustmentReason?: string;
}
```

API routes:
```typescript
// POST /api/v1/warmup/schedules
// Body: { mailboxId: string, config: Partial<WarmupConfig> }
// Response: { schedule: WarmupSchedule }

// GET /api/v1/warmup/schedules
// Query: { mailboxId?: string, status?: string }
// Response: { schedules: WarmupSchedule[] }

// PATCH /api/v1/warmup/schedules/:id
// Body: { status?: string, config?: Partial<WarmupConfig> }
// Response: { schedule: WarmupSchedule }

// GET /api/v1/warmup/schedules/:id/daily-log
// Query: { from?: string, to?: string }
// Response: { logs: WarmupDailyLog[] }
```

**Testing**:
- `Unit: default WarmupConfig → strategy=linear, dailyTarget=30, dailyMax=200, rampIncrement=5`
- `Integration: POST /api/v1/warmup/schedules → 201, schedule created with status=active`
- `Integration: POST /api/v1/warmup/schedules for mailbox already warming up → 409`
- `Integration: PATCH /api/v1/warmup/schedules/:id { status: "paused" } → pausedAt set`
- `Integration: GET /api/v1/warmup/schedules/:id/daily-log → returns chronological daily logs`

---

#### 4.2 — Warm-Up Peer Network

**What**: Manage the pool of seed inboxes that participate in warm-up exchanges (receive warm-up emails, reply, mark as important/not-spam).

**Design**:

```typescript
export interface WarmupPeer {
  id: string;
  emailAddress: string;
  provider: 'gmail' | 'outlook' | 'yahoo' | 'aol';
  region: string;        // ISO 3166-1 alpha-2
  isActive: boolean;
  stats: {
    totalInteractions: number;
    lastActiveAt: string;
    avgResponseTimeHours: number;
    inboxDeliveryRate: number;
  };
  createdAt: string;
}
```

Peer selection algorithm:
```
1. Filter active peers
2. Match target provider distribution from warmup config
3. Prefer peers with high inbox delivery rate (>0.95)
4. Distribute across regions to simulate natural sending patterns
5. Avoid sending to the same peer more than once per 24h from the same mailbox
6. Return N peers where N = dailyTarget for current day
```

**Testing**:
- `Unit: selectPeers(config, dailyTarget=30, excludeRecentIds) → 30 peers matching provider distribution`
- `Unit: selectPeers with provider distribution {gmail: 0.5} → ~15 Gmail peers out of 30`
- `Unit: selectPeers excludes peers used in last 24h`
- `Unit: selectPeers with insufficient active peers → returns available count, logs warning`
- `Integration: warm-up peer CRUD operations work correctly`

---

#### 4.3 — Warm-Up Execution Worker

**What**: BullMQ job that executes daily warm-up cycles: sends emails to peers, monitors for replies, records engagement metrics.

**Design**:

Warm-up cycle flow:
```
1. Scheduled job fires at configurable time (default: 8:00 AM tenant timezone)
2. For each active warm-up schedule:
   a. Calculate today's target volume (daily_target + ramp_increment if linear)
   b. Select peers via peer selection algorithm
   c. For each peer:
      - Generate warm-up email (subject + body from topic pool)
      - Send via EmailTransport
      - Record send event
   d. Wait 2-4 hours (configurable)
   e. Check peer inboxes for replies, mark-as-important, mark-as-not-spam actions
   f. Record engagement metrics in warmup_daily_log
   g. If spam rate > pauseOnSpamRate, auto-pause schedule
   h. Update warmup_schedules.config.dailyTarget for next day
   i. Update domains.auth_health and reputation JSONB
```

Daily log record:
```typescript
export interface WarmupDailyLog {
  id: string;
  scheduleId: string;
  logDate: string;         // YYYY-MM-DD
  emailsSent: number;
  repliesReceived: number;
  landedInbox: number;
  landedSpam: number;
  landedPromotions: number;
  providerBreakdown: Record<string, {
    sent: number;
    inbox: number;
    spam: number;
    promotions: number;
  }>;
  createdAt: string;
}
```

**Testing**:
- `Unit: calculateDailyTarget(schedule with linear strategy, day 5) → dailyTarget + (rampIncrement * 5), capped at dailyMax`
- `Unit: shouldAutoPause(spamRate=0.12, threshold=0.10) → true`
- `Unit: shouldAutoPause(spamRate=0.05, threshold=0.10) → false`
- `Unit: generateWarmupEmail(topics=["technology"], language="en") → EmailMessage with relevant subject and body`
- `Integration (mocked transport): warmup-cycle job → sends emails, records in daily log`
- `Integration (mocked transport): warmup-cycle with high spam rate → auto-pauses schedule`
- `Integration: daily log has UNIQUE(schedule_id, log_date) → duplicate date rejected`
- `Integration: warmup progress API returns cumulative metrics over time`

---

## Phase 5: Inbox Placement Testing

### Purpose

Enable users to test whether their emails land in Inbox, Spam, or Promotions across major ISPs by sending to a seed inbox network and collecting results. Combined with spam filter analysis.

### Tasks

#### 5.1 — Placement Test Execution

**What**: Send test emails to seed inboxes across multiple ISPs, collect placement results, and compute aggregate statistics.

**Design**:

```typescript
export interface PlacementTestRequest {
  domainId: string;
  mailboxId?: string;        // Which mailbox to send from
  subjectLine: string;
  htmlBody: string;
  textBody?: string;
}

export interface PlacementTestResult {
  id: string;
  domainId: string;
  tenantId: string;
  status: 'pending' | 'sending' | 'collecting' | 'completed' | 'failed';
  subjectLine: string;
  seedCount: number;
  results: {
    summary: {
      inboxRate: number;
      spamRate: number;
      promotionsRate: number;
      missingRate: number;
      avgDeliveryMs: number;
    };
    byProvider: Record<string, {
      seeds: number;
      inbox: number;
      spam: number;
      promotions: number;
      missing: number;
    }>;
    spamFilters: Record<string, {
      score: number;
      threshold: number;
      passed: boolean;
      rules: string[];
    }>;
    seedResults: Array<{
      email: string;
      provider: string;
      placement: 'inbox' | 'spam' | 'promotions' | 'social' | 'missing';
      deliveryMs: number;
      spamScore: number;
    }>;
  };
  startedAt?: string;
  completedAt?: string;
  createdAt: string;
}
```

API routes:
```typescript
// POST /api/v1/placement/tests
// Body: PlacementTestRequest
// Response: { test: PlacementTestResult }  // status=pending

// GET /api/v1/placement/tests
// Query: { domainId?: string, status?: string, page?: number }
// Response: { tests: PlacementTestResult[], total: number }

// GET /api/v1/placement/tests/:id
// Response: { test: PlacementTestResult }
```

Test execution flow (BullMQ job):
```
1. Create placement_tests row with status=pending
2. Select seed inboxes (70+ across Gmail, Outlook, Yahoo, AOL)
3. Send test email to all seeds → status=sending
4. Wait 5-15 minutes for delivery
5. Check each seed inbox for the test email → status=collecting
6. Record placement (inbox/spam/promotions/missing) for each seed
7. Run SpamAssassin scoring on the test email content
8. Aggregate results into summary, byProvider, spamFilters JSONB
9. Update placement_tests row → status=completed
```

**Testing**:
- `Unit: aggregatePlacementResults(seedResults) → correct inbox/spam/promotions/missing rates`
- `Unit: aggregateByProvider(seedResults) → grouped by provider with correct counts`
- `Unit: seeds with no response after 15 minutes → classified as "missing"`
- `Integration (mocked transport): POST /api/v1/placement/tests → test created, job enqueued`
- `Integration (mocked transport): placement-collect job → collects results from seed inboxes, computes summary`
- `Integration: GET /api/v1/placement/tests/:id after completion → full results with byProvider breakdown`
- `Integration: placement test with 0 seeds available → status=failed, error message`

---

#### 5.2 — Spam Content Analysis

**What**: Analyse email content for spam trigger words, formatting patterns, link reputation, and HTML issues. Returns a risk score and actionable recommendations.

**Design**:

```typescript
export interface ContentAnalysis {
  id: string;
  tenantId: string;
  bodyHash: string;
  overallRisk: 'low' | 'medium' | 'high' | 'critical';
  spamScore: number;
  analysis: {
    triggerWords: string[];
    triggerWordCount: number;
    linkCount: number;
    linkReputation: { safe: number; suspicious: number; blocked: number };
    imageCount: number;
    imageToTextRatio: number;
    hasTrackingPixel: boolean;
    htmlValid: boolean;
    htmlIssues: string[];
    subjectLineRisk: 'low' | 'medium' | 'high';
    recommendations: string[];
  };
  createdAt: string;
}

// Spam trigger word categories (rule-based)
export const TRIGGER_CATEGORIES = {
  urgency: ['act now', 'limited time', 'expires', 'urgent', 'immediately'],
  financial: ['free', 'discount', 'save', 'cash', 'profit', 'earn', 'income'],
  pressure: ['don\'t miss', 'last chance', 'final', 'only today'],
  suspicious: ['click here', 'click below', 'open immediately', 'verify your account'],
  formatting: ['ALL CAPS subjects', 'excessive exclamation marks', 'red text'],
} as const;
```

API route:
```typescript
// POST /api/v1/content/analyze
// Body: { subjectLine: string, htmlBody: string, textBody?: string }
// Response: { analysis: ContentAnalysis }
```

**Testing**:
- `Unit: analyzeContent("FREE MONEY!!!") → overallRisk: "critical", triggerWords include "free"`
- `Unit: analyzeContent(clean business email) → overallRisk: "low", no trigger words`
- `Unit: analyzeContent with 10 links → recommendation "Reduce link count"`
- `Unit: analyzeContent with tracking pixel → hasTrackingPixel: true`
- `Unit: analyzeContent with invalid HTML → htmlValid: false, htmlIssues populated`
- `Unit: analyzeContent with image-heavy email → imageToTextRatio > 0.5, recommendation`
- `Integration: POST /api/v1/content/analyze → 200, ContentAnalysis returned with recommendations`
- `Fixture: tests/fixtures/email-content/spam-trigger-email.html → high risk`
- `Fixture: tests/fixtures/email-content/clean-email.html → low risk`

---

## Phase 6: Reputation Scoring and Blacklist Monitoring

### Purpose

Compute daily domain reputation scores from email event data, track IP and domain blacklist status, and provide trend analysis. This enables the dashboard's core health monitoring view.

### Tasks

#### 6.1 — Email Event Ingestion (ESP Webhooks)

**What**: Receive and normalise webhook events from SendGrid, Mailgun, Postmark, and Amazon SES into the unified `email_events` table.

**Design**:

```typescript
// Webhook endpoints (unauthenticated but signature-verified)
// POST /api/v1/webhooks/sendgrid
// POST /api/v1/webhooks/mailgun
// POST /api/v1/webhooks/postmark
// POST /api/v1/webhooks/ses

export interface NormalisedEmailEvent {
  tenantId: string;
  domainId: string;
  mailboxId?: string;
  eventType: 'delivered' | 'bounced' | 'deferred' | 'dropped' | 'open' | 'click' | 'spam_report' | 'unsubscribe';
  eventCategory: 'delivery' | 'engagement';
  recipientEmail: string;
  provider?: string;
  occurredAt: string;
  details: Record<string, unknown>;
  rawPayload: Record<string, unknown>;
  externalEventId?: string;
  externalMessageId?: string;
}

// Webhook signature verification per provider
export function verifySendGridSignature(payload: Buffer, signature: string, timestamp: string): boolean;
export function verifyMailgunSignature(payload: Buffer, signature: string, timestamp: string): boolean;
export function verifyPostmarkSignature(payload: Buffer, signature: string): boolean;
```

**Testing**:
- `Unit: normaliseSendGridEvent(sendgridBouncePayload) → NormalisedEmailEvent with eventType="bounced", correct bounce classification`
- `Unit: normaliseMailgunEvent(mailgunDeliveryPayload) → NormalisedEmailEvent with eventType="delivered"`
- `Unit: normalisePostmarkEvent(postmarkSpamPayload) → NormalisedEmailEvent with eventType="spam_report"`
- `Unit: verifySendGridSignature(valid) → true`
- `Unit: verifySendGridSignature(tampered) → false`
- `Integration: POST /api/v1/webhooks/sendgrid with valid signature → 200, event stored`
- `Integration: POST /api/v1/webhooks/sendgrid with invalid signature → 401`
- `Integration: POST /api/v1/webhooks/sendgrid with duplicate externalEventId → 200, no duplicate stored (idempotent)`
- `Fixture: tests/fixtures/webhook-payloads/sendgrid-bounce.json → correct normalisation`

---

#### 6.2 — Daily Reputation Score Computation

**What**: Worker job that computes daily domain reputation scores by aggregating email events, placement test results, and blacklist status.

**Design**:

```typescript
export interface ReputationScore {
  id: string;
  domainId: string;
  scoreDate: string;       // YYYY-MM-DD
  overallScore: number;    // 0.00 - 100.00
  inboxRate: number;
  bounceRate: number;
  complaintRate: number;   // Must stay below 0.003 (0.3%) per Google requirements
  emailsSent: number;
  breakdown: {
    spamRate: number;
    openRate: number;
    clickRate: number;
    unsubscribeRate: number;
    byProvider: Record<string, {
      inboxRate: number;
      spamRate: number;
      volume: number;
    }>;
    blacklists: string[];
    ipScores: Record<string, { score: number; blacklists: string[] }>;
  };
}
```

Scoring algorithm:
```
overallScore = (
  inboxRate * 0.35 +
  (1 - bounceRate) * 100 * 0.20 +
  (1 - complaintRate / 0.003) * 100 * 0.20 +  // Normalised against Google's 0.3% threshold
  authScore * 0.15 +       // SPF + DKIM + DMARC all passing = 100
  engagementScore * 0.10   // (openRate + clickRate) normalised
)

Clamp to [0, 100].
Trend = "improving" if score > 7-day moving average, "declining" if < 7-day MA, else "stable".
```

Worker job (`apps/worker/src/jobs/reputation-compute.ts`):
```
Runs daily at 02:00 UTC:
1. For each domain with email activity in the past 24h:
   a. Query email_events for delivered, bounced, spam_report, open, click counts
   b. Calculate rates
   c. Get latest auth check results
   d. Get latest blacklist status
   e. Compute overall score
   f. Insert reputation_scores row
   g. Update domains.reputation JSONB snapshot
```

**Testing**:
- `Unit: computeReputationScore({ inboxRate: 95, bounceRate: 0.02, complaintRate: 0.001, authPassing: true, openRate: 25 }) → score > 85`
- `Unit: computeReputationScore({ complaintRate: 0.005 }) → low score, complaint penalty applied`
- `Unit: computeReputationScore({ bounceRate: 0.15 }) → score < 60`
- `Unit: determineTrend(scores=[80, 82, 85, 87]) → "improving"`
- `Unit: determineTrend(scores=[90, 88, 85, 83]) → "declining"`
- `Unit: determineTrend(scores=[85, 86, 85, 84]) → "stable"`
- `Integration: reputation-compute job → processes domains, inserts scores, updates domain.reputation`
- `Integration: GET /api/v1/domains/:id/reputation → returns current score and history`
- `Integration: GET /api/v1/domains/:id/reputation/history?from=2026-01-01 → time-series data`

---

#### 6.3 — Blacklist Monitoring

**What**: Periodic checks of domain and IP addresses against major DNS-based blacklists (DNSBLs).

**Design**:

```typescript
const BLACKLISTS = [
  'zen.spamhaus.org',           // Spamhaus SBL/XBL/PBL
  'b.barracudacentral.org',     // Barracuda BRBL
  'bl.spamcop.net',             // SpamCop
  'dnsbl.sorbs.net',            // SORBS
  'psbl.surriel.com',           // PSBL
  'dnsbl-1.uceprotect.net',     // UCE Protect Level 1
  'cbl.abuseat.org',            // Composite Blocking List
  'dnsbl.invaluement.com',      // Invaluement
] as const;

export interface BlacklistCheckResult {
  id: string;
  domainId?: string;
  ipAddress?: string;
  checkedAt: string;
  results: {
    totalChecked: number;
    listedOn: Array<{
      name: string;
      reason?: string;
      listedSince?: string;
    }>;
    cleanOn: string[];
  };
  isListed: boolean;  // Generated column
}
```

DNSBL lookup mechanism:
```
For IP 203.0.113.5 checking zen.spamhaus.org:
  1. Reverse the IP octets: 5.113.0.203
  2. Append blacklist domain: 5.113.0.203.zen.spamhaus.org
  3. DNS A record lookup
  4. If A record exists → listed (127.0.0.x codes indicate listing type)
  5. If NXDOMAIN → not listed
```

**Testing**:
- `Unit: reverseIp("203.0.113.5") → "5.113.0.203"`
- `Unit: buildDnsblQuery("203.0.113.5", "zen.spamhaus.org") → "5.113.0.203.zen.spamhaus.org"`
- `Integration (mocked DNS): checkBlacklists("203.0.113.5") → checks all lists, returns combined result`
- `Integration (mocked DNS): IP listed on Spamhaus → isListed=true, listedOn includes Spamhaus`
- `Integration: scheduled blacklist-scan job → processes all domains/IPs, stores results`
- `Integration: GET /api/v1/domains/:id/blacklists → returns latest check with listed/clean breakdown`

---

## Phase 7: DMARC Report Ingestion and Analysis

### Purpose

Ingest DMARC aggregate reports (RUA) received via email, parse the XML, store in the hybrid JSONB model, and surface unauthorised senders and authentication failures in the dashboard.

### Tasks

#### 7.1 — DMARC Report Email Ingestion

**What**: Monitor a designated IMAP mailbox for incoming DMARC aggregate reports, extract the compressed XML attachments, and queue for parsing.

**Design**:

DMARC aggregate report flow:
```
1. Domain owner sets rua=mailto:dmarc-reports@edp.example.com in DMARC record
2. ISPs (Google, Yahoo, Microsoft) send aggregate reports to this address
3. Reports arrive as email attachments (gzip or zip compressed XML)
4. Worker monitors IMAP inbox, extracts attachments
5. Decompresses and parses XML per RFC 7489 schema
6. Stores in dmarc_reports table with rows as JSONB array
```

```typescript
export interface DmarcAggregateReport {
  id: string;
  domainId: string;
  reportId: string;
  reportingOrg: string;
  dateRangeBegin: string;
  dateRangeEnd: string;
  policyPublished: {
    domain: string;
    p: string;
    sp?: string;
    adkim: string;
    aspf: string;
    pct: number;
  };
  rows: Array<{
    sourceIp: string;
    count: number;
    disposition: 'none' | 'quarantine' | 'reject';
    dkim: { domain: string; result: 'pass' | 'fail'; selector?: string };
    spf: { domain: string; result: 'pass' | 'fail'; scope: string };
    envelopeFrom: string;
    headerFrom: string;
  }>;
  summary: {
    totalMessages: number;
    passCount: number;
    quarantineCount: number;
    rejectCount: number;
    uniqueSourceIps: number;
    unauthorisedIps: string[];
  };
  rawXml?: string;
  ingestedAt: string;
}
```

API routes:
```typescript
// GET /api/v1/domains/:id/dmarc/reports
// Query: { from?: string, to?: string, reportingOrg?: string, page?: number }
// Response: { reports: DmarcAggregateReport[], total: number }

// GET /api/v1/domains/:id/dmarc/reports/:reportId
// Response: { report: DmarcAggregateReport }

// GET /api/v1/domains/:id/dmarc/unauthorised-senders
// Response: { senders: Array<{ sourceIp, firstSeen, lastSeen, totalMessages, dkimResult, spfResult }> }

// GET /api/v1/domains/:id/dmarc/summary
// Query: { from?: string, to?: string }
// Response: { totalMessages, passRate, failRate, topReportingOrgs, topUnauthorisedIps }
```

**Testing**:
- `Unit: parseDmarcXml(sampleXml) → DmarcAggregateReport with correct fields`
- `Unit: parseDmarcXml with malformed XML → throws DmarcParseError`
- `Unit: computeSummary(rows) → correct passCount, uniqueSourceIps, unauthorisedIps`
- `Unit: decompressAttachment(gzipBuffer) → XML string`
- `Unit: decompressAttachment(zipBuffer) → XML string`
- `Fixture: tests/fixtures/dmarc-report.xml → parses to expected structure`
- `Integration (mocked IMAP): dmarc-ingest job → reads email, extracts XML, stores report`
- `Integration: duplicate report_id + reporting_org → upserted, not duplicated`
- `Integration: GET /api/v1/domains/:id/dmarc/unauthorised-senders → aggregates across reports`

---

## Phase 8: Alerting System

### Purpose

Enable users to configure alert rules (reputation drops, blacklist listings, auth failures, bounce rate thresholds) and receive notifications via email, Slack webhook, or in-app.

### Tasks

#### 8.1 — Alert Rule Management

**What**: CRUD for alert rules with configurable conditions, thresholds, notification channels, and cooldown periods.

**Design**:

```typescript
export interface AlertRule {
  id: string;
  tenantId: string;
  domainId?: string;    // null = applies to all domains
  name: string;
  isActive: boolean;
  config: {
    condition: 'reputation_drop' | 'blacklist_added' | 'auth_failure' |
               'bounce_rate_threshold' | 'complaint_rate_threshold' |
               'spam_rate_threshold' | 'warmup_paused';
    threshold: number;
    windowHours: number;    // Time window for evaluation (e.g., 24)
    severity: 'info' | 'warning' | 'critical';
    channels: ('email' | 'slack' | 'webhook')[];
    slackWebhook?: string;
    webhookUrl?: string;
    cooldownHours: number;  // Minimum hours between alerts for same condition
  };
  createdAt: string;
  updatedAt: string;
}

export interface AlertInstance {
  id: string;
  ruleId: string;
  tenantId: string;
  domainId?: string;
  severity: 'info' | 'warning' | 'critical';
  message: string;
  context: {
    previousScore?: number;
    currentScore?: number;
    dropAmount?: number;
    triggerEvents?: string[];
    recommendedActions?: string[];
  };
  acknowledged: boolean;
  acknowledgedBy?: string;
  acknowledgedAt?: string;
  firedAt: string;
}
```

API routes:
```typescript
// POST /api/v1/alerts/rules
// GET /api/v1/alerts/rules
// PATCH /api/v1/alerts/rules/:id
// DELETE /api/v1/alerts/rules/:id

// GET /api/v1/alerts
// Query: { acknowledged?: boolean, severity?: string, domainId?: string }
// Response: { alerts: AlertInstance[], total: number }

// PATCH /api/v1/alerts/:id/acknowledge
// Response: { alert: AlertInstance }
```

**Testing**:
- `Integration: POST /api/v1/alerts/rules → 201, rule created`
- `Integration: PATCH /api/v1/alerts/rules/:id { isActive: false } → rule deactivated`
- `Integration: GET /api/v1/alerts?acknowledged=false → returns only unacknowledged alerts`
- `Integration: PATCH /api/v1/alerts/:id/acknowledge → acknowledged=true, acknowledgedAt set`

---

#### 8.2 — Alert Evaluation Engine

**What**: Worker job that periodically evaluates alert rules against current data and fires alerts when conditions are met.

**Design**:

Alert evaluation flow:
```
Runs every 15 minutes:
1. Fetch all active alert rules
2. For each rule:
   a. Check cooldown: skip if alert fired within cooldownHours
   b. Evaluate condition against current data:
      - reputation_drop: compare latest score vs score from windowHours ago
      - blacklist_added: check if any new blacklist listing since last evaluation
      - auth_failure: check if SPF/DKIM/DMARC changed from valid to invalid
      - bounce_rate_threshold: compute bounce rate over windowHours
      - complaint_rate_threshold: compute complaint rate over windowHours
   c. If condition met:
      - Create alert_instances row
      - Send notifications via configured channels
      - Record cooldown timestamp
```

Notification dispatch:
```typescript
export async function sendAlertNotification(
  alert: AlertInstance,
  rule: AlertRule
): Promise<void> {
  for (const channel of rule.config.channels) {
    switch (channel) {
      case 'email':
        await sendAlertEmail(alert, rule);
        break;
      case 'slack':
        await sendSlackWebhook(alert, rule.config.slackWebhook!);
        break;
      case 'webhook':
        await sendWebhookNotification(alert, rule.config.webhookUrl!);
        break;
    }
  }
}
```

**Testing**:
- `Unit: evaluateReputationDrop(currentScore=80, previousScore=90, threshold=5) → fires alert`
- `Unit: evaluateReputationDrop(currentScore=88, previousScore=90, threshold=5) → no alert`
- `Unit: evaluateBounceRateThreshold(bounceRate=0.06, threshold=0.05) → fires alert`
- `Unit: shouldSkipCooldown(lastFiredAt=1h ago, cooldown=4h) → true (skip)`
- `Unit: shouldSkipCooldown(lastFiredAt=5h ago, cooldown=4h) → false (evaluate)`
- `Integration (mocked notifications): alert fires → email sent, Slack webhook called`
- `Integration: alert evaluation with cooldown active → no duplicate alert created`
- `Integration: reputation drop across window → alert created with correct context`

---

## Phase 9: Dashboard Frontend

### Purpose

Build the Next.js dashboard with domain health overview, reputation trends, warm-up progress, placement test results, DMARC report viewer, and alert management.

### Tasks

#### 9.1 — Dashboard Layout and Navigation

**What**: Authenticated dashboard shell with sidebar navigation, tenant context, and responsive layout.

**Design**:

Dashboard route structure:
```
/dashboard                    → Overview (domain health cards, recent alerts)
/dashboard/domains            → Domain list
/dashboard/domains/:id        → Domain detail (auth health, reputation, compliance)
/dashboard/mailboxes          → Mailbox list + connection status
/dashboard/warmup             → Warm-up schedules + progress charts
/dashboard/placement          → Placement test list + results
/dashboard/reputation         → Reputation dashboard + trends
/dashboard/content            → Content analysis tool
/dashboard/dmarc              → DMARC report viewer
/dashboard/alerts             → Alert rules + history
/dashboard/settings           → Tenant settings, users, API keys
```

API client (`apps/web/src/lib/api-client.ts`):
```typescript
import { QueryClient } from '@tanstack/react-query';

export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 30_000,        // 30 seconds
      refetchOnWindowFocus: true,
    },
  },
});

export async function apiClient<T>(
  path: string,
  options?: RequestInit
): Promise<T> {
  const res = await fetch(`${process.env.NEXT_PUBLIC_API_URL}${path}`, {
    ...options,
    headers: {
      'Content-Type': 'application/json',
      Authorization: `Bearer ${getToken()}`,
      ...options?.headers,
    },
  });
  if (!res.ok) throw new ApiError(res.status, await res.json());
  return res.json();
}
```

**Testing**:
- `E2E: unauthenticated user visits /dashboard → redirected to /login`
- `E2E: authenticated user visits /dashboard → sees sidebar with all nav items`
- `E2E: sidebar navigation → each link loads the correct page`
- `E2E: responsive layout → sidebar collapses on mobile viewport`

---

#### 9.2 — Domain Health and Reputation Views

**What**: Domain list with health indicators, domain detail page with authentication status cards, and reputation trend charts.

**Design**:

Components:
```typescript
// DomainHealthCard: displays SPF/DKIM/DMARC/BIMI status as green/red indicators
interface DomainHealthCardProps {
  domain: Domain;
}

// ReputationChart: time-series line chart of reputation score over time
interface ReputationChartProps {
  scores: ReputationScore[];
  dateRange: { from: string; to: string };
}

// AuthStatusPanel: detailed auth record display with validation errors
interface AuthStatusPanelProps {
  authHealth: AuthHealth;
  records: AuthRecord[];
}
```

Charts use Recharts with:
- Reputation score over time (line chart, 30/60/90-day ranges)
- Inbox/spam/bounce rate stacked area chart
- Provider-level breakdown bar chart
- Complaint rate with Google's 0.3% threshold line

**Testing**:
- `E2E: domain list page → shows all tenant domains with status badges`
- `E2E: domain detail page → shows auth health cards (green check / red X for each protocol)`
- `E2E: reputation chart → renders with correct data points, responds to date range selector`
- `E2E: domain with DMARC at p=none → warning indicator suggesting enforcement upgrade`

---

#### 9.3 — Warm-Up and Placement Test Views

**What**: Warm-up progress dashboard with daily volume charts and placement test results with ISP-level breakdowns.

**Design**:

Warm-up dashboard:
- Active schedule list with status indicators
- Daily volume chart (sent, inbox, spam, promotions) as stacked bar chart
- Provider-level breakdown table
- Start/pause/resume controls

Placement test results:
- Overall placement pie chart (inbox/spam/promotions/missing)
- Provider-level breakdown table with placement percentages
- Spam filter results panel (SpamAssassin score, rules triggered)
- Individual seed results expandable table
- "Run new test" form with subject line and HTML body input

**Testing**:
- `E2E: warm-up page → shows active schedules with progress bars`
- `E2E: pause warm-up → schedule status changes to paused, UI updates`
- `E2E: placement test results → pie chart renders, provider breakdown matches data`
- `E2E: run new placement test → form submits, test appears in list with pending status`

---

#### 9.4 — DMARC Report Viewer and Alert Management

**What**: DMARC aggregate report viewer showing authentication results per reporting org, and alert rule configuration with history view.

**Design**:

DMARC viewer:
- Report list grouped by reporting org (Google, Yahoo, Microsoft)
- Report detail: pass/fail summary, unauthorised sender table, source IP list
- Cross-report trend: authentication pass rate over time
- Unauthorised senders view: IPs sending unauthenticated email on behalf of the domain

Alert management:
- Alert rule list with toggle for active/inactive
- New rule form: condition type, threshold, window, severity, notification channels
- Alert history with severity badges, acknowledge button
- Unacknowledged alert count in sidebar badge

**Testing**:
- `E2E: DMARC report list → displays reports grouped by reporting org with date ranges`
- `E2E: DMARC report detail → shows pass/fail table with source IPs`
- `E2E: alert rules page → list rules, toggle active/inactive`
- `E2E: create alert rule → form submits, rule appears in list`
- `E2E: unacknowledged alerts → badge shows count in sidebar, acknowledge button works`

---

## Phase 10: AI-Powered Features

### Purpose

Implement the AI-native differentiators: adaptive warm-up scheduling, content risk prediction, automated DMARC progression guidance, and sender reputation forecasting. These features move the platform from reactive monitoring to proactive optimisation.

### Tasks

#### 10.1 — Adaptive Warm-Up Scheduling

**What**: Replace the linear warm-up ramp with an AI model that adjusts daily volume based on real-time engagement, bounce, and spam-trap signals.

**Design**:

```typescript
export interface WarmupAdaptiveModel {
  // Inputs (features)
  recentInboxRate: number;       // 7-day rolling inbox placement rate
  recentSpamRate: number;        // 7-day rolling spam rate
  recentBounceRate: number;      // 7-day rolling bounce rate
  daysSinceStart: number;
  currentDailyVolume: number;
  providerMix: Record<string, number>;  // Actual provider distribution

  // Outputs (decisions)
  recommendedDailyVolume: number;
  riskScore: number;             // 0.0 (safe) to 1.0 (dangerous)
  adjustmentReason: string;
  shouldPause: boolean;
}

export function computeAdaptiveSchedule(
  schedule: WarmupSchedule,
  recentLogs: WarmupDailyLog[],
  authHealth: AuthHealth
): WarmupAdaptiveModel {
  // Decision rules:
  // 1. If spam rate > 10% over last 3 days → reduce volume by 30%, increase riskScore
  // 2. If inbox rate > 95% and bounce rate < 2% over last 7 days → increase volume by rampIncrement * 1.5
  // 3. If auth checks failing → pause warm-up until resolved
  // 4. If provider-specific spam rate > 15% → reduce volume to that provider
  // 5. If engagement momentum > 0.85 → accelerate ramp-up
  // 6. Auto-complete warm-up when dailyTarget reaches dailyMax and inbox rate stable > 90% for 14 days
}
```

The adaptive model stores its state in `warmup_schedules.ai_state` JSONB, enabling inspection and debugging of AI decisions.

**Testing**:
- `Unit: computeAdaptiveSchedule with high inbox rate → increases daily volume`
- `Unit: computeAdaptiveSchedule with spam rate spike → reduces volume, increases riskScore`
- `Unit: computeAdaptiveSchedule with auth failure → shouldPause=true`
- `Unit: computeAdaptiveSchedule with sustained good metrics → accelerated ramp-up`
- `Unit: computeAdaptiveSchedule reaching dailyMax with stable metrics → status=completed`
- `Integration: warm-up cycle with adaptive strategy → ai_state updated with recommendation`
- `Integration: adaptive model pauses warm-up → schedule status=paused, alert fired`

---

#### 10.2 — Content Risk Prediction (AI-Enhanced)

**What**: Enhance rule-based content analysis with GPT-4o for context-aware risk assessment and composition suggestions.

**Design**:

```typescript
export interface AiContentAnalysis extends ContentAnalysis {
  aiInsights: {
    contextualRisk: 'low' | 'medium' | 'high';
    riskExplanation: string;
    suggestedRevisions: Array<{
      original: string;
      suggested: string;
      reason: string;
    }>;
    subjectLineAlternatives: string[];
    overallAssessment: string;
  };
}

// System prompt for content risk analysis
const CONTENT_RISK_PROMPT = `
You are an email deliverability expert. Analyze the following email content
for deliverability risks. Consider:
1. Spam trigger words and phrases in context (some are fine in context)
2. Link-to-text ratio and link placement patterns
3. Image-to-text ratio
4. Formatting patterns that trigger spam filters
5. Subject line deliverability risks
6. Overall tone and sender reputation implications

Respond with:
- contextualRisk: "low", "medium", or "high"
- riskExplanation: one paragraph explaining the key risks
- suggestedRevisions: specific text changes to improve deliverability
- subjectLineAlternatives: 3 alternative subject lines
- overallAssessment: one-sentence summary
`;
```

API route:
```typescript
// POST /api/v1/content/analyze-ai
// Body: { subjectLine: string, htmlBody: string, senderDomain?: string }
// Response: { analysis: AiContentAnalysis }
```

**Testing**:
- `Unit (mocked LLM): analyzeContentWithAi(email with "free" in business context) → contextualRisk: "low"`
- `Unit (mocked LLM): analyzeContentWithAi(spammy email) → contextualRisk: "high", suggestedRevisions populated`
- `Integration (mocked LLM): POST /api/v1/content/analyze-ai → returns AI insights alongside rule-based analysis`
- `Unit: AI response parsing handles malformed LLM output gracefully → falls back to rule-based only`

---

#### 10.3 — Automated DMARC Progression Guidance

**What**: Analyse DMARC aggregate reports and authentication alignment to recommend safe policy progression from p=none to p=quarantine to p=reject. Address the 75-80% of domains that stall at p=none.

**Design**:

```typescript
export interface DmarcProgressionAssessment {
  domainId: string;
  currentPolicy: 'none' | 'quarantine' | 'reject';
  recommendedPolicy: 'none' | 'quarantine' | 'reject';
  readyToProgress: boolean;
  confidence: number;        // 0.0 - 1.0
  blockers: string[];
  assessment: {
    dkimAlignmentRate: number;   // % of messages with DKIM pass + alignment
    spfAlignmentRate: number;    // % of messages with SPF pass + alignment
    unauthorisedSenderCount: number;
    reportCoverage: number;      // Number of days with DMARC report data
    knownSendingSources: Array<{
      sourceIp: string;
      authenticated: boolean;
      volume: number;
      service?: string;          // e.g., "Google Workspace", "SendGrid"
    }>;
  };
  recommendation: string;       // Human-readable recommendation
}

export function assessDmarcProgression(
  domain: Domain,
  reports: DmarcAggregateReport[],
  authRecords: AuthRecord[]
): DmarcProgressionAssessment {
  // Progression rules:
  // p=none → p=quarantine when:
  //   - DKIM alignment rate > 98% over 30+ days of reports
  //   - SPF alignment rate > 95% over 30+ days of reports
  //   - All legitimate sending sources identified and authenticated
  //   - No more than 2% unauthenticated traffic from known IPs
  //
  // p=quarantine → p=reject when:
  //   - All above criteria met at p=quarantine for 30+ days
  //   - No legitimate email quarantined (0% false positive rate)
  //   - DKIM + SPF alignment both > 99%
}
```

API route:
```typescript
// GET /api/v1/domains/:id/dmarc/progression
// Response: { assessment: DmarcProgressionAssessment }
```

**Testing**:
- `Unit: assessDmarcProgression with 99% alignment, 60 days data → readyToProgress=true, recommend quarantine`
- `Unit: assessDmarcProgression with 90% alignment → readyToProgress=false, blockers include alignment issue`
- `Unit: assessDmarcProgression with < 30 days data → readyToProgress=false, blocker: insufficient data`
- `Unit: assessDmarcProgression at p=reject → recommendedPolicy=reject, no further progression needed`
- `Integration: GET /api/v1/domains/:id/dmarc/progression → returns full assessment with known senders`

---

#### 10.4 — Sender Reputation Forecasting

**What**: Time-series model that predicts future reputation scores based on current sending patterns, enabling proactive intervention.

**Design**:

```typescript
export interface ReputationForecast {
  domainId: string;
  forecastDate: string;
  currentScore: number;
  predictions: Array<{
    date: string;
    predictedScore: number;
    confidenceInterval: { low: number; high: number };
  }>;
  riskFactors: Array<{
    factor: string;
    impact: number;       // Negative values = downward pressure
    description: string;
  }>;
  recommendation: string;
}

export function forecastReputation(
  historicalScores: ReputationScore[],
  currentEvents: NormalisedEmailEvent[],
  authHealth: AuthHealth
): ReputationForecast {
  // Model: weighted moving average with trend detection
  // Inputs: 90 days of historical scores, recent 7-day event rates
  // Output: 30-day forecast with daily predictions
  //
  // Risk factors:
  // - Rising bounce rate → negative impact
  // - Increasing complaint rate → strong negative impact
  // - Auth failures → negative impact
  // - Declining engagement (open/click rates) → moderate negative impact
  // - Blacklist listing → severe negative impact
}
```

API route:
```typescript
// GET /api/v1/domains/:id/reputation/forecast
// Response: { forecast: ReputationForecast }
```

**Testing**:
- `Unit: forecastReputation with stable high scores → flat prediction, no risk factors`
- `Unit: forecastReputation with rising bounce rate → declining prediction, risk factor identified`
- `Unit: forecastReputation with blacklist listing → sharp decline predicted`
- `Unit: forecastReputation with insufficient historical data (< 14 days) → error: insufficient data`
- `Integration: GET /api/v1/domains/:id/reputation/forecast → returns 30-day forecast with confidence intervals`

---

## Phase 11: API Polish, OpenAPI Spec, and Developer Experience

### Purpose

Generate an OpenAPI 3.1 specification from the Fastify routes, add rate limiting, request validation, comprehensive error responses, and API documentation.

### Tasks

#### 11.1 — OpenAPI Specification and Documentation

**What**: Auto-generate OpenAPI 3.1 spec from Fastify route schemas, serve Swagger UI, and publish API reference documentation.

**Design**:

Fastify Swagger plugin configuration:
```typescript
import fastifySwagger from '@fastify/swagger';
import fastifySwaggerUi from '@fastify/swagger-ui';

await app.register(fastifySwagger, {
  openapi: {
    info: {
      title: 'Email Deliverability Platform API',
      version: '1.0.0',
      description: 'API for sender reputation monitoring, warm-up automation, and inbox placement testing.',
    },
    servers: [
      { url: 'http://localhost:3001', description: 'Development' },
    ],
    components: {
      securitySchemes: {
        bearerAuth: { type: 'http', scheme: 'bearer', bearerFormat: 'JWT' },
        apiKeyAuth: { type: 'apiKey', in: 'header', name: 'Authorization' },
      },
    },
    security: [{ bearerAuth: [] }, { apiKeyAuth: [] }],
  },
});

await app.register(fastifySwaggerUi, { routePrefix: '/docs' });
```

Every route must have JSON Schema for request body, query params, path params, and response.

**Testing**:
- `Integration: GET /docs → Swagger UI loads`
- `Integration: GET /docs/json → valid OpenAPI 3.1 spec returned`
- `Integration: OpenAPI spec validates with swagger-cli validate → no errors`
- `Integration: every API route appears in the spec with request/response schemas`

---

#### 11.2 — Rate Limiting and Error Handling

**What**: Redis-based rate limiting per tenant/API key, standardised error response format, and request validation.

**Design**:

Rate limiting tiers:
```typescript
export const RATE_LIMITS = {
  free:       { windowMs: 60_000, max: 60 },
  starter:    { windowMs: 60_000, max: 300 },
  growth:     { windowMs: 60_000, max: 600 },
  enterprise: { windowMs: 60_000, max: 1200 },
} as const;
```

Standard error response:
```typescript
export interface ApiError {
  error: string;
  message: string;
  statusCode: number;
  details?: Record<string, unknown>;
}

// Examples:
// 400: { error: "VALIDATION_ERROR", message: "domainName is required", statusCode: 400, details: { field: "domainName" } }
// 401: { error: "UNAUTHORIZED", message: "Invalid or expired token", statusCode: 401 }
// 403: { error: "FORBIDDEN", message: "Insufficient permissions", statusCode: 403 }
// 404: { error: "NOT_FOUND", message: "Domain not found", statusCode: 404 }
// 409: { error: "CONFLICT", message: "Domain already registered", statusCode: 409 }
// 429: { error: "RATE_LIMITED", message: "Rate limit exceeded", statusCode: 429 }
```

**Testing**:
- `Integration: 61 requests in 60s on free plan → 429 on 61st request`
- `Integration: 301 requests in 60s on starter plan → 429 on 301st`
- `Integration: request with invalid JSON body → 400 with VALIDATION_ERROR`
- `Integration: request to non-existent route → 404 with NOT_FOUND`
- `Integration: rate limit headers (X-RateLimit-Remaining, X-RateLimit-Reset) present in responses`

---

## Phase 12: Deployment, CI/CD, and Production Readiness

### Purpose

Containerise all services, set up CI/CD pipeline, add health checks, structured logging, and production configuration. The platform is deployable to any Docker-capable environment after this phase.

### Tasks

#### 12.1 — Docker Production Images

**What**: Multi-stage Docker builds for API, worker, and web services optimised for production.

**Design**:

`Dockerfile.api`:
```dockerfile
FROM node:22-alpine AS base
RUN corepack enable && corepack prepare pnpm@latest --activate

FROM base AS build
WORKDIR /app
COPY pnpm-workspace.yaml pnpm-lock.yaml ./
COPY packages/shared/package.json packages/shared/
COPY apps/api/package.json apps/api/
RUN pnpm install --frozen-lockfile
COPY packages/shared/ packages/shared/
COPY apps/api/ apps/api/
RUN pnpm --filter @edp/api build

FROM base AS production
WORKDIR /app
COPY --from=build /app/apps/api/dist ./dist
COPY --from=build /app/node_modules ./node_modules
ENV NODE_ENV=production
EXPOSE 3001
CMD ["node", "dist/server.js"]
```

**Testing**:
- `Integration: docker build -f Dockerfile.api → builds without errors`
- `Integration: docker build -f Dockerfile.web → builds without errors`
- `Integration: docker build -f Dockerfile.worker → builds without errors`
- `Integration: docker-compose up → all services start, API responds to health check`
- `Integration: API container → /health returns { status: "ok", db: "connected", redis: "connected" }`

---

#### 12.2 — CI/CD Pipeline

**What**: GitHub Actions workflow for linting, type checking, testing, building, and optional deployment.

**Design**:

`.github/workflows/ci.yml`:
```yaml
name: CI
on: [push, pull_request]
jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: pnpm }
      - run: pnpm install --frozen-lockfile
      - run: pnpm lint
      - run: pnpm typecheck
      - run: pnpm test
  docker:
    runs-on: ubuntu-latest
    needs: quality
    steps:
      - uses: actions/checkout@v4
      - run: docker compose build
      - run: docker compose up -d
      - run: ./scripts/healthcheck.sh
      - run: docker compose down
```

**Testing**:
- `Integration: CI workflow runs on push → lint, typecheck, test, build all pass`
- `Integration: CI workflow fails on lint error → build marked as failed`
- `Integration: health check script → verifies API, Redis, PostgreSQL connectivity`

---

#### 12.3 — Structured Logging and Monitoring

**What**: Pino-based structured logging with request IDs, tenant context, and JSON output for log aggregation.

**Design**:

```typescript
import pino from 'pino';

export const logger = pino({
  level: env.LOG_LEVEL,
  transport: env.NODE_ENV === 'development'
    ? { target: 'pino-pretty', options: { colorize: true } }
    : undefined,
  serializers: {
    req: (req) => ({
      method: req.method,
      url: req.url,
      tenantId: req.auth?.tenantId,
      userId: req.auth?.userId,
    }),
    res: (res) => ({
      statusCode: res.statusCode,
    }),
  },
});

// Every log line includes: timestamp, level, requestId, tenantId, message
```

**Testing**:
- `Unit: logger in production → outputs JSON without colours`
- `Unit: logger in development → outputs formatted with colours`
- `Integration: API request → log line includes requestId, tenantId, method, url, statusCode`
- `Integration: worker job → log line includes jobId, jobType, duration`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation (scaffolding, DB, auth)        ─── required by everything
    │
Phase 2: Domain Management & DNS Auth              ─── requires Phase 1
    │
Phase 3: Mailbox Connection & Email Infrastructure ─── requires Phase 1
    │
    ├── Phase 4: Warm-Up Engine                    ─── requires Phases 2, 3
    │
    ├── Phase 5: Inbox Placement Testing           ─── requires Phases 2, 3
    │
    └── Phase 6: Reputation Scoring & Blacklists   ─── requires Phase 2
         │
Phase 7: DMARC Report Ingestion                    ─── requires Phases 2, 3
    │
Phase 8: Alerting System                           ─── requires Phases 2, 6
    │
Phase 9: Dashboard Frontend                        ─── requires Phases 2-8
    │
Phase 10: AI-Powered Features                      ─── requires Phases 4, 5, 6, 7
    │
Phase 11: API Polish & Developer Experience        ─── requires Phases 2-8
    │
Phase 12: Deployment & Production Readiness        ─── requires all phases

Parallelism opportunities:
  - Phases 4, 5, 6 can be developed concurrently after Phases 2 and 3
  - Phase 7 can be developed in parallel with Phases 4-6
  - Phase 8 can start once Phase 6 is complete
  - Phases 10 and 11 can be developed in parallel after Phases 4-8
```

---

## Definition of Done (per phase)

1. All tasks in the phase are implemented.
2. All unit tests pass (`pnpm test`).
3. All integration tests pass (mocked external dependencies).
4. ESLint passes with zero warnings (`pnpm lint`).
5. Prettier formatting applied (`pnpm format:check`).
6. TypeScript strict mode compilation passes (`pnpm typecheck`).
7. Docker build succeeds for affected services.
8. Feature works end-to-end (manual verification or E2E test).
9. New API endpoints appear in the auto-generated OpenAPI spec.
10. Database migrations created and tested (up and down).
11. New configuration options documented in `.env.example`.
12. No secrets, credentials, or API keys committed to source control.
