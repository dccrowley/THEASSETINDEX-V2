# Heavyweight SaaS Architecture Specification

**Purpose**: Cursor AI development reference for production-ready applications
**Last Updated**: 2025-11-16
**Target**: Production deployments, customer-facing applications, team projects
**Cost Target**: £75-200/month base + usage

---

## Core Technology Stack

### Primary Components

- **Frontend**: Next.js 14+ (App Router)
- **Auth**: Clerk Pro (with organization support)
- **Database**: Supabase Pro (PostgreSQL with real-time)
- **Cache Layer**: Upstash Redis (Pay-as-you-go)
- **Monitoring**: Sentry Team (with performance monitoring)
- **CI/CD**: GitHub Actions
- **Testing**: Vitest + Playwright
- **Backup**: Supabase automated + S3 archive
- **Email**: Resend (transactional email)

---

## Project Structure for Cursor

```
/app
  /[locale]              # i18n support
    /api
      /v1                # Versioned API routes
        /users
        /organizations
        /webhooks
      /auth              # Clerk webhooks
    /(dashboard)         # Protected application
      /[orgId]           # Organization-scoped routes
    /(marketing)         # Public pages
    
/lib
  /supabase
    /client.ts           # Browser Supabase client
    /server.ts           # Server-side client
    /admin.ts            # Service role client
    /types.ts            # Generated TypeScript types
    /migrations          # SQL migrations
  /clerk
    /middleware.ts       # Auth + RBAC middleware
    /webhooks.ts         # Webhook handlers
    /rbac.ts             # Role-based access control
  /monitoring
    /sentry.ts           # Error tracking
    /analytics.ts        # Custom event tracking
  /email
    /templates           # Email templates
    /send.ts             # Email sending logic
  /cache
    /redis.ts            # Redis client and helpers
  /utils
    /validation.ts       # Zod schemas
    /errors.ts           # Custom error classes
    
/tests
  /unit                  # Vitest unit tests
  /integration           # API integration tests
  /e2e                   # Playwright end-to-end tests
  /fixtures              # Test data
  
/scripts
  /backup.ts             # Database backup to S3
  /seed.ts               # Database seeding
  /migrate.ts            # Migration runner
  
/.github
  /workflows
    main.yml             # CI/CD pipeline
    backup.yml           # Daily backup job
    security.yml         # Dependency scanning
```

---

## Database Architecture

### Supabase PostgreSQL Schema

```sql
-- Enable necessary extensions
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

-- Enable Row Level Security globally
ALTER DEFAULT PRIVILEGES REVOKE EXECUTE ON FUNCTIONS FROM PUBLIC;

-- Organizations (B2B multi-tenancy)
CREATE TABLE organizations (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  clerk_org_id TEXT UNIQUE NOT NULL,
  name TEXT NOT NULL,
  slug TEXT UNIQUE NOT NULL,
  subscription_tier TEXT DEFAULT 'free' CHECK (subscription_tier IN ('free', 'starter', 'pro', 'enterprise')),
  subscription_status TEXT DEFAULT 'active' CHECK (subscription_status IN ('active', 'cancelled', 'past_due')),
  max_seats INTEGER DEFAULT 5,
  settings JSONB DEFAULT '{}'::JSONB,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_organizations_slug ON organizations(slug);
CREATE INDEX idx_organizations_clerk_id ON organizations(clerk_org_id);

-- Users (synced from Clerk via webhook)
CREATE TABLE profiles (
  id UUID PRIMARY KEY REFERENCES auth.users ON DELETE CASCADE,
  clerk_user_id TEXT UNIQUE NOT NULL,
  email TEXT NOT NULL,
  full_name TEXT,
  avatar_url TEXT,
  current_organization_id UUID REFERENCES organizations ON DELETE SET NULL,
  preferences JSONB DEFAULT '{}'::JSONB,
  last_seen_at TIMESTAMPTZ DEFAULT NOW(),
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_profiles_email ON profiles(email);
CREATE INDEX idx_profiles_clerk_id ON profiles(clerk_user_id);
CREATE INDEX idx_profiles_org_id ON profiles(current_organization_id);

-- Organization Memberships
CREATE TABLE organization_memberships (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  organization_id UUID NOT NULL REFERENCES organizations ON DELETE CASCADE,
  user_id UUID NOT NULL REFERENCES profiles ON DELETE CASCADE,
  role TEXT NOT NULL CHECK (role IN ('owner', 'admin', 'member', 'viewer')),
  joined_at TIMESTAMPTZ DEFAULT NOW(),
  UNIQUE(organization_id, user_id)
);

CREATE INDEX idx_memberships_org ON organization_memberships(organization_id);
CREATE INDEX idx_memberships_user ON organization_memberships(user_id);

-- Audit Log (compliance and debugging)
CREATE TABLE audit_logs (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  organization_id UUID REFERENCES organizations ON DELETE CASCADE,
  user_id UUID REFERENCES profiles ON DELETE SET NULL,
  action TEXT NOT NULL,
  resource_type TEXT NOT NULL,
  resource_id TEXT,
  metadata JSONB DEFAULT '{}'::JSONB,
  ip_address INET,
  user_agent TEXT,
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_audit_logs_org ON audit_logs(organization_id, created_at DESC);
CREATE INDEX idx_audit_logs_user ON audit_logs(user_id, created_at DESC);
CREATE INDEX idx_audit_logs_action ON audit_logs(action);

-- Example Business Entity (customize to your needs)
CREATE TABLE projects (
  id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
  organization_id UUID NOT NULL REFERENCES organizations ON DELETE CASCADE,
  created_by UUID NOT NULL REFERENCES profiles ON DELETE RESTRICT,
  title TEXT NOT NULL,
  description TEXT,
  status TEXT DEFAULT 'active' CHECK (status IN ('active', 'archived', 'deleted')),
  metadata JSONB DEFAULT '{}'::JSONB,
  created_at TIMESTAMPTZ DEFAULT NOW(),
  updated_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_projects_org ON projects(organization_id, created_at DESC);
CREATE INDEX idx_projects_creator ON projects(created_by);
CREATE INDEX idx_projects_status ON projects(status);

-- Row Level Security Policies

-- Organizations: Users can only see orgs they're members of
ALTER TABLE organizations ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Users can view their organizations"
  ON organizations FOR SELECT
  USING (
    EXISTS (
      SELECT 1 FROM organization_memberships
      WHERE organization_id = organizations.id
      AND user_id = auth.uid()
    )
  );

CREATE POLICY "Owners and admins can update their organizations"
  ON organizations FOR UPDATE
  USING (
    EXISTS (
      SELECT 1 FROM organization_memberships
      WHERE organization_id = organizations.id
      AND user_id = auth.uid()
      AND role IN ('owner', 'admin')
    )
  );

-- Profiles: Users can view any profile but only update their own
ALTER TABLE profiles ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Profiles are viewable by authenticated users"
  ON profiles FOR SELECT
  TO authenticated
  USING (true);

CREATE POLICY "Users can update own profile"
  ON profiles FOR UPDATE
  USING (auth.uid() = id);

-- Organization Memberships: Viewable by org members
ALTER TABLE organization_memberships ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Memberships viewable by org members"
  ON organization_memberships FOR SELECT
  USING (
    EXISTS (
      SELECT 1 FROM organization_memberships om
      WHERE om.organization_id = organization_memberships.organization_id
      AND om.user_id = auth.uid()
    )
  );

-- Projects: Scoped to organization
ALTER TABLE projects ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Projects viewable by org members"
  ON projects FOR SELECT
  USING (
    EXISTS (
      SELECT 1 FROM organization_memberships
      WHERE organization_id = projects.organization_id
      AND user_id = auth.uid()
    )
  );

CREATE POLICY "Projects manageable by members"
  ON projects FOR ALL
  USING (
    EXISTS (
      SELECT 1 FROM organization_memberships
      WHERE organization_id = projects.organization_id
      AND user_id = auth.uid()
      AND role IN ('owner', 'admin', 'member')
    )
  );

-- Audit logs: Viewable by org admins and owners
ALTER TABLE audit_logs ENABLE ROW LEVEL SECURITY;

CREATE POLICY "Audit logs viewable by org admins"
  ON audit_logs FOR SELECT
  USING (
    EXISTS (
      SELECT 1 FROM organization_memberships
      WHERE organization_id = audit_logs.organization_id
      AND user_id = auth.uid()
      AND role IN ('owner', 'admin')
    )
  );

-- Functions and Triggers

-- Update updated_at timestamp automatically
CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
  NEW.updated_at = NOW();
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER update_organizations_updated_at
  BEFORE UPDATE ON organizations
  FOR EACH ROW
  EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_profiles_updated_at
  BEFORE UPDATE ON profiles
  FOR EACH ROW
  EXECUTE FUNCTION update_updated_at_column();

CREATE TRIGGER update_projects_updated_at
  BEFORE UPDATE ON projects
  FOR EACH ROW
  EXECUTE FUNCTION update_updated_at_column();

-- Automatic audit logging function
CREATE OR REPLACE FUNCTION log_audit_event(
  p_organization_id UUID,
  p_user_id UUID,
  p_action TEXT,
  p_resource_type TEXT,
  p_resource_id TEXT,
  p_metadata JSONB DEFAULT '{}'::JSONB
)
RETURNS UUID AS $$
DECLARE
  v_log_id UUID;
BEGIN
  INSERT INTO audit_logs (
    organization_id,
    user_id,
    action,
    resource_type,
    resource_id,
    metadata
  ) VALUES (
    p_organization_id,
    p_user_id,
    p_action,
    p_resource_type,
    p_resource_id,
    p_metadata
  ) RETURNING id INTO v_log_id;
  
  RETURN v_log_id;
END;
$$ LANGUAGE plpgsql SECURITY DEFINER;
```

### Type Generation

```bash
# Run after schema changes to generate TypeScript types
npx supabase gen types typescript --project-id your-project-ref > lib/supabase/types.ts
```

---

## Authentication and Authorization

### Clerk Setup with Organizations

```typescript
// middleware.ts
import { clerkMiddleware, createRouteMatcher } from '@clerk/nextjs/server';
import { NextResponse } from 'next/server';

const isPublicRoute = createRouteMatcher([
  '/',
  '/sign-in(.*)',
  '/sign-up(.*)',
  '/api/webhooks(.*)',
]);

const isApiRoute = createRouteMatcher(['/api(.*)']);

export default clerkMiddleware((auth, req) => {
  // Public routes don't need auth
  if (isPublicRoute(req)) {
    return NextResponse.next();
  }

  // Protect all other routes
  const { userId, orgId } = auth();
  
  if (!userId) {
    return auth().redirectToSignIn();
  }

  // API routes need valid auth but don't redirect
  if (isApiRoute(req) && !userId) {
    return NextResponse.json({ error: 'Unauthorized' }, { status: 401 });
  }

  // Organization-scoped routes require org context
  if (req.nextUrl.pathname.includes('/dashboard') && !orgId) {
    return NextResponse.redirect(new URL('/select-organization', req.url));
  }

  return NextResponse.next();
});

export const config = {
  matcher: ['/((?!.*\\..*|_next).*)', '/', '/(api|trpc)(.*)'],
};
```

### RBAC Implementation

```typescript
// lib/clerk/rbac.ts
import { auth } from '@clerk/nextjs/server';
import { createClient } from '@/lib/supabase/server';

export type Role = 'owner' | 'admin' | 'member' | 'viewer';

export type Permission = 
  | 'projects:create'
  | 'projects:read'
  | 'projects:update'
  | 'projects:delete'
  | 'members:invite'
  | 'members:remove'
  | 'billing:manage'
  | 'organization:update'
  | 'organization:delete';

const rolePermissions: Record<Role, Permission[]> = {
  owner: [
    'projects:create',
    'projects:read',
    'projects:update',
    'projects:delete',
    'members:invite',
    'members:remove',
    'billing:manage',
    'organization:update',
    'organization:delete',
  ],
  admin: [
    'projects:create',
    'projects:read',
    'projects:update',
    'projects:delete',
    'members:invite',
    'members:remove',
    'organization:update',
  ],
  member: [
    'projects:create',
    'projects:read',
    'projects:update',
  ],
  viewer: [
    'projects:read',
  ],
};

export async function hasPermission(permission: Permission): Promise<boolean> {
  const { userId, orgId } = await auth();
  
  if (!userId || !orgId) return false;

  const supabase = createClient();
  
  const { data: membership } = await supabase
    .from('organization_memberships')
    .select('role')
    .eq('user_id', userId)
    .eq('organization_id', orgId)
    .single();

  if (!membership) return false;

  const role = membership.role as Role;
  return rolePermissions[role].includes(permission);
}

export async function requirePermission(permission: Permission) {
  const allowed = await hasPermission(permission);
  
  if (!allowed) {
    throw new Error(`Permission denied: ${permission}`);
  }
}

export async function getUserRole(orgId: string): Promise<Role | null> {
  const { userId } = await auth();
  if (!userId) return null;

  const supabase = createClient();
  
  const { data } = await supabase
    .from('organization_memberships')
    .select('role')
    .eq('user_id', userId)
    .eq('organization_id', orgId)
    .single();

  return data?.role as Role || null;
}
```

### Clerk Webhook Handler (User Sync)

```typescript
// app/api/webhooks/clerk/route.ts
import { headers } from 'next/headers';
import { Webhook } from 'svix';
import { WebhookEvent } from '@clerk/nextjs/server';
import { createClient } from '@/lib/supabase/admin';

export async function POST(req: Request) {
  const WEBHOOK_SECRET = process.env.CLERK_WEBHOOK_SECRET;

  if (!WEBHOOK_SECRET) {
    throw new Error('Missing CLERK_WEBHOOK_SECRET');
  }

  const headerPayload = headers();
  const svix_id = headerPayload.get('svix-id');
  const svix_timestamp = headerPayload.get('svix-timestamp');
  const svix_signature = headerPayload.get('svix-signature');

  if (!svix_id || !svix_timestamp || !svix_signature) {
    return new Response('Missing svix headers', { status: 400 });
  }

  const payload = await req.json();
  const body = JSON.stringify(payload);

  const wh = new Webhook(WEBHOOK_SECRET);

  let evt: WebhookEvent;

  try {
    evt = wh.verify(body, {
      'svix-id': svix_id,
      'svix-timestamp': svix_timestamp,
      'svix-signature': svix_signature,
    }) as WebhookEvent;
  } catch (err) {
    console.error('Webhook verification failed:', err);
    return new Response('Invalid signature', { status: 400 });
  }

  const supabase = createClient();

  // Handle different webhook events
  switch (evt.type) {
    case 'user.created':
      await supabase.from('profiles').insert({
        clerk_user_id: evt.data.id,
        email: evt.data.email_addresses[0].email_address,
        full_name: `${evt.data.first_name} ${evt.data.last_name}`,
        avatar_url: evt.data.image_url,
      });
      break;

    case 'user.updated':
      await supabase
        .from('profiles')
        .update({
          email: evt.data.email_addresses[0].email_address,
          full_name: `${evt.data.first_name} ${evt.data.last_name}`,
          avatar_url: evt.data.image_url,
        })
        .eq('clerk_user_id', evt.data.id);
      break;

    case 'user.deleted':
      await supabase
        .from('profiles')
        .delete()
        .eq('clerk_user_id', evt.data.id!);
      break;

    case 'organization.created':
      await supabase.from('organizations').insert({
        clerk_org_id: evt.data.id,
        name: evt.data.name,
        slug: evt.data.slug,
      });
      break;

    case 'organizationMembership.created':
      const { data: profile } = await supabase
        .from('profiles')
        .select('id')
        .eq('clerk_user_id', evt.data.public_user_data.user_id)
        .single();

      const { data: org } = await supabase
        .from('organizations')
        .select('id')
        .eq('clerk_org_id', evt.data.organization.id)
        .single();

      if (profile && org) {
        await supabase.from('organization_memberships').insert({
          organization_id: org.id,
          user_id: profile.id,
          role: evt.data.role as any,
        });
      }
      break;
  }

  return new Response('Webhook processed', { status: 200 });
}
```

---

## Caching Strategy with Redis

```typescript
// lib/cache/redis.ts
import { Redis } from '@upstash/redis';

export const redis = new Redis({
  url: process.env.UPSTASH_REDIS_REST_URL!,
  token: process.env.UPSTASH_REDIS_REST_TOKEN!,
});

// Cache patterns
export const CacheKeys = {
  user: (userId: string) => `user:${userId}`,
  organization: (orgId: string) => `org:${orgId}`,
  membership: (userId: string, orgId: string) => `membership:${userId}:${orgId}`,
  projects: (orgId: string) => `projects:${orgId}`,
} as const;

// Cache with automatic invalidation
export async function cacheGet<T>(key: string): Promise<T | null> {
  const cached = await redis.get(key);
  return cached as T | null;
}

export async function cacheSet<T>(
  key: string,
  value: T,
  ttl: number = 300 // 5 minutes default
): Promise<void> {
  await redis.set(key, JSON.stringify(value), { ex: ttl });
}

export async function cacheDelete(key: string): Promise<void> {
  await redis.del(key);
}

export async function cacheDeletePattern(pattern: string): Promise<void> {
  const keys = await redis.keys(pattern);
  if (keys.length > 0) {
    await redis.del(...keys);
  }
}

// Example: Cached database query
export async function getOrganizationCached(orgId: string) {
  const cacheKey = CacheKeys.organization(orgId);
  
  // Try cache first
  const cached = await cacheGet(cacheKey);
  if (cached) return cached;
  
  // Fetch from database
  const supabase = createClient();
  const { data: org } = await supabase
    .from('organizations')
    .select('*')
    .eq('id', orgId)
    .single();
  
  // Cache for 10 minutes
  if (org) {
    await cacheSet(cacheKey, org, 600);
  }
  
  return org;
}
```

---

## Monitoring and Error Tracking

### Sentry Configuration

```typescript
// sentry.client.config.ts
import * as Sentry from '@sentry/nextjs';

Sentry.init({
  dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,
  
  // Performance monitoring
  tracesSampleRate: 1.0,
  
  // Session replay
  replaysSessionSampleRate: 0.1,
  replaysOnErrorSampleRate: 1.0,
  
  // Environment
  environment: process.env.NODE_ENV,
  
  // Add user context
  beforeSend(event, hint) {
    // Strip sensitive data
    if (event.request) {
      delete event.request.cookies;
      delete event.request.headers;
    }
    return event;
  },
  
  integrations: [
    new Sentry.BrowserTracing(),
    new Sentry.Replay({
      maskAllText: true,
      blockAllMedia: true,
    }),
  ],
});
```

```typescript
// sentry.server.config.ts
import * as Sentry from '@sentry/nextjs';

Sentry.init({
  dsn: process.env.NEXT_PUBLIC_SENTRY_DSN,
  tracesSampleRate: 1.0,
  environment: process.env.NODE_ENV,
  
  // Add custom context
  beforeSend(event, hint) {
    // Add organization context if available
    const { orgId } = auth();
    if (orgId) {
      event.contexts = {
        ...event.contexts,
        organization: { id: orgId },
      };
    }
    return event;
  },
});
```

### Audit Logging

```typescript
// lib/monitoring/audit.ts
import { createClient } from '@/lib/supabase/admin';
import { auth } from '@clerk/nextjs/server';
import { headers } from 'next/headers';

export async function logAuditEvent({
  organizationId,
  action,
  resourceType,
  resourceId,
  metadata = {},
}: {
  organizationId: string;
  action: string;
  resourceType: string;
  resourceId?: string;
  metadata?: Record<string, any>;
}) {
  const { userId } = await auth();
  const headersList = headers();
  
  const supabase = createClient();
  
  await supabase.rpc('log_audit_event', {
    p_organization_id: organizationId,
    p_user_id: userId,
    p_action: action,
    p_resource_type: resourceType,
    p_resource_id: resourceId,
    p_metadata: metadata,
  });
}

// Usage in API routes or server actions
export async function deleteProject(projectId: string) {
  const supabase = createClient();
  const { data: project } = await supabase
    .from('projects')
    .select('organization_id, title')
    .eq('id', projectId)
    .single();

  await supabase
    .from('projects')
    .delete()
    .eq('id', projectId);

  // Log the deletion
  await logAuditEvent({
    organizationId: project.organization_id,
    action: 'project.deleted',
    resourceType: 'project',
    resourceId: projectId,
    metadata: { title: project.title },
  });
}
```

---

## Data Flow Architecture

### Request Flow (Production)

1. User request hits Vercel Edge Network
2. Next.js middleware validates Clerk session
3. RBAC check via Redis cache (or Supabase if cache miss)
4. Server component/action queries Supabase with RLS
5. Response cached in Redis if appropriate
6. Audit log written asynchronously
7. Sentry monitors request duration and errors

### Real-time Subscriptions

```typescript
// components/ProjectList.tsx
'use client';

import { useEffect, useState } from 'react';
import { createClient } from '@/lib/supabase/client';

export function ProjectList({ organizationId }: { organizationId: string }) {
  const [projects, setProjects] = useState([]);
  const supabase = createClient();

  useEffect(() => {
    // Initial fetch
    const fetchProjects = async () => {
      const { data } = await supabase
        .from('projects')
        .select('*')
        .eq('organization_id', organizationId)
        .order('created_at', { ascending: false });
      
      setProjects(data || []);
    };

    fetchProjects();

    // Subscribe to real-time changes
    const channel = supabase
      .channel('projects')
      .on(
        'postgres_changes',
        {
          event: '*',
          schema: 'public',
          table: 'projects',
          filter: `organization_id=eq.${organizationId}`,
        },
        (payload) => {
          if (payload.eventType === 'INSERT') {
            setProjects((prev) => [payload.new, ...prev]);
          } else if (payload.eventType === 'UPDATE') {
            setProjects((prev) =>
              prev.map((p) => (p.id === payload.new.id ? payload.new : p))
            );
          } else if (payload.eventType === 'DELETE') {
            setProjects((prev) => prev.filter((p) => p.id !== payload.old.id));
          }
        }
      )
      .subscribe();

    return () => {
      supabase.removeChannel(channel);
    };
  }, [organizationId]);

  return (
    <div>
      {projects.map((project) => (
        <div key={project.id}>{project.title}</div>
      ))}
    </div>
  );
}
```

---

## Backup and Disaster Recovery

### Automated Backup Strategy

**Supabase Built-in Backups**
- Daily automatic snapshots (retained 7 days on Pro tier)
- Point-in-time recovery (PITR) available
- Backups stored in geographically distributed locations

**Custom S3 Archive Backups**

```typescript
// scripts/backup.ts
import { createClient } from '@supabase/supabase-js';
import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3';
import { exec } from 'child_process';
import { promisify } from 'util';

const execAsync = promisify(exec);

const supabase = createClient(
  process.env.NEXT_PUBLIC_SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_ROLE_KEY!
);

const s3 = new S3Client({
  region: process.env.AWS_REGION!,
  credentials: {
    accessKeyId: process.env.AWS_ACCESS_KEY_ID!,
    secretAccessKey: process.env.AWS_SECRET_ACCESS_KEY!,
  },
});

async function backupDatabase() {
  const timestamp = new Date().toISOString().split('T')[0];
  const filename = `backup-${timestamp}.sql`;

  try {
    // Use pg_dump to create SQL backup
    const dbUrl = process.env.DATABASE_URL!;
    const { stdout } = await execAsync(
      `pg_dump "${dbUrl}" --clean --if-exists`
    );

    // Upload to S3
    await s3.send(
      new PutObjectCommand({
        Bucket: process.env.BACKUP_BUCKET!,
        Key: `database/${filename}`,
        Body: stdout,
        ContentType: 'application/sql',
      })
    );

    console.log(`Backup completed: ${filename}`);
    
    // Store backup metadata in database
    await supabase.from('backup_logs').insert({
      filename,
      size_bytes: Buffer.byteLength(stdout),
      backup_type: 'automated',
      status: 'completed',
    });

    return { success: true, filename };
  } catch (error) {
    console.error('Backup failed:', error);
    
    await supabase.from('backup_logs').insert({
      filename,
      backup_type: 'automated',
      status: 'failed',
      error_message: error.message,
    });

    throw error;
  }
}

// Run backup
backupDatabase();
```

### GitHub Actions Backup Job

```yaml
# .github/workflows/backup.yml
name: Daily Database Backup

on:
  schedule:
    - cron: '0 2 * * *'  # Run at 2 AM UTC daily
  workflow_dispatch:     # Allow manual trigger

jobs:
  backup:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '20'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run backup script
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL }}
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          AWS_REGION: ${{ secrets.AWS_REGION }}
          BACKUP_BUCKET: ${{ secrets.BACKUP_BUCKET }}
          NEXT_PUBLIC_SUPABASE_URL: ${{ secrets.NEXT_PUBLIC_SUPABASE_URL }}
          SUPABASE_SERVICE_ROLE_KEY: ${{ secrets.SUPABASE_SERVICE_ROLE_KEY }}
        run: npm run backup
      
      - name: Notify on failure
        if: failure()
        uses: 8398a7/action-slack@v3
        with:
          status: ${{ job.status }}
          text: 'Database backup failed!'
          webhook_url: ${{ secrets.SLACK_WEBHOOK }}
```

### Recovery Procedures

**Standard Recovery (Recent Backup)**
1. Access Supabase dashboard
2. Navigate to Database > Backups
3. Select appropriate snapshot
4. Click "Restore" and confirm
5. Verify data integrity post-restore

**Point-in-Time Recovery**
1. Contact Supabase support with specific timestamp
2. Request PITR restoration
3. Test on staging environment first
4. Apply to production after verification

**S3 Archive Recovery**
1. Download backup from S3: `aws s3 cp s3://bucket/database/backup-YYYY-MM-DD.sql ./`
2. Restore to Supabase: `psql "DATABASE_URL" < backup-YYYY-MM-DD.sql`
3. Verify critical tables and relationships
4. Run data integrity checks
5. Clear Redis cache: `redis-cli FLUSHALL`
6. Notify team of restoration

**RTO/RPO Targets**
- Recovery Time Objective: 2 hours for standard recovery
- Recovery Point Objective: 24 hours (daily backup cadence)
- For critical data: Consider more frequent backups

---

## Security Model

### Authentication Security

- **Multi-Factor Authentication**: Enforced for organization owners/admins
- **Session Management**: 
  - JWT tokens with 1-hour expiry
  - Refresh tokens with 7-day expiry
  - Automatic renewal on activity
- **OAuth Providers**: Google, GitHub, Microsoft supported
- **Password Requirements**: Minimum 12 characters when applicable

### Authorization Security

- **Row Level Security**: All tables protected with RLS policies
- **RBAC**: Role-based permissions enforced at middleware and database levels
- **API Authentication**: All API routes require valid Clerk session
- **Service Role Key**: Used only in secure server contexts, never exposed to client

### Data Protection

- **Encryption at Rest**: All Supabase data encrypted with AES-256
- **Encryption in Transit**: TLS 1.3 enforced for all connections
- **Secrets Management**: Environment variables via Vercel, never committed to git
- **API Keys**: Rate-limited and IP-restricted where possible
- **File Uploads**: Virus scanning via ClamAV integration (if handling user uploads)

### GDPR Compliance

**Data Subject Rights**
- Right to access: API endpoint for data export
- Right to deletion: Cascading delete across all tables
- Right to rectification: User profile update endpoints
- Right to portability: JSON export of all user data

```typescript
// app/api/gdpr/export/route.ts
import { auth } from '@clerk/nextjs/server';
import { createClient } from '@/lib/supabase/server';

export async function GET() {
  const { userId } = await auth();
  if (!userId) {
    return Response.json({ error: 'Unauthorized' }, { status: 401 });
  }

  const supabase = createClient();

  // Fetch all user data
  const { data: profile } = await supabase
    .from('profiles')
    .select('*')
    .eq('id', userId)
    .single();

  const { data: projects } = await supabase
    .from('projects')
    .select('*')
    .eq('created_by', userId);

  const { data: auditLogs } = await supabase
    .from('audit_logs')
    .select('*')
    .eq('user_id', userId);

  const exportData = {
    profile,
    projects,
    auditLogs,
    exportedAt: new Date().toISOString(),
  };

  return Response.json(exportData, {
    headers: {
      'Content-Disposition': `attachment; filename="user-data-${userId}.json"`,
    },
  });
}
```

**Cookie Consent**
- Cookie consent banner on first visit
- Preferences stored in localStorage
- Only essential cookies without consent
- Analytics/tracking cookies opt-in only

**Privacy Policy Requirements**
- Data collection disclosure
- Third-party service disclosure (Clerk, Supabase, Sentry)
- Data retention policies
- Contact information for privacy inquiries

---

## Testing Strategy

### Unit Tests (Vitest)

```typescript
// tests/unit/rbac.test.ts
import { describe, it, expect, beforeEach, vi } from 'vitest';
import { hasPermission, rolePermissions } from '@/lib/clerk/rbac';

vi.mock('@clerk/nextjs/server', () => ({
  auth: vi.fn(() => ({ userId: 'user_123', orgId: 'org_456' })),
}));

vi.mock('@/lib/supabase/server', () => ({
  createClient: vi.fn(() => ({
    from: vi.fn(() => ({
      select: vi.fn(() => ({
        eq: vi.fn(() => ({
          single: vi.fn(() => ({
            data: { role: 'admin' },
          })),
        })),
      })),
    })),
  })),
}));

describe('RBAC Permissions', () => {
  it('should grant project:create to admin', async () => {
    const result = await hasPermission('projects:create');
    expect(result).toBe(true);
  });

  it('should deny organization:delete to admin', async () => {
    const result = await hasPermission('organization:delete');
    expect(result).toBe(false);
  });

  it('should have correct permissions hierarchy', () => {
    expect(rolePermissions.owner.length).toBeGreaterThan(
      rolePermissions.admin.length
    );
    expect(rolePermissions.admin.length).toBeGreaterThan(
      rolePermissions.member.length
    );
  });
});
```

### Integration Tests

```typescript
// tests/integration/api.test.ts
import { describe, it, expect, beforeAll, afterAll } from 'vitest';
import { createClient } from '@supabase/supabase-js';

const testSupabase = createClient(
  process.env.TEST_SUPABASE_URL!,
  process.env.TEST_SUPABASE_KEY!
);

describe('API Integration Tests', () => {
  let testOrgId: string;
  let testUserId: string;

  beforeAll(async () => {
    // Create test organization
    const { data: org } = await testSupabase
      .from('organizations')
      .insert({ name: 'Test Org', slug: 'test-org' })
      .select()
      .single();
    testOrgId = org.id;

    // Create test user
    const { data: user } = await testSupabase
      .from('profiles')
      .insert({ 
        clerk_user_id: 'test_user',
        email: 'test@example.com'
      })
      .select()
      .single();
    testUserId = user.id;
  });

  afterAll(async () => {
    // Cleanup test data
    await testSupabase.from('organizations').delete().eq('id', testOrgId);
    await testSupabase.from('profiles').delete().eq('id', testUserId);
  });

  it('should create project with valid auth', async () => {
    const response = await fetch('http://localhost:3000/api/v1/projects', {
      method: 'POST',
      headers: {
        'Content-Type': 'application/json',
        'Authorization': `Bearer ${testToken}`,
      },
      body: JSON.stringify({
        organizationId: testOrgId,
        title: 'Test Project',
      }),
    });

    expect(response.status).toBe(201);
    const data = await response.json();
    expect(data.project).toHaveProperty('id');
  });

  it('should reject unauthenticated requests', async () => {
    const response = await fetch('http://localhost:3000/api/v1/projects');
    expect(response.status).toBe(401);
  });
});
```

### E2E Tests (Playwright)

```typescript
// tests/e2e/auth.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Authentication Flow', () => {
  test('should sign up new user', async ({ page }) => {
    await page.goto('/sign-up');
    
    await page.fill('[name="email"]', 'test@example.com');
    await page.fill('[name="password"]', 'SecurePass123!');
    await page.click('button[type="submit"]');
    
    await expect(page).toHaveURL('/dashboard');
    await expect(page.locator('h1')).toContainText('Welcome');
  });

  test('should require organization selection', async ({ page }) => {
    await page.goto('/dashboard');
    
    // Should redirect to org selection if no org context
    await expect(page).toHaveURL('/select-organization');
  });

  test('should handle invalid credentials', async ({ page }) => {
    await page.goto('/sign-in');
    
    await page.fill('[name="email"]', 'invalid@example.com');
    await page.fill('[name="password"]', 'wrong');
    await page.click('button[type="submit"]');
    
    await expect(page.locator('[role="alert"]')).toContainText(
      'Invalid credentials'
    );
  });
});
```

### Test Coverage Requirements

- **Unit Tests**: 80% code coverage minimum
- **Integration Tests**: All API routes covered
- **E2E Tests**: All critical user journeys
- **Run Before Deploy**: All tests must pass in CI/CD

---

## CI/CD Pipeline

```yaml
# .github/workflows/main.yml
name: CI/CD Pipeline

on:
  push:
    branches: [main, staging, develop]
  pull_request:
    branches: [main, staging]

env:
  NODE_VERSION: '20'

jobs:
  lint-and-typecheck:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run ESLint
        run: npm run lint
      
      - name: TypeScript type check
        run: npm run type-check

  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run unit tests
        run: npm run test:unit -- --coverage
      
      - name: Upload coverage to Codecov
        uses: codecov/codecov-action@v3
        with:
          files: ./coverage/coverage-final.json

  integration-tests:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: test
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run database migrations
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/test
        run: npm run db:migrate
      
      - name: Run integration tests
        env:
          DATABASE_URL: postgresql://postgres:postgres@localhost:5432/test
          TEST_SUPABASE_URL: ${{ secrets.TEST_SUPABASE_URL }}
          TEST_SUPABASE_KEY: ${{ secrets.TEST_SUPABASE_KEY }}
        run: npm run test:integration

  e2e-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Install Playwright browsers
        run: npx playwright install --with-deps
      
      - name: Run E2E tests
        env:
          BASE_URL: http://localhost:3000
          TEST_USER_EMAIL: ${{ secrets.TEST_USER_EMAIL }}
          TEST_USER_PASSWORD: ${{ secrets.TEST_USER_PASSWORD }}
        run: |
          npm run build
          npm run start &
          npx wait-on http://localhost:3000
          npm run test:e2e
      
      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v3
        with:
          name: playwright-report
          path: playwright-report/

  security-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Run Snyk security scan
        uses: snyk/actions/node@master
        env:
          SNYK_TOKEN: ${{ secrets.SNYK_TOKEN }}
        with:
          args: --severity-threshold=high

  deploy-staging:
    if: github.ref == 'refs/heads/staging'
    needs: [lint-and-typecheck, unit-tests, integration-tests, e2e-tests]
    runs-on: ubuntu-latest
    environment:
      name: staging
      url: https://staging.yourdomain.com
    steps:
      - uses: actions/checkout@v3
      
      - name: Deploy to Vercel (Staging)
        uses: amondnet/vercel-action@v25
        with:
          vercel-token: ${{ secrets.VERCEL_TOKEN }}
          vercel-org-id: ${{ secrets.VERCEL_ORG_ID }}
          vercel-project-id: ${{ secrets.VERCEL_PROJECT_ID }}
          vercel-args: '--prod'
          scope: ${{ secrets.VERCEL_ORG_ID }}

  deploy-production:
    if: github.ref == 'refs/heads/main'
    needs: [lint-and-typecheck, unit-tests, integration-tests, e2e-tests, security-scan]
    runs-on: ubuntu-latest
    environment:
      name: production
      url: https://yourdomain.com
    steps:
      - uses: actions/checkout@v3
      
      - name: Deploy to Vercel (Production)
        uses: amondnet/vercel-action@v25
        with:
          vercel-token: ${{ secrets.VERCEL_TOKEN }}
          vercel-org-id: ${{ secrets.VERCEL_ORG_ID }}
          vercel-project-id: ${{ secrets.VERCEL_PROJECT_ID }}
          vercel-args: '--prod'
          scope: ${{ secrets.VERCEL_ORG_ID }}
      
      - name: Run post-deployment tests
        run: |
          npx wait-on https://yourdomain.com
          npm run test:smoke
      
      - name: Notify team
        if: always()
        uses: 8398a7/action-slack@v3
        with:
          status: ${{ job.status }}
          text: 'Production deployment ${{ job.status }}'
          webhook_url: ${{ secrets.SLACK_WEBHOOK }}
```

---

## Environment Variables (Production)

```bash
# .env.production

# Clerk Authentication
NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_live_xxx
CLERK_SECRET_KEY=sk_live_xxx
CLERK_WEBHOOK_SECRET=whsec_xxx

# Supabase
NEXT_PUBLIC_SUPABASE_URL=https://xxx.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJxxx
SUPABASE_SERVICE_ROLE_KEY=eyJxxx
DATABASE_URL=postgresql://postgres:[PASSWORD]@xxx.supabase.co:5432/postgres

# Redis Cache
UPSTASH_REDIS_REST_URL=https://xxx.upstash.io
UPSTASH_REDIS_REST_TOKEN=xxx

# Sentry Monitoring
NEXT_PUBLIC_SENTRY_DSN=https://xxx@xxx.ingest.sentry.io/xxx
SENTRY_AUTH_TOKEN=xxx
SENTRY_ORG=your-org
SENTRY_PROJECT=your-project

# Email (Resend)
RESEND_API_KEY=re_xxx
FROM_EMAIL=noreply@yourdomain.com

# AWS S3 (for backups)
AWS_ACCESS_KEY_ID=xxx
AWS_SECRET_ACCESS_KEY=xxx
AWS_REGION=eu-west-1
BACKUP_BUCKET=your-backup-bucket

# Application
NODE_ENV=production
NEXT_PUBLIC_APP_URL=https://yourdomain.com
```

---

## Cost Breakdown (Monthly)

### Base Tier (Production Ready)

- **Vercel Pro**: £16/month (1TB bandwidth, team features)
- **Clerk Pro**: £20/month (10k MAU, organizations, MFA)
- **Supabase Pro**: £20/month (8GB database, 100GB storage, daily backups)
- **Upstash Redis**: £5/month (100k commands/day)
- **Sentry Team**: £20/month (50k errors, 100k transactions, session replay)
- **Resend**: £0-15/month (3k emails free, then pay-as-go)
- **AWS S3**: £2/month (backup storage)

**Total Base**: £83-98/month

### Scaling Tier (10k+ MAU)

- **Vercel Pro**: £16/month
- **Clerk Production**: £99/month (50k MAU)
- **Supabase Team**: £99/month (100GB database, PITR)
- **Upstash Redis**: £20/month (1M commands/day)
- **Sentry Business**: £80/month (500k errors, 1M transactions)
- **Resend**: £50/month (estimated for 50k emails)
- **AWS S3**: £5/month

**Total Scaling**: £369/month

### Enterprise Tier (100k+ MAU)

Custom pricing, typically:
- Clerk: £500+/month
- Supabase: £500+/month
- Dedicated support contracts
- Custom SLAs

---

## Deployment Checklist

### Pre-Launch

- [ ] All environment variables configured in Vercel
- [ ] Database migrations run on production Supabase
- [ ] Clerk production instance configured with custom domain
- [ ] DNS records configured (A, CNAME, TXT for email)
- [ ] SSL certificates active
- [ ] Sentry error tracking verified
- [ ] Backup automation tested
- [ ] GDPR pages published (Privacy Policy, Terms of Service)
- [ ] Cookie consent banner functional
- [ ] Rate limiting configured
- [ ] Email templates tested
- [ ] Payment integration tested (if applicable)

### Post-Launch

- [ ] Monitor error rates in Sentry (first 24 hours)
- [ ] Verify backup completion logs
- [ ] Test critical user flows in production
- [ ] Monitor database performance
- [ ] Check Redis cache hit rates
- [ ] Review Clerk authentication logs
- [ ] Verify webhook deliveries
- [ ] Test audit logging
- [ ] Monitor API response times
- [ ] Set up uptime monitoring (e.g., UptimeRobot)

### Ongoing Maintenance

- [ ] Weekly: Review error reports
- [ ] Weekly: Check backup logs
- [ ] Monthly: Review cost usage
- [ ] Monthly: Security dependency updates
- [ ] Quarterly: Review RBAC permissions
- [ ] Quarterly: Audit log analysis
- [ ] Annually: Security audit
- [ ] Annually: GDPR compliance review

---

## Quick Reference for Cursor

### Creating New API Routes

```typescript
// Standard production API route pattern
import { auth } from '@clerk/nextjs/server';
import { createClient } from '@/lib/supabase/server';
import { requirePermission } from '@/lib/clerk/rbac';
import { logAuditEvent } from '@/lib/monitoring/audit';
import * as Sentry from '@sentry/nextjs';
import { z } from 'zod';

const schema = z.object({
  title: z.string().min(1).max(100),
  description: z.string().optional(),
});

export async function POST(request: Request) {
  const transaction = Sentry.startTransaction({
    name: 'POST /api/v1/resource',
    op: 'http.server',
  });

  try {
    const { userId, orgId } = await auth();
    if (!userId || !orgId) {
      return Response.json({ error: 'Unauthorized' }, { status: 401 });
    }

    await requirePermission('resource:create');

    const body = await request.json();
    const validated = schema.parse(body);

    const supabase = createClient();
    const { data, error } = await supabase
      .from('table_name')
      .insert({
        organization_id: orgId,
        created_by: userId,
        ...validated,
      })
      .select()
      .single();

    if (error) throw error;

    await logAuditEvent({
      organizationId: orgId,
      action: 'resource.created',
      resourceType: 'resource',
      resourceId: data.id,
    });

    return Response.json({ data }, { status: 201 });
  } catch (error) {
    Sentry.captureException(error);
    console.error('API Error:', error);
    
    if (error instanceof z.ZodError) {
      return Response.json(
        { error: 'Validation failed', details: error.errors },
        { status: 400 }
      );
    }

    return Response.json(
      { error: 'Internal server error' },
      { status: 500 }
    );
  } finally {
    transaction.finish();
  }
}
```

### Common Commands

```bash
# Development
npm run dev                    # Start dev server
npm run type-check             # TypeScript validation
npm run lint                   # ESLint

# Testing
npm run test:unit              # Unit tests
npm run test:integration       # Integration tests
npm run test:e2e              # E2E tests
npm run test:e2e:ui           # E2E with UI

# Database
npm run db:migrate            # Run migrations
npm run db:seed               # Seed database
npm run db:reset              # Reset database
npm run db:types              # Generate TypeScript types

# Deployment
vercel --prod                 # Deploy to production
npm run backup                # Manual backup trigger

# Monitoring
npm run sentry:sourcemaps     # Upload source maps
```

---

## Support and Troubleshooting

### Common Issues

**Database Connection Issues**
- Verify Supabase service status
- Check connection pooling limits
- Review RLS policies for permission issues

**Authentication Problems**
- Verify Clerk webhook delivery in dashboard
- Check middleware configuration
- Review auth token expiry settings

**Cache Inconsistency**
- Flush Redis cache: `redis-cli FLUSHALL`
- Check cache invalidation logic
- Verify TTL settings

**Performance Degradation**
- Review Sentry performance traces
- Check database query performance in Supabase
- Analyze Redis cache hit rates

### Escalation Contacts

- **Database Issues**: Supabase support (support@supabase.io)
- **Auth Issues**: Clerk support (support@clerk.dev)
- **Monitoring Issues**: Sentry support
- **Infrastructure**: Vercel support

---

## Documentation Links

- **Next.js**: https://nextjs.org/docs
- **Clerk**: https://clerk.com/docs
- **Supabase**: https://supabase.com/docs
- **Sentry**: https://docs.sentry.io
- **Upstash Redis**: https://docs.upstash.com/redis
- **Resend**: https://resend.com/docs
- **Playwright**: https://playwright.dev
- **Vitest**: https://vitest.dev
