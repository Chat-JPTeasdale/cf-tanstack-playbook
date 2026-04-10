---
name: workflow
description: Full-stack architecture playbook for Cloudflare Workers + TanStack Start + Neon Postgres projects. Encodes battle-tested patterns from production codebases — directory structure, API design, auth, database, frontend, and deployment. Use when scaffolding a new project, refactoring an existing one, or reviewing architecture decisions.
---

# CF Workers + TanStack Start Playbook

Derived from production patterns. Every rule exists because the alternative caused a real problem.

## 0. APPLICATION DATA FLOW

### Architecture Overview

```
Browser
  │
  ├── GET /practice/clients ─────> Cloudflare Worker ──> TanStack Start (SSR)
  │                                    src/server.ts         │
  │                                    routes to the         │ renders HTML
  │                                    correct app           │ with React
  │                                    based on path         │
  │                                                          ▼
  │                                                     Browser (hydrated)
  │                                                          │
  └── fetch /api/v1/practitioner/clients ──> Cloudflare Worker ──> Hono API
                                                                     │
                                              ┌──────────────────────┘
                                              │
                                              ▼
                                         Middleware Stack
                                         1. Rate limit
                                         2. Setup (create DB + Auth from env)
                                         3. Logger
                                         4. Auth (validate session via better-auth)
                                              │
                                              ▼
                                         Route Handler
                                              │
                                              ▼
                                    Cloudflare Hyperdrive
                                    (connection pool, TLS)
                                              │
                                              ▼
                                       Neon Postgres
                                       SET LOCAL ROLE ch_<X>
                                       RLS enforces row access
```

**The rule:** One Worker, two apps. `/api/*` goes to Hono. Everything else goes to TanStack Start. They share nothing at runtime — separate middleware, separate auth validation, separate DB connections. The split happens in `src/server.ts`:

```typescript
if (url.pathname.startsWith('/api/')) return api.fetch(request, env, ctx);
return startHandler(request, { context: { executionCtx: ctx, env } });
```

### Layer Responsibilities

| Layer | Technology | Responsibility | Why This Choice |
|-------|-----------|---------------|-----------------|
| **SSR Frontend** | TanStack Start | Server-rendered React pages, file-based routing, `createServerFn` for page data loading | Full React 19 SSR on CF Workers with streaming |
| **Typed REST API** | Hono (OpenAPIHono) | All business logic, CRUD operations, integrations | Consumable by any client (mobile, CLI, third-party). TanStack server functions are NOT externally consumable — they're tied to the React component tree |
| **Auth** | better-auth | User accounts, sessions, email OTP, organizations, invitations, role management | Generates its own Drizzle schema (`schema/auth.ts`). Runs as Hono middleware on `/api/auth/*` AND as TanStack Start server function for SSR session loading |
| **Database** | Drizzle ORM + Neon | Schema definition, migrations, type-safe queries | All non-auth tables are RLS-protected. Auth tables are managed by better-auth (no manual RLS needed — better-auth handles its own access control) |
| **Connection Pool** | Cloudflare Hyperdrive | Persistent TCP connections to Neon from the edge | Eliminates cold-start latency for DB connections. Handles TLS termination |
| **Session Cache** | Cloudflare KV | better-auth secondary storage for session tokens | Avoids hitting Neon for every session validation. KV is globally replicated |
| **Object Storage** | Cloudflare R2 | Media uploads (lab PDFs, profile images) | S3-compatible. Presigned URLs for direct upload. Serve through Worker for HIPAA, presigned URLs for non-HIPAA |
| **Type Generation** | openapi-typescript | `v1.d.ts` from OpenAPI spec | Single source of truth for frontend types. `npm run gen:api` regenerates after any route change |
| **Frontend API Client** | openapi-fetch + openapi-react-query | Typed fetch + React Query hooks | Zero manual fetch wrappers. Types flow from Zod schemas -> OpenAPI spec -> TypeScript types -> React hooks |

### Data Flow: Page Load (SSR)
```
1. Browser requests /practice/clients
2. Worker routes to TanStack Start (not /api/*)
3. Root route beforeLoad calls getSession() server function
4. Server function creates DB + auth from Worker env bindings
5. auth.api.getSession({ headers }) validates session cookie
6. Session injected into route context
7. /practice route beforeLoad checks context.session, redirects if missing
8. /practice/clients component renders with session context
9. Component calls api.useQuery('get', '/api/v1/practitioner/clients')
10. Browser makes fetch to /api/v1/practitioner/clients with session cookie
11. Hono middleware validates session, resolves org membership, sets RLS role
12. Handler queries Neon through Hyperdrive with SET LOCAL ROLE ch_practitioner
13. RLS policies filter rows to current org only
14. Typed response flows back through openapi-react-query hook
```

### Data Flow: API Call (Client-Side)
```
1. Component calls api.useMutation('post', '/api/v1/practitioner/clients')
2. openapi-fetch sends typed POST with session cookie
3. Hono validates session + org membership
4. Zod schema validates request body (from createRoute definition)
5. Handler inserts via Drizzle within RLS transaction
6. Response validated against Zod response schema
7. openapi-react-query invalidates relevant query cache
```

### Auth Table Ownership
better-auth manages these tables (generated by `npm run gen:auth`):
- `user`, `session`, `account`, `verification` — core auth
- `organization`, `member`, `invitation` — multi-tenancy

**Do NOT add RLS policies to auth tables.** better-auth handles its own access control. Only add RLS to your application tables (clients, labs, biomarkers, etc.).

### Cloudflare Products: Current + Available

| Product | Status | Use Case |
|---------|--------|----------|
| **Workers** | Active | Application runtime (Hono API + TanStack Start SSR) |
| **Hyperdrive** | Active | Postgres connection pooling to Neon |
| **KV** | Active | Session token cache for better-auth secondary storage |
| **R2** | Active | Object storage (lab PDFs, media uploads) |
| **Durable Objects** | Available | Rate limiting (per-IP state), real-time collaboration (live cursors, shared editing), WebSocket connections, stateful workflow orchestration |
| **Queues** | Available | Background job processing (lab PDF extraction, email sending, webhook delivery, convergence analysis). Decouple long-running work from request/response cycle |
| **Workflows** | Available | Multi-step durable execution (lab upload -> extract biomarkers -> run convergence -> notify practitioner). Survives Worker restarts, has built-in retry/backoff |
| **AI Gateway** | Available | Proxy and cache LLM calls (Claude, GPT). Rate limiting, logging, cost tracking for AI features |
| **Workers AI** | Available | On-edge inference for lightweight models (embeddings, classification). Useful for biomarker categorization without external API calls |
| **D1** | Not recommended | SQLite at the edge. Neon + Hyperdrive is superior for Postgres-native features (RLS, JSONB, full-text search) |
| **Vectorize** | Available | Vector database for semantic search over biomarker knowledge, clinical literature |
| **Browser Rendering** | Available | Server-side PDF generation (lab reports, client summaries) |

### When to Add Queues
Add Queues when any of these become true:
- Lab PDF extraction takes > 5 seconds (move to background)
- Email sending blocks API responses
- Webhook delivery needs retry logic
- Convergence analysis is computationally expensive

Pattern: API handler enqueues a message, Queue consumer processes it, writes result to DB, client polls or gets notified via WebSocket.

### When to Add Workflows
Add Workflows when you need multi-step processes that must complete even if the Worker restarts:
- Lab upload pipeline: validate -> extract biomarkers -> categorize -> run convergence -> store results -> notify
- Onboarding flow: create org -> invite practitioners -> set up billing -> send welcome email
- Billing: create invoice -> charge card -> handle failure -> retry -> notify

### When to Add Durable Objects
Add Durable Objects when you need:
- Per-entity state that survives across requests (rate limiter per IP, session counter per org)
- WebSocket connections (real-time collaboration on lab analysis)
- Coordination between multiple Workers (distributed locking)

## 1. DIRECTORY STRUCTURE

```
src/
  components/
    ui/           # shadcn primitives — never edit logic, only restyle
    blocks/       # composed page-level components (shells, layouts, cards)
  hooks/          # React hooks (use-mobile, use-debounce, etc.)
  lib/
    api/          # API app, middleware, routes, schemas, typed client
      middleware/  # Hono middleware (auth, rate-limit, logging, error handler)
      routes/     # Route modules grouped by audience (practitioner/, client/)
      utils/      # API helpers (REST route factory, response builders)
      client.ts   # openapi-fetch typed client for frontend consumption
      index.ts    # Hono app — mounts all routers, exports for Worker
      schemas.ts  # Zod schemas for all request/response types
      types.ts    # ApiContext, Env, middleware type extensions
      v1.d.ts     # Generated — openapi-typescript output, never hand-edit
    auth/         # Auth client, hooks, server functions, config
    db/
      schema/     # One file per table, shared/ for helpers, barrel index.ts
      queries/    # Query modules (pagination, search helpers)
      index.ts    # createDb, type exports
    email/        # Email templates and sending logic
  routes/         # TanStack Router file routes
    auth/         # sign-in, sign-up, forgot-password, reset-password
    practice/     # Practitioner portal (behind auth guard)
    __root.tsx    # Root layout, providers, head config
    index.tsx     # Landing page
  types/          # Global type declarations (fonts.d.ts, env augmentation)
  styles.css      # Tailwind v4 + shadcn theme variables
scripts/          # CLI scripts (db-init, seed, gen-api-types)
drizzle/          # Generated migrations — never hand-edit
docs/             # PLAN.md, SPEC.md, DEPLOYMENT.md, LOCAL_SETUP.md
```

### Rules

- **One file per table** in `db/schema/`. Name matches table: `clients.ts` for `clients` table. Barrel export from `index.ts`.
- **Route files grouped by audience**, not by resource. `routes/practitionerRouter/` has clients, labs, billing. `routes/clientRouter/` has patient-facing endpoints. This prevents accidental role leakage.
- **`components/ui/`** is shadcn-managed. Customize colors and radii via CSS variables, not by editing component internals. If you need behavior changes, create a wrapper in `blocks/`.
- **`blocks/`** contains page-level compositions: `auth-layout.tsx`, `practice-shell.tsx`, `landing-hero.tsx`. These import from `ui/` and compose them.
- **`v1.d.ts` is generated.** Run `npm run gen:api` after any route change. Never hand-write types here.

## 2. WORKER ENTRY POINT (src/server.ts)

The Cloudflare Worker runs **two apps side by side** from a single `fetch` handler:

```typescript
// src/server.ts
import { createStartHandler, defaultStreamHandler } from '@tanstack/react-start/server';
import { api } from '~/lib/api';

const startHandler = createStartHandler(defaultStreamHandler);

export default {
  async fetch(request: Request, env: Env, ctx: ExecutionContext) {
    const url = new URL(request.url);
    // Hono API handles /api/* routes
    if (url.pathname.startsWith('/api/')) {
      return api.fetch(request, env, ctx);
    }
    // TanStack Start handles everything else (SSR, pages, assets)
    return startHandler(request, {
      context: { executionCtx: ctx, env },
    });
  },
};

// Type augmentation for TanStack Start server context
declare module '@tanstack/react-start' {
  interface Register {
    server: {
      requestContext: { env: Env; executionCtx: ExecutionContext };
    };
  }
}
```

**Key insight:** The Hono API and TanStack Start share the same Worker but are independent apps. The API has its own middleware stack, auth validation, and DB connections. TanStack Start has its own server functions and middleware via `createMiddleware`/`createServerFn`.

## 3. API ARCHITECTURE

### Stack
- **OpenAPIHono** (not plain Hono) for all route files. Every route uses `createRoute()` with Zod schemas.
- **Zod schemas** define request params, query, body, and response for every endpoint. These drive both validation and OpenAPI spec generation.
- **`app.doc31('/api/openapi.json', ...)`** endpoint serves the spec. `gen:api` script feeds this into `openapi-typescript` to produce `v1.d.ts`.

### API App Setup (lib/api/index.ts)
```typescript
const api = new OpenAPIHono<ApiContext>().basePath('/api');

api.onError(handleError);
api.use('*', rateLimitMiddleware);
api.use('*', apiMiddlewareSetup);     // creates DB + auth instances
api.use('*', apiMiddlewareRequestLogger);

// Public routes
api.route('/health', healthRouter);

// better-auth passthrough — handles its own auth logic
api.on(['POST', 'GET'], '/auth/*', (c) => {
  const auth = c.get('auth');
  return auth.handler(c.req.raw);
});

// Protected routes
api.use('/v1/*', apiMiddlewareRequireAuth);
api.route('/v1/invitations', invitationsRouter);
api.route('/v1/client', clientRouter);
api.route('/v1/practitioner', practitionerRouter);

api.doc31('/openapi.json', { ... });
```

### API Context Type (lib/api/types.ts)
```typescript
type ApiContext = {
  Bindings: Env;
  Variables: {
    db: Database;           // Drizzle instance from Hyperdrive
    session: AuthSession;   // better-auth session (set by auth middleware)
    auth: Auth;             // better-auth instance (for session validation)
    query: DatabaseQuery;   // RLS-scoped query runner
    querySystem: DatabaseQuery; // System-role query runner
    requestId: string;
  };
};
```

### Setup Middleware (lib/api/middleware/api-setup.ts)
Creates DB and auth instances per-request from Worker bindings:
```typescript
const db = createDb(c.env.DB.connectionString);       // Hyperdrive binding
const auth = createAuth(db, c.env, ctx);              // better-auth with DB + env
c.set('db', db);
c.set('auth', auth);
```

### Pagination Pattern

**CRITICAL RULE:** Every list endpoint MUST support pagination. Never return unbounded result sets.

#### Standard Pagination Schema

```typescript
// lib/api/schemas/pagination.ts
import { z } from 'zod';

export const paginationQuerySchema = z.object({
  page: z.coerce.number().int().min(1).default(1),
  limit: z.coerce.number().int().min(1).max(100).default(20),
});

export type PaginationQuery = z.infer<typeof paginationQuerySchema>;

export const paginatedResponseSchema = <T extends z.ZodTypeAny>(itemSchema: T) =>
  z.object({
    data: z.array(itemSchema),
    pagination: z.object({
      page: z.number(),
      limit: z.number(),
      total: z.number(),
      totalPages: z.number(),
      hasNext: z.boolean(),
      hasPrev: z.boolean(),
    }),
  });
```

#### Pagination Helper

```typescript
// lib/api/utils/pagination.ts
export interface PaginationParams {
  page: number;
  limit: number;
}

export interface PaginationMeta {
  page: number;
  limit: number;
  total: number;
  totalPages: number;
  hasNext: boolean;
  hasPrev: boolean;
}

export function getPaginationMeta(
  params: PaginationParams,
  total: number
): PaginationMeta {
  const totalPages = Math.ceil(total / params.limit);
  return {
    page: params.page,
    limit: params.limit,
    total,
    totalPages,
    hasNext: params.page < totalPages,
    hasPrev: params.page > 1,
  };
}

export function applyPagination<T>(
  query: any, // Drizzle query builder
  params: PaginationParams
) {
  return query
    .limit(params.limit)
    .offset((params.page - 1) * params.limit);
}
```

#### Usage in Routes

```typescript
// lib/api/routes/practitioner/clients.ts
import { createRoute } from '@hono/zod-openapi';
import { count } from 'drizzle-orm';
import { paginationQuerySchema, paginatedResponseSchema } from '~/lib/api/schemas/pagination';
import { getPaginationMeta, applyPagination } from '~/lib/api/utils/pagination';
import { clientResponseSchema } from '~/lib/db/schema/clients.zod';
import { clients } from '~/lib/db/schema/clients';

const listRoute = createRoute({
  method: 'get',
  path: '/',
  request: { 
    query: paginationQuerySchema  // ✅ Standard pagination
  },
  responses: {
    200: { 
      content: { 
        'application/json': { 
          schema: paginatedResponseSchema(clientResponseSchema)  // ✅ Standard wrapper
        } 
      }, 
      description: 'List clients' 
    },
  },
});

router.openapi(listRoute, async (c) => {
  const params = c.req.valid('query');  // { page: 1, limit: 20 }
  const query = c.get('query');  // RLS-scoped
  
  // Get total count (for pagination meta)
  const [{ total }] = await query
    .select({ total: count() })
    .from(clients);
  
  // Get paginated data
  const data = await applyPagination(
    query.select().from(clients),
    params
  );
  
  return c.json({
    data,
    pagination: getPaginationMeta(params, total),
  });
});
```

#### Advanced: Cursor-Based Pagination

For high-performance pagination on large datasets:

```typescript
// lib/api/schemas/cursor.ts
export const cursorPaginationQuerySchema = z.object({
  limit: z.coerce.number().int().min(1).max(100).default(20),
  cursor: z.string().optional(),  // base64-encoded { id, createdAt }
});

export const cursorPaginatedResponseSchema = <T extends z.ZodTypeAny>(itemSchema: T) =>
  z.object({
    data: z.array(itemSchema),
    nextCursor: z.string().nullable(),
    hasMore: z.boolean(),
  });

// Usage
router.openapi(listRoute, async (c) => {
  const { limit, cursor } = c.req.valid('query');
  const query = c.get('query');
  
  let where = undefined;
  if (cursor) {
    const decoded = JSON.parse(Buffer.from(cursor, 'base64').toString());
    where = lt(clients.createdAt, new Date(decoded.createdAt));
  }
  
  const data = await query
    .select()
    .from(clients)
    .where(where)
    .orderBy(desc(clients.createdAt))
    .limit(limit + 1);  // Fetch one extra to check hasMore
  
  const hasMore = data.length > limit;
  const items = hasMore ? data.slice(0, -1) : data;
  
  const nextCursor = hasMore && items.length > 0
    ? Buffer.from(JSON.stringify({ 
        id: items[items.length - 1].id,
        createdAt: items[items.length - 1].createdAt 
      })).toString('base64')
    : null;
  
  return c.json({ data: items, nextCursor, hasMore });
});
```

#### When to Use Each

- **Offset pagination (`page` + `limit`):**
  - Default for most list endpoints
  - User needs to jump to specific pages
  - Dataset is < 10,000 rows
  - Sorting by fields other than creation time

- **Cursor pagination:**
  - Infinite scroll UIs
  - Large datasets (> 10,000 rows)
  - Real-time feeds (new items appear frequently)
  - Performance is critical

### Route Structure
```typescript
// Each route file exports a single OpenAPIHono router
import { OpenAPIHono, createRoute } from '@hono/zod-openapi';
import { z } from 'zod';
import { count } from 'drizzle-orm';
import { 
  createClientSchema, 
  clientResponseSchema 
} from '~/lib/db/schema/clients.zod';  // ✅ Generated from Drizzle table
import { clients } from '~/lib/db/schema/clients';
import { paginationQuerySchema, paginatedResponseSchema } from '~/lib/api/schemas/pagination';
import { getPaginationMeta, applyPagination } from '~/lib/api/utils/pagination';

const router = new OpenAPIHono<ApiContext>();

// List clients with pagination
const listRoute = createRoute({
  method: 'get',
  path: '/',
  request: { 
    query: paginationQuerySchema  // ✅ Every list endpoint has pagination
  },
  responses: {
    200: { 
      content: { 
        'application/json': { 
          schema: paginatedResponseSchema(clientResponseSchema)  // ✅ Standard wrapper
        } 
      }, 
      description: 'List clients with pagination' 
    },
  },
});

router.openapi(listRoute, async (c) => {
  const params = c.req.valid('query');
  const query = c.get('query');  // RLS-scoped
  
  // Get total count
  const [{ total }] = await query
    .select({ total: count() })
    .from(clients);
  
  // Get paginated data
  const data = await applyPagination(
    query.select().from(clients),
    params
  );
  
  return c.json({
    data,
    pagination: getPaginationMeta(params, total),
  });
});

// Create client
const createRoute = createRoute({
  method: 'post',
  path: '/',
  request: {
    body: {
      content: {
        'application/json': { schema: createClientSchema },  // ✅ From drizzle-zod
      },
    },
  },
  responses: {
    201: {
      content: {
        'application/json': { schema: clientResponseSchema },  // ✅ From drizzle-zod
      },
      description: 'Client created',
    },
  },
});

router.openapi(createRoute, async (c) => {
  const data = c.req.valid('json');  // ✅ Typed as createClientSchema
  const query = c.get('query');
  
  const [client] = await query
    .insert(clients)
    .values(data)
    .returning();
  
  return c.json(client, 201);
});

export { router as clientsRouter };
```

### Middleware Stack (order matters)
1. `rateLimitMiddleware` — IP-based, per-route configurable
2. `apiMiddlewareSetup` — creates DB client + auth instance, attaches to context
3. `apiMiddlewareRequestLogger` — structured logging
4. `apiMiddlewareRequireAuth` — validates session via `auth.api.getSession()`, attaches user/session to context
5. Route-specific: `auditMiddleware`, `rlsMiddleware` as needed

### API Split
- **`/api/health`** — no auth, no DB
- **`/api/auth/*`** — passthrough to better-auth handler (handles sign-in, sign-up, OTP, password reset, org management)
- **`/api/v1/practitioner/*`** — requires auth + practice context, RLS as ch_admin/ch_practitioner
- **`/api/v1/client/*`** — requires auth, RLS as ch_client
- **`/api/v1/invitations`** — requires auth, no practice context needed

## 4. DATABASE PATTERNS

### Connection Pipeline
```
Cloudflare Worker
  -> Hyperdrive binding (env.DB.connectionString)
    -> Connection pool (Hyperdrive manages TLS + pooling)
      -> Neon Postgres (visitor_role login role)
        -> SET LOCAL ROLE ch_<X> (per-request RLS)
```

### Tech Stack
- **Neon Postgres** — serverless Postgres with branching
- **Cloudflare Hyperdrive** — connection pooling + edge caching between Worker and Neon
- **Drizzle ORM** — type-safe query builder + migrations
- **`drizzle-zod`** — derive Zod schemas from Drizzle table definitions
- **Row-Level Security (RLS)** — enforced at the Postgres level, not app level

### wrangler.jsonc Hyperdrive Config
```jsonc
{
  "hyperdrive": [
    { "binding": "DB", "id": "abc123..." }
  ]
}
```
The `DB` binding exposes `.connectionString` at runtime. For preview environments, the ID is a placeholder `__HYPERDRIVE_ID_REPLACE__` that CI replaces via `sed`.

### DB Client Creation (lib/db/index.ts)
```typescript
import postgres from 'postgres';
import { drizzle } from 'drizzle-orm/postgres-js';

export function createDb(connectionString: string) {
  const isHyperdrive = !connectionString.includes('.neon.tech');
  const client = postgres(connectionString, {
    ssl: isHyperdrive ? false : 'prefer',   // Hyperdrive handles TLS
    prepare: isHyperdrive ? false : true,    // Hyperdrive pools connections
    max: 1,
  });
  return drizzle(client, { schema });
}
```
**Why the detection:** Direct Neon connections (local dev, scripts) need `ssl: 'prefer'`. Hyperdrive connections MUST have `ssl: false` and `prepare: false` — Hyperdrive handles TLS internally and prepared statements are connection-scoped.

### Database Initialization (scripts/db-init-roles.ts)
Run once per Neon project to set up the role hierarchy:
```sql
-- 1. Create login role (used by Hyperdrive)
CREATE ROLE visitor_role LOGIN PASSWORD '...';

-- 2. Create data roles (NOLOGIN — used via SET LOCAL ROLE)
CREATE ROLE ch_admin NOLOGIN;
CREATE ROLE ch_practitioner NOLOGIN;
CREATE ROLE ch_client NOLOGIN;
CREATE ROLE ch_system NOLOGIN;

-- 3. Grant data roles to login role
GRANT ch_admin, ch_practitioner, ch_client, ch_system TO visitor_role;
GRANT USAGE ON SCHEMA public TO visitor_role, ch_admin, ch_practitioner, ch_client, ch_system;

-- 4. Grant table permissions to data roles
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO ch_admin;
-- (similar for other roles with appropriate restrictions)

-- 5. Set default privileges for future tables
ALTER DEFAULT PRIVILEGES IN SCHEMA public
  GRANT SELECT, INSERT, UPDATE, DELETE ON TABLES TO ch_admin;
```

### Schema Conventions
- **Integer identity primary keys** (`serial` or `identity`), optional public UUID for external exposure
- **`created_at` / `updated_at`** on every table with `DEFAULT now()`
- **One file per table** in `lib/db/schema/` — name matches table (`clients.ts` for `clients`)
- **Barrel export** from `lib/db/schema/index.ts`
- **Soft delete** via RLS policy (restrictive policy excludes `deleted_at IS NOT NULL`), not application-level WHERE clause
- **Foreign keys** use the integer PK, not the UUID
- **Auth schema generated** by `npm run gen:auth` — never hand-edit `lib/db/schema/auth.ts`

### RLS Setup
Four Postgres roles:
- `ch_admin` — full CRUD within their org
- `ch_practitioner` — CRUD on assigned clients within their org
- `ch_client` — read-only on own records
- `ch_system` — full CRUD, no org restriction (for background jobs, webhooks)

Login role (Hyperdrive connection) is granted all four. App code does `SET LOCAL ROLE ch_<X>` within a transaction based on the user's org membership.

### Migrations
```bash
npm run db:generate    # Drizzle Kit generates SQL migration
npm run db:migrate     # Applies pending migrations
npm run gen:auth       # Regenerates auth schema from better-auth config
```
Migration files in `drizzle/` — never hand-edit. If you need a custom migration, use Drizzle Kit's `custom` option.

### Drizzle-Zod Schema Generation

**CRITICAL RULE:** Use `drizzle-zod` helpers to generate Zod schemas from Drizzle tables. Never manually write schemas that duplicate table structure.

#### Installation
```bash
npm install drizzle-zod
```

#### Pattern: Single Source of Truth

```typescript
// ❌ ANTI-PATTERN: Manual schema duplication
import { z } from 'zod';

const ClientSchema = z.object({
  id: z.number(),
  orgId: z.number(),
  name: z.string(),
  email: z.string().email(),
  createdAt: z.date(),
  updatedAt: z.date(),
  // ⚠️ If table changes, this breaks silently!
});

// ✅ CORRECT: Generate from Drizzle table
import { createInsertSchema, createSelectSchema } from 'drizzle-zod';
import { clients } from './schema/clients';

// Full schemas
export const selectClientSchema = createSelectSchema(clients);
export const insertClientSchema = createInsertSchema(clients);

// API-specific refinements
export const createClientSchema = insertClientSchema
  .omit({ id: true, createdAt: true, updatedAt: true }) // auto-generated fields
  .extend({
    email: z.string().email().toLowerCase(), // additional validation
  });

export const updateClientSchema = insertClientSchema
  .omit({ id: true, orgId: true, createdAt: true, updatedAt: true })
  .partial(); // all fields optional for PATCH

export const clientResponseSchema = selectClientSchema
  .omit({ orgId: true }); // don't expose internal IDs in API responses
```

#### Available Helpers

```typescript
import { 
  createInsertSchema,  // For INSERT operations (excludes auto-generated fields)
  createSelectSchema,  // For SELECT results (all fields)
  createUpdateSchema,  // For UPDATE operations (like insert but may differ)
} from 'drizzle-zod';

const insertSchema = createInsertSchema(clients);    // Omits serial/identity PKs
const selectSchema = createSelectSchema(clients);    // Includes all columns
const updateSchema = createUpdateSchema(clients);    // Same as insert by default
```

#### Refining Generated Schemas

```typescript
import { createInsertSchema } from 'drizzle-zod';
import { clients } from './schema/clients';
import { z } from 'zod';

// Add custom validation beyond what Drizzle knows
export const insertClientSchema = createInsertSchema(clients, {
  email: z.string().email().toLowerCase(),           // additional transform
  name: z.string().min(2).max(100),                  // length constraints
  phone: z.string().regex(/^\d{10}$/).optional(),    // format validation
});

// Create endpoint-specific variants
export const createClientSchema = insertClientSchema
  .omit({ id: true, createdAt: true, updatedAt: true });

export const updateClientSchema = insertClientSchema
  .omit({ id: true, orgId: true, createdAt: true, updatedAt: true })
  .partial();  // all fields optional

export const patchClientNameSchema = insertClientSchema
  .pick({ name: true });  // only allow name updates on this endpoint
```

#### Usage in API Routes

```typescript
// lib/api/routes/practitioner/clients.ts
import { createRoute } from '@hono/zod-openapi';
import { 
  createClientSchema, 
  updateClientSchema, 
  clientResponseSchema 
} from '~/lib/db/schema/clients.zod';

const createRoute = createRoute({
  method: 'post',
  path: '/clients',
  request: {
    body: {
      content: {
        'application/json': { schema: createClientSchema },
      },
    },
  },
  responses: {
    201: {
      content: {
        'application/json': { schema: clientResponseSchema },
      },
      description: 'Client created',
    },
  },
});

router.openapi(createRoute, async (c) => {
  const data = c.req.valid('json');  // ✅ Typed as createClientSchema
  const query = c.get('query');

  const [client] = await query
    .insert(clients)
    .values(data)
    .returning();

  return c.json(client, 201);  // ✅ Typed as clientResponseSchema
});
```

#### Why This Matters

**Single source of truth:**
```
Drizzle table definition
  ↓ (drizzle-zod generates)
Zod schemas
  ↓ (used in createRoute)
OpenAPI spec
  ↓ (npm run gen:api)
TypeScript types
  ↓ (used in frontend)
Fully typed API calls
```

**Change the table, everything updates:**
1. Add column to Drizzle schema
2. Run `npm run db:generate && npm run db:migrate`
3. Zod schemas auto-update (they're derived, not manual)
4. Run `npm run gen:api`
5. TypeScript errors show every API call that needs updating

**Without drizzle-zod:**
- Change table → manually update Zod schema → easy to forget fields
- Schemas drift from tables over time
- Runtime errors from schema mismatches

#### File Organization

```
lib/db/schema/
  clients.ts           # Drizzle table definition
  clients.zod.ts       # Generated Zod schemas + API refinements
  index.ts             # Barrel exports
```

```typescript
// clients.zod.ts
import { createInsertSchema, createSelectSchema } from 'drizzle-zod';
import { clients } from './clients';
import { z } from 'zod';

export const selectClientSchema = createSelectSchema(clients);
export const insertClientSchema = createInsertSchema(clients);

export const createClientSchema = insertClientSchema
  .omit({ id: true, createdAt: true, updatedAt: true });

export const updateClientSchema = insertClientSchema
  .omit({ id: true, orgId: true, createdAt: true, updatedAt: true })
  .partial();

export const clientResponseSchema = selectClientSchema
  .omit({ orgId: true });  // don't leak internal IDs
```

**Import pattern:**
```typescript
// ✅ Import from .zod file for API routes
import { createClientSchema } from '~/lib/db/schema/clients.zod';

// ✅ Import table from base file for queries
import { clients } from '~/lib/db/schema/clients';
```

## 5. AUTH PATTERNS (better-auth)

### Key Files
```
./better-auth.config.ts    # CLI config — used by `npm run gen:auth` to generate schema
src/lib/auth/
  better-auth.ts           # Server-side auth factory (betterAuthOptions + createAuth)
  auth-client.ts           # Browser-side client (createAuthClient with plugins)
  auth.functions.ts        # TanStack Start server functions (getSession, ensureSession)
  auth-hooks.ts            # React hook (useAuth — wraps route context + signOut)
  types.ts                 # Auth, AuthSession, AuthUser type exports
```

### Server Auth Factory (lib/auth/better-auth.ts)
Two exports:
1. **`betterAuthOptions`** — shared config (plugins, email settings, ID generation). Used by both the Worker runtime and the CLI config.
2. **`createAuth(db, env, ctx)`** — creates a fully-configured better-auth instance per-request. Adds:
   - Trusted origins
   - Organization plugin with invitation hooks (e.g., auto-create care team assignment on client invite accept)
   - Cookie prefix per environment (`ch_${env.APP_ENV}`)
   - KV-backed secondary storage for session caching
   - Background task handler via `ctx.waitUntil`

### CLI Config (./better-auth.config.ts)
Used by `npm run gen:auth` to generate the auth schema at `src/lib/db/schema/auth.ts`:
```bash
npx @better-auth/cli@latest generate --config ./better-auth.config.ts --output ./src/lib/db/schema/auth.ts
```
This reads `betterAuthOptions` + a direct DB connection to introspect/generate the Drizzle schema for auth tables (user, session, account, verification, organization, member, invitation).

### Browser Client (lib/auth/auth-client.ts)
```typescript
import { createAuthClient } from 'better-auth/react';
export const authClient = createAuthClient({
  plugins: [organizationClient(), adminClient(), emailOTPClient(), jwtClient()],
});
```
Used in components for: `authClient.signIn.email()`, `authClient.signUp.email()`, `authClient.signOut()`, `authClient.useSession()`, `authClient.useListOrganizations()`, etc.

### TanStack Start Server Functions (lib/auth/auth.functions.ts)
Uses TanStack Start's `createMiddleware` + `createServerFn` pattern:
```typescript
const authMiddleware = createMiddleware().server(async ({ next, context }) => {
  const { env, executionCtx } = context;
  const db = createDb(env.DB.connectionString);
  const auth = createAuth(db, env, executionCtx);
  const session = await auth.api.getSession({ headers: getRequestHeaders() });
  return next({ context: { session } });
});

export const getSession = createServerFn({ method: 'GET' })
  .middleware([authMiddleware])
  .handler(async ({ context }) => context.session);
```
**Critical:** This is how SSR routes get the session — through TanStack Start's server function system, NOT through the Hono API middleware. The Hono API has its own auth middleware (`apiMiddlewareRequireAuth`) that uses the same `auth.api.getSession()` but from the Hono context.

### Session Flow (Two Paths)
**SSR path (page loads):**
1. Root route `beforeLoad` calls `getSession()` server function
2. Session + user injected into TanStack Router context
3. Protected routes check `context.session` in `beforeLoad`, redirect to `/auth/sign-in` if missing

**API path (fetch calls):**
1. `apiMiddlewareSetup` creates auth instance from Hono bindings
2. `apiMiddlewareRequireAuth` calls `auth.api.getSession({ headers })` with the request headers
3. Session attached to `c.set('session', session)`

### Plugins
- **organization** — multi-tenancy (practices), invitation system, role-based membership
- **emailOTP** — email verification codes for sign-up and sign-in
- **admin** — admin-level user management
- **jwt** — JWT token support

### Auth Pages
Split-screen layout: left form panel + right brand panel with AuthLayout component. Auth flow includes email+password with OTP email verification. Each page (sign-in, sign-up, forgot-password, reset-password) focuses only on form logic.

## 5. FRONTEND PATTERNS

### Typed API Client
```typescript
// lib/api/client.ts
import createFetchClient from 'openapi-fetch';
import type { paths } from './v1';

const apiClient = createFetchClient<paths>({ baseUrl: '/' });
```

### Typed React Query Hooks
```typescript
// Via openapi-react-query
import createClient from 'openapi-react-query';
import type { paths } from './v1';

const api = createClient<paths>(apiClient);

// Usage in components:
const { data } = api.useQuery('get', '/api/v1/practitioner/clients');
```
**Delete manual fetch hooks.** The typed client generates hooks from the OpenAPI spec. No `src/hooks/use-clients.ts` needed.

### Component Library: shadcn (base-nova style)
- Install via `npx shadcn@latest add <component>`
- Theme in `styles.css` using CSS custom properties (oklch colors)
- Never hardcode hex colors — use `bg-background`, `text-foreground`, `text-primary`, `border-border`, etc.
- Customize via CSS variables, not by editing component files

### Application Shell (Practitioner Portal)
Uses shadcn Sidebar (collapsible icon mode):
- SidebarProvider > Sidebar + SidebarInset
- SidebarHeader: app wordmark
- SidebarContent: nav groups with lucide icons
- SidebarFooter: user avatar + dropdown (profile, settings, sign out)
- SidebarInset header: trigger + separator + breadcrumbs
- Main content: `<Outlet />` in padded container

### Fonts
- **Sans:** Geist Variable (primary body/UI font)
- **Serif:** Newsreader Variable (wordmarks, editorial accents only)
- **Mono:** Geist Mono (code, data)

Never use Inter. Never use serif for dashboard UI.

## 7. DEPLOYMENT & CI

### wrangler.jsonc Structure
```jsonc
{
  "name": "my-app",
  "main": "./src/server.ts",
  "compatibility_date": "2026-03-01",
  "compatibility_flags": ["nodejs_compat"],
  "hyperdrive": [
    { "binding": "DB", "id": "abc123-production-hyperdrive-id" }
  ],
  "r2_buckets": [
    { "binding": "MEDIA_BUCKET", "bucket_name": "my-app-media" }
  ],
  "kv_namespaces": [
    { "binding": "KV", "id": "abc123-kv-id" }
  ],
  "env": {
    "preview": {
      "vars": { "APP_ENV": "preview" },
      "hyperdrive": [
        { "binding": "DB", "id": "__HYPERDRIVE_ID_REPLACE__" }
      ],
      // R2 and KV shared with production
      "r2_buckets": [
        { "binding": "MEDIA_BUCKET", "bucket_name": "my-app-media" }
      ]
    }
  }
}
```
**Key pattern:** The `preview` env uses `__HYPERDRIVE_ID_REPLACE__` as a placeholder. CI replaces it with the actual Hyperdrive config ID via `sed` during deployment.

### Preview Workflow (.github/workflows/preview.yml)
Triggered on PR open/sync/reopen/close. Full pipeline:

```yaml
# 1. Create Neon branch (idempotent — reuses if exists)
- uses: neondatabase/create-branch-action@v6
  with:
    branch_name: preview/pr-${{ github.event.pull_request.number }}
    role: neondb_owner
    database: neondb

# 2. Reset branch if it already existed (re-fork from parent)
- uses: neondatabase/reset-branch-action@v1
  if: steps.neon.outputs.created == 'false'

# 3. Get visitor role connection string for Hyperdrive
- run: |
    VISITOR_URL=$(npx neonctl connection-string "preview/pr-$PR_NUM" \
      --role-name visitor_role --database-name neondb --api-key "$NEON_API_KEY")
    # Fix connection string params for pg driver compatibility
    VISITOR_URL=$(echo "$VISITOR_URL" | sed 's/[&?]channel_binding=require//g; s/sslmode=require/sslmode=verify-full/g')

# 4. Run migrations on the branch
- run: npm run db:migrate
  env:
    DATABASE_URL: ${{ steps.neon.outputs.db_url }}

# 5. Seed database (idempotent upsert)
- run: npm run db:seed
  env:
    DATABASE_URL: ${{ steps.neon.outputs.db_url }}

# 6. Create/update Hyperdrive config pointing to branch
- run: |
    HD_NAME="myapp-preview-pr-${PR_NUM}"
    EXISTING_ID=$(npx wrangler hyperdrive list | grep -A1 "$HD_NAME" | grep -oE '[a-f0-9-]{8,}' || true)
    if [ -n "$EXISTING_ID" ]; then
      npx wrangler hyperdrive update "$EXISTING_ID" --connection-string="$VISITOR_URL"
    else
      npx wrangler hyperdrive create "$HD_NAME" --connection-string="$VISITOR_URL"
    fi

# 7. Replace placeholder IDs in wrangler config
- run: |
    sed -i 's/__HYPERDRIVE_ID_REPLACE__/${{ steps.hyperdrive.outputs.id }}/' wrangler.jsonc

# 8. Build + deploy preview worker
- uses: cloudflare/wrangler-action@v3
  with:
    command: deploy --env preview --name "myapp-preview-pr-${{ github.event.pull_request.number }}"

# 9. Comment preview URL on PR
- uses: actions/github-script@v7  # Posts "Preview deployed: https://..."

# Teardown (on PR close):
# - Delete Hyperdrive config
# - Delete preview worker
# - Delete Neon branch
```

### CI Pipeline (.github/workflows/ci.yml)
Runs on every PR push:
1. Format check (`npx biome check`)
2. Type check (`npx tsc --noEmit`)
3. Unit tests with 100% coverage (`npx vitest run --coverage`)
4. E2E tests (`npx playwright test`)
5. Build (`npm run build`)

CI is the gate. PRs cannot merge if CI fails.

### Production Deployment (.github/workflows/deploy.yml)
Triggered on push to main:
1. Run CI checks
2. Run migrations against production Neon
3. Deploy worker via `wrangler deploy`

### Secrets & Variables (GitHub)
**Secrets:** `NEON_API_KEY`, `CLOUDFLARE_API_TOKEN`, `BETTER_AUTH_SECRET`
**Variables:** `NEON_PROJECT_ID`, `CLOUDFLARE_ACCOUNT_ID`
**Worker secrets** (set via `wrangler secret put`): `BETTER_AUTH_SECRET`, `APP_URL`, `STRIPE_SECRET_KEY`

## 7. WHEN SCAFFOLDING A NEW PROJECT

Execute in this order:
1. `npm create cloudflare@latest` or `cargo init` (per CLAUDE.md language rules)
2. Configure biome, vitest, husky + lint-staged
3. Initialize Drizzle with Neon connection
4. Create `db-init-roles.ts` script for RLS role setup
5. Set up better-auth with email OTP
6. Create the API app with OpenAPIHono + health route
7. Wire TanStack Start with root route + auth guard
8. Install shadcn components (button, input, label, card minimum)
9. Build auth pages with AuthLayout
10. Build application shell with Sidebar
11. Set up CI + preview workflow
12. Generate initial `v1.d.ts` with `gen:api`

## 8. WHEN REFACTORING AN EXISTING PROJECT

Assess in this order:
1. **API typing**: Are routes using plain Hono or OpenAPIHono? Convert to `createRoute()` + Zod schemas.
2. **Auth**: Is session validation duplicated? Unify into single middleware. Is auth provider modern (better-auth) or legacy (Clerk/PropelAuth/Neon Auth)?
3. **DB roles**: Are queries running with proper RLS roles or bypassing security? Every authenticated query should go through `SET LOCAL ROLE`.
4. **Frontend hooks**: Are there manual fetch wrappers? Replace with typed `openapi-react-query` hooks.
5. **Hardcoded colors**: Replace all hex with shadcn theme tokens.
6. **Directory structure**: Reorganize to match the taxonomy above.
7. **Preview deploys**: Set up Neon branching + Hyperdrive preview workflow if missing.

## 9. ANTI-PATTERNS (things that caused production bugs)

- **`ssl: true` with Hyperdrive** — breaks connection. Hyperdrive handles TLS.
- **`prepare: true` with Hyperdrive** — prepared statements are connection-scoped, Hyperdrive pools.
- **Hand-editing `v1.d.ts`** — drifts from actual API. Always regenerate.
- **Hardcoded hex in components** — breaks dark mode and theme switching.
- **Orgs vs Practices naming split** — pick one term and use it everywhere in the API. DB can differ.
- **Manual fetch hooks** — duplicate work, miss type updates. Use the generated typed client.
- **Missing seed step in preview workflow** — empty tables on preview = broken preview.
- **Neon Auth cookie forwarding without the correct pattern** — must forward the raw cookie header, not use browser client server-side.
- **Duplicated auth middleware** — session validation in two files inevitably drifts. Single source of truth.
- **Unbounded list endpoints** — returning all rows without pagination causes timeouts and crashes on production data. Every list endpoint MUST have `page` + `limit` parameters and return paginated responses.
- **Manual Zod schemas duplicating Drizzle tables** — use `createInsertSchema`/`createSelectSchema` from drizzle-zod. Manual schemas drift from tables and break silently.
