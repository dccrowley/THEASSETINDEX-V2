# Lightweight SaaS Architecture Specification

**Purpose**: Cursor AI development reference for MVP and personal projects
**Last Updated**: 2025-11-16
**Target**: Solo developers, prototypes, proof of concept builds
**Cost Target**: £0-10/month

---

## Core Technology Stack

### Primary Components

- **Frontend**: Next.js 14+ (App Router)
- **Auth**: Clerk (Hobby tier) - Optional: NextAuth.js for full local control
- **Database**: SQLite (local) OR Supabase (free tier) OR Turso (edge SQLite)
- **NoSQL Option**: Redis (Upstash free tier) for caching and sessions
- **Monitoring**: Sentry (Developer tier, 5k errors/month)
- **Persistent Memory**: Claude AI Projects with conversation context

---

## Database Options

### Option 1: SQLite (Local Development)

**Best for**: Prototypes, offline-first apps, single-instance deployments

```typescript
// lib/db/sqlite.ts
import Database from 'better-sqlite3';
import path from 'path';

const db = new Database(path.join(process.cwd(), 'data', 'app.db'));

// Enable WAL mode for better concurrency
db.pragma('journal_mode = WAL');

export default db;
```

**Advantages**:
- Zero cost
- No network latency
- Perfect for Cursor local development
- Easy backup (single file)

**Limitations**:
- Single-writer constraint
- No horizontal scaling
- Must handle migrations manually

### Option 2: Turso (Edge SQLite)

**Best for**: Global distribution, serverless deployments

```typescript
// lib/db/turso.ts
import { createClient } from '@libsql/client';

export const turso = createClient({
  url: process.env.TURSO_DATABASE_URL!,
  authToken: process.env.TURSO_AUTH_TOKEN,
});
```

**Advantages**:
- SQLite syntax and simplicity
- Edge deployment (low latency globally)
- Free tier: 500 databases, 9GB storage
- Built-in replication

**Limitations**:
- Newer ecosystem
- Limited real-time features

### Option 3: Supabase (PostgreSQL)

**Best for**: Need real-time features, future scaling path

```typescript
// lib/db/supabase.ts
import { createClient } from '@supabase/supabase-js';

export const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
);
```

**Advantages**:
- Real-time subscriptions
- Built-in auth (can replace Clerk)
- Row Level Security
- Free tier: 500MB database

**Limitations**:
- Network dependency
- More complex than SQLite

### NoSQL Supplement: Upstash Redis

**Use cases**: Session storage, rate limiting, caching, pub/sub

```typescript
// lib/db/redis.ts
import { Redis } from '@upstash/redis';

export const redis = new Redis({
  url: process.env.UPSTASH_REDIS_REST_URL!,
  token: process.env.UPSTASH_REDIS_REST_TOKEN!,
});

// Example: Cache expensive queries
export async function getCachedData(key: string, fetchFn: () => Promise<any>) {
  const cached = await redis.get(key);
  if (cached) return cached;
  
  const data = await fetchFn();
  await redis.set(key, data, { ex: 3600 }); // 1 hour TTL
  return data;
}
```

**Free tier**: 10k commands/day, 256MB storage

---

## Project Structure for Cursor

```
/app
  /api
    /auth           # Authentication endpoints
    /data           # CRUD operations
  /(dashboard)      # Protected routes
  /(public)         # Marketing pages
  
/lib
  /db
    /client.ts      # Database client initialization
    /schema.ts      # Database schema definitions
    /migrations     # SQL migration files
  /auth
    /middleware.ts  # Route protection
  /utils
    /memory.ts      # Claude persistent memory helpers
    
/data                # SQLite database files (gitignored)
  app.db
  app.db-shm
  app.db-wal
  
/tests
  /unit             # Vitest tests
  
/prompts            # Claude AI prompt templates
  context.md        # Project context for Claude
  patterns.md       # Code patterns for Cursor
```

---

## Schema Definition (SQLite/Turso)

```sql
-- migrations/001_initial.sql
-- Users table
CREATE TABLE IF NOT EXISTS users (
  id TEXT PRIMARY KEY,
  clerk_user_id TEXT UNIQUE NOT NULL,
  email TEXT NOT NULL,
  name TEXT,
  role TEXT DEFAULT 'user',
  created_at INTEGER DEFAULT (strftime('%s', 'now')),
  updated_at INTEGER DEFAULT (strftime('%s', 'now'))
);

CREATE INDEX idx_users_clerk_id ON users(clerk_user_id);
CREATE INDEX idx_users_email ON users(email);

-- Projects table
CREATE TABLE IF NOT EXISTS projects (
  id TEXT PRIMARY KEY,
  user_id TEXT NOT NULL,
  title TEXT NOT NULL,
  description TEXT,
  status TEXT DEFAULT 'active',
  created_at INTEGER DEFAULT (strftime('%s', 'now')),
  updated_at INTEGER DEFAULT (strftime('%s', 'now')),
  FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);

CREATE INDEX idx_projects_user_id ON projects(user_id);
CREATE INDEX idx_projects_status ON projects(status);

-- Session cache (if not using Redis)
CREATE TABLE IF NOT EXISTS sessions (
  id TEXT PRIMARY KEY,
  user_id TEXT NOT NULL,
  data TEXT NOT NULL, -- JSON string
  expires_at INTEGER NOT NULL,
  FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
);

CREATE INDEX idx_sessions_user_id ON sessions(user_id);
CREATE INDEX idx_sessions_expires ON sessions(expires_at);
```

---

## Persistent Memory Integration

### Claude AI Memory Context

**Strategy**: Use Claude Projects to maintain development context across sessions

```markdown
<!-- prompts/context.md -->
# Project Context for Claude

## Architecture
- Next.js 14 App Router
- SQLite database (better-sqlite3)
- Clerk authentication
- Upstash Redis for caching

## Database Schema
[Include current schema here - update as it evolves]

## Coding Patterns
- Use server actions for mutations
- Client components only when interactivity required
- All database queries wrapped in try-catch with Sentry
- TypeScript strict mode enabled

## Current Sprint
[Update with current features being built]

## Known Issues
[Track bugs and technical debt]
```

### Memory Helper Functions

```typescript
// lib/utils/memory.ts
/**
 * Store development context for Claude to reference
 * Useful for tracking architecture decisions and patterns
 */

export interface ProjectMemory {
  decisions: ArchitectureDecision[];
  patterns: CodePattern[];
  issues: KnownIssue[];
}

export interface ArchitectureDecision {
  id: string;
  date: string;
  decision: string;
  rationale: string;
  alternatives: string[];
}

export interface CodePattern {
  name: string;
  description: string;
  example: string;
  when_to_use: string;
}

export interface KnownIssue {
  id: string;
  description: string;
  workaround?: string;
  tracked_at: string;
}

// Store in SQLite for persistence
export async function saveMemoryContext(db: any, memory: ProjectMemory) {
  const stmt = db.prepare(`
    INSERT OR REPLACE INTO project_memory (id, data, updated_at)
    VALUES (?, ?, ?)
  `);
  
  stmt.run('main', JSON.stringify(memory), Date.now());
}

export async function loadMemoryContext(db: any): Promise<ProjectMemory | null> {
  const row = db.prepare('SELECT data FROM project_memory WHERE id = ?').get('main');
  return row ? JSON.parse(row.data) : null;
}
```

### Memory Schema

```sql
-- migrations/002_memory.sql
CREATE TABLE IF NOT EXISTS project_memory (
  id TEXT PRIMARY KEY,
  data TEXT NOT NULL, -- JSON blob
  updated_at INTEGER DEFAULT (strftime('%s', 'now'))
);

CREATE TABLE IF NOT EXISTS architecture_decisions (
  id TEXT PRIMARY KEY,
  title TEXT NOT NULL,
  decision TEXT NOT NULL,
  rationale TEXT NOT NULL,
  alternatives TEXT, -- JSON array
  decided_at INTEGER DEFAULT (strftime('%s', 'now'))
);
```

---

## Authentication Setup

### Option 1: Clerk (Managed)

```typescript
// middleware.ts
import { clerkMiddleware, createRouteMatcher } from '@clerk/nextjs/server';

const isProtectedRoute = createRouteMatcher(['/dashboard(.*)']);

export default clerkMiddleware((auth, req) => {
  if (isProtectedRoute(req)) auth().protect();
});

export const config = {
  matcher: ['/((?!.*\\..*|_next).*)', '/', '/(api|trpc)(.*)'],
};
```

### Option 2: NextAuth.js (Self-hosted)

```typescript
// app/api/auth/[...nextauth]/route.ts
import NextAuth from 'next-auth';
import GoogleProvider from 'next-auth/providers/google';
import { db } from '@/lib/db/client';

const handler = NextAuth({
  providers: [
    GoogleProvider({
      clientId: process.env.GOOGLE_CLIENT_ID!,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET!,
    }),
  ],
  callbacks: {
    async session({ session, token }) {
      // Store session in SQLite or Redis
      const user = db.prepare('SELECT * FROM users WHERE email = ?')
        .get(session.user?.email);
      
      session.userId = user?.id;
      return session;
    },
  },
});

export { handler as GET, handler as POST };
```

---

## Data Flow Architecture

### Request Flow (SQLite)

1. User requests protected route
2. Middleware validates Clerk/NextAuth session
3. Server component/action queries SQLite directly (no network hop)
4. Data transformed and returned to client
5. Errors captured by Sentry

### Caching Strategy (with Redis)

```typescript
// lib/db/cache.ts
import { redis } from './redis';
import { db } from './client';

export async function getUserWithCache(userId: string) {
  // Try cache first
  const cached = await redis.get(`user:${userId}`);
  if (cached) return JSON.parse(cached as string);
  
  // Fall back to database
  const user = db.prepare('SELECT * FROM users WHERE id = ?').get(userId);
  
  // Cache for 5 minutes
  await redis.set(`user:${userId}`, JSON.stringify(user), { ex: 300 });
  
  return user;
}

export async function invalidateUserCache(userId: string) {
  await redis.del(`user:${userId}`);
}
```

---

## Environment Variables

```bash
# .env.local

# Clerk Authentication
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_test_xxx
CLERK_SECRET_KEY=sk_test_xxx

# Database (choose one)
DATABASE_PATH=./data/app.db              # SQLite
TURSO_DATABASE_URL=libsql://xxx          # Turso
TURSO_AUTH_TOKEN=xxx

# Redis Cache (optional)
UPSTASH_REDIS_REST_URL=https://xxx
UPSTASH_REDIS_REST_TOKEN=xxx

# Monitoring
NEXT_PUBLIC_SENTRY_DSN=https://xxx
SENTRY_AUTH_TOKEN=xxx

# Development
NODE_ENV=development
```

---

## Cost Breakdown (Monthly)

### Zero-Cost Option

- **Frontend**: Vercel Hobby (Free, 100GB bandwidth)
- **Auth**: Clerk Hobby (Free, 10k MAU) OR NextAuth (Free, self-hosted)
- **Database**: SQLite (Free, local file) OR Turso (Free, 500 DBs)
- **Cache**: Upstash Redis (Free, 10k commands/day)
- **Monitoring**: Sentry Developer (Free, 5k errors/month)

**Total**: £0/month

### Low-Cost Option (Better limits)

- **Frontend**: Vercel Hobby (Free)
- **Auth**: Clerk Hobby (Free)
- **Database**: Turso Starter (£7/month, 1000 DBs, 50GB)
- **Cache**: Upstash Redis Pay-as-go (~£1/month for hobby project)
- **Monitoring**: Sentry Developer (Free)

**Total**: £8/month

---

## Security Model (Lightweight)

### Authentication
- Clerk handles OAuth and session management
- JWT tokens validated in middleware
- No passwords stored (OAuth only)

### Database Access
- All queries server-side only (no client DB access)
- Prepared statements prevent SQL injection
- User ID from auth token filters all queries

### Data Protection
- SQLite file permissions restricted (600)
- Sensitive data encrypted at application level
- API routes protected by middleware
- HTTPS enforced via Vercel

### Basic GDPR Compliance
- User data deletion endpoint
- Data export functionality
- Cookie consent banner
- Privacy policy page

---

## Testing Strategy

### Unit Tests (Vitest)

```typescript
// tests/unit/db.test.ts
import { describe, it, expect, beforeEach } from 'vitest';
import Database from 'better-sqlite3';
import { createUser, getUser } from '@/lib/db/users';

describe('User Database Operations', () => {
  let testDb: Database.Database;
  
  beforeEach(() => {
    testDb = new Database(':memory:');
    // Run migrations
    testDb.exec(`
      CREATE TABLE users (
        id TEXT PRIMARY KEY,
        email TEXT NOT NULL
      )
    `);
  });
  
  it('should create and retrieve user', () => {
    const user = createUser(testDb, {
      id: '1',
      email: 'test@example.com'
    });
    
    const retrieved = getUser(testDb, '1');
    expect(retrieved.email).toBe('test@example.com');
  });
});
```

### Integration Tests

- Test API routes with mock auth
- Verify database constraints
- Check caching behaviour

### Manual Testing Focus

- Critical user flows
- Payment integration (if applicable)
- Mobile responsiveness

---

## Cursor-Specific Development Patterns

### API Route Template

```typescript
// app/api/data/route.ts
import { auth } from '@clerk/nextjs/server';
import { db } from '@/lib/db/client';
import * as Sentry from '@sentry/nextjs';

export async function GET(request: Request) {
  try {
    const { userId } = await auth();
    if (!userId) {
      return Response.json({ error: 'Unauthorized' }, { status: 401 });
    }

    // SQLite query with prepared statement
    const stmt = db.prepare(`
      SELECT * FROM projects 
      WHERE user_id = ? 
      ORDER BY created_at DESC
    `);
    
    const projects = stmt.all(userId);

    return Response.json({ projects });
  } catch (error) {
    Sentry.captureException(error);
    console.error('API Error:', error);
    return Response.json(
      { error: 'Internal server error' },
      { status: 500 }
    );
  }
}
```

### Server Action Template

```typescript
// app/actions/projects.ts
'use server';

import { auth } from '@clerk/nextjs/server';
import { db } from '@/lib/db/client';
import { revalidatePath } from 'next/cache';
import { nanoid } from 'nanoid';

export async function createProject(formData: FormData) {
  const { userId } = await auth();
  if (!userId) throw new Error('Unauthorized');

  const title = formData.get('title') as string;
  const description = formData.get('description') as string;

  const stmt = db.prepare(`
    INSERT INTO projects (id, user_id, title, description)
    VALUES (?, ?, ?, ?)
  `);

  stmt.run(nanoid(), userId, title, description);
  
  revalidatePath('/dashboard');
  return { success: true };
}
```

---

## Migration Management

### Running Migrations

```typescript
// lib/db/migrate.ts
import Database from 'better-sqlite3';
import fs from 'fs';
import path from 'path';

export function runMigrations(db: Database.Database) {
  const migrationsDir = path.join(process.cwd(), 'lib/db/migrations');
  const files = fs.readdirSync(migrationsDir).sort();
  
  // Create migrations table
  db.exec(`
    CREATE TABLE IF NOT EXISTS migrations (
      id INTEGER PRIMARY KEY AUTOINCREMENT,
      filename TEXT UNIQUE NOT NULL,
      applied_at INTEGER DEFAULT (strftime('%s', 'now'))
    )
  `);
  
  const applied = db.prepare('SELECT filename FROM migrations')
    .all()
    .map((row: any) => row.filename);
  
  for (const file of files) {
    if (!file.endsWith('.sql') || applied.includes(file)) continue;
    
    console.log(`Applying migration: ${file}`);
    const sql = fs.readFileSync(path.join(migrationsDir, file), 'utf-8');
    db.exec(sql);
    db.prepare('INSERT INTO migrations (filename) VALUES (?)').run(file);
  }
}
```

### Initial Setup Script

```typescript
// scripts/setup.ts
import Database from 'better-sqlite3';
import { runMigrations } from '@/lib/db/migrate';
import path from 'path';
import fs from 'fs';

const dataDir = path.join(process.cwd(), 'data');
if (!fs.existsSync(dataDir)) {
  fs.mkdirSync(dataDir, { recursive: true });
}

const db = new Database(path.join(dataDir, 'app.db'));
db.pragma('journal_mode = WAL');

runMigrations(db);
console.log('Database setup complete');
db.close();
```

---

## Deployment Options

### Vercel (Recommended for Next.js)

- Free hobby tier
- Zero configuration
- Automatic HTTPS
- Edge functions support

**Note**: SQLite requires file system, use Turso for Vercel deployments

### VPS (for SQLite)

- DigitalOcean/Hetzner: £4-8/month
- Full control over file system
- Run SQLite directly
- Manual deployment setup

---

## Quick Start Checklist

- [ ] Clone starter repository or initialize Next.js
- [ ] Choose database option (SQLite/Turso/Supabase)
- [ ] Set up Clerk or NextAuth authentication
- [ ] Configure environment variables
- [ ] Run initial database migrations
- [ ] Set up Sentry error tracking
- [ ] (Optional) Configure Redis caching
- [ ] Create initial Claude Project with context.md
- [ ] Test authentication flow
- [ ] Deploy to Vercel or VPS

---

## When to Upgrade to Heavyweight

**Upgrade if you:**
- Exceed 1,000 monthly active users
- Need team collaboration features
- Require audit trails and compliance
- Handle sensitive customer data
- Need guaranteed uptime SLAs
- Require automated backups
- Scale beyond single-region deployment

**Migration path**: SQLite → Turso → Supabase → Self-hosted PostgreSQL
