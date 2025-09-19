# MicroMatic Blueprint — v0.1

## Idea Summary & Stack Confirmation
- **Idea:** MicroMatic automates generation of editable micro-SaaS blueprints (schema, API, auth, scaffold) from natural language prompts.
- **Primary Stack:** Next.js 14 (App Router) + TypeScript, pnpm workspace, Tailwind CSS, shadcn/ui.
- **Backend/Data:** Neon Postgres with Prisma ORM, Drizzle migrations for SQL exports.
- **Auth:** NextAuth.js with email magic link + OAuth (GitHub/Google).
- **Queue/Background:** Vercel Edge functions for orchestrating model calls; optional background worker via Vercel Cron.
- **Why:** Matches PRD requirement for explainable, editable outputs, Neon integration, and fast deploy via Vercel.

## Domain Model & SQL Schema (Neon/Postgres)
```sql
-- users and organizations
CREATE TABLE organizations (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name TEXT NOT NULL,
  slug TEXT UNIQUE NOT NULL,
  plan TEXT NOT NULL DEFAULT 'free',
  created_at TIMESTAMP WITH TIME ZONE DEFAULT now()
);

CREATE TABLE users (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  email CITEXT UNIQUE NOT NULL,
  full_name TEXT,
  avatar_url TEXT,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT now()
);

CREATE TABLE organization_members (
  organization_id UUID REFERENCES organizations(id) ON DELETE CASCADE,
  user_id UUID REFERENCES users(id) ON DELETE CASCADE,
  role TEXT NOT NULL CHECK (role IN ('owner','editor','viewer')),
  invited_at TIMESTAMP WITH TIME ZONE DEFAULT now(),
  joined_at TIMESTAMP WITH TIME ZONE,
  PRIMARY KEY (organization_id, user_id)
);

-- blueprint lifecycle
CREATE TABLE blueprints (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id UUID REFERENCES organizations(id) ON DELETE CASCADE,
  title TEXT NOT NULL,
  idea_summary TEXT NOT NULL,
  target_persona TEXT,
  status TEXT NOT NULL DEFAULT 'draft',
  created_at TIMESTAMP WITH TIME ZONE DEFAULT now(),
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT now()
);

CREATE TABLE blueprint_versions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  blueprint_id UUID REFERENCES blueprints(id) ON DELETE CASCADE,
  version INT NOT NULL,
  generated_by UUID REFERENCES users(id),
  model_used TEXT NOT NULL,
  rationale JSONB,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT now(),
  UNIQUE (blueprint_id, version)
);

CREATE TABLE blueprint_sections (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  blueprint_version_id UUID REFERENCES blueprint_versions(id) ON DELETE CASCADE,
  section_type TEXT NOT NULL,
  content JSONB NOT NULL,
  updated_at TIMESTAMP WITH TIME ZONE DEFAULT now()
);

CREATE TABLE generation_runs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  blueprint_id UUID REFERENCES blueprints(id) ON DELETE CASCADE,
  requested_by UUID REFERENCES users(id),
  status TEXT NOT NULL CHECK (status IN ('pending','running','succeeded','failed')),
  duration_ms INTEGER,
  cost_cents INTEGER,
  error TEXT,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT now()
);

CREATE TABLE exports (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  blueprint_version_id UUID REFERENCES blueprint_versions(id) ON DELETE CASCADE,
  export_type TEXT NOT NULL,
  storage_url TEXT NOT NULL,
  checksum TEXT,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT now()
);

-- billing & usage
CREATE TABLE subscriptions (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id UUID REFERENCES organizations(id) ON DELETE CASCADE,
  provider TEXT NOT NULL,
  status TEXT NOT NULL,
  current_period_end TIMESTAMP WITH TIME ZONE,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT now()
);

CREATE TABLE usage_counters (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  organization_id UUID REFERENCES organizations(id) ON DELETE CASCADE,
  metric TEXT NOT NULL,
  value INTEGER NOT NULL DEFAULT 0,
  period_start TIMESTAMP WITH TIME ZONE DEFAULT date_trunc('month', now()),
  period_end TIMESTAMP WITH TIME ZONE
);

CREATE TABLE feedback (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  blueprint_version_id UUID REFERENCES blueprint_versions(id) ON DELETE CASCADE,
  submitted_by UUID REFERENCES users(id),
  rating SMALLINT CHECK (rating BETWEEN 1 AND 5),
  comment TEXT,
  created_at TIMESTAMP WITH TIME ZONE DEFAULT now()
);

CREATE INDEX idx_generation_runs_blueprint ON generation_runs (blueprint_id);
CREATE INDEX idx_blueprint_versions_blueprint ON blueprint_versions (blueprint_id);
CREATE INDEX idx_usage_counters_org_metric ON usage_counters (organization_id, metric);
```

### Schema Notes
- Separate `blueprint_versions` + `blueprint_sections` preserves editable chunks and diffing.
- `generation_runs` retains audit data (latency, cost) for observability KPIs.
- Usage counters keep monthly quotas; consider materialized view for dashboards.

## API Routes & Services (Next.js App Router)
| Method | Route | Description | Auth Scope |
| --- | --- | --- | --- |
| POST | `/api/blueprints` | Create blueprint from idea prompt; enqueue generation run. | `editor` |
| GET | `/api/blueprints` | List blueprints for org with filters. | `viewer` |
| GET | `/api/blueprints/[id]` | Retrieve blueprint metadata + latest version. | `viewer` |
| POST | `/api/blueprints/[id]/generate` | Regenerate sections with options (schema/api/ui). | `editor` |
| PATCH | `/api/blueprints/[id]` | Update metadata (title, persona). | `editor` |
| GET | `/api/blueprints/[id]/versions` | List versions with diffs. | `viewer` |
| GET | `/api/blueprints/[id]/versions/[versionId]` | Fetch specific version and sections. | `viewer` |
| POST | `/api/blueprints/[id]/sections` | Save manual edits to section JSON. | `editor` |
| POST | `/api/blueprints/[id]/exports` | Trigger ZIP export (code + SQL). | `editor` |
| GET | `/api/usage` | Return usage counters for dashboard. | `owner` |
| POST | `/api/feedback` | Record thumbs up/down with context. | `viewer` |
| POST | `/api/billing/checkout` | Create Stripe checkout session. | `owner` |
| POST | `/api/webhooks/stripe` | Stripe webhook handler (no auth; secret header). | Public + signature |

### Service Considerations
- Use tRPC or REST typed with Zod; align with Prisma models.
- Async generation handled via background worker (Edge function hitting job queue/Neon stored procedure).
- Webhooks validated with Stripe signing secret.

## Auth & RBAC
- **Roles:** `owner` (billing, invite), `editor` (generate/edit blueprints), `viewer` (read-only).
- **Session Flow:** NextAuth email magic link or OAuth; organization selection stored in session token.
- **Access Enforcement:** Middleware checks `organization_members.role` before hitting handlers; fallback to 403.
- **Row Security:** Optionally enable Postgres RLS when connecting through API layer for direct read (analytics exports).
- **Audit:** Log critical events (plan changes, exports) to `audit_logs` table (future addition).

## UI Scaffold Outline
1. **Marketing Landing (Public)** – Hero promise, CTA, templates carousel, pricing tiers.
2. **Onboarding Wizard (Auth)** – Guided questions, persona pickers, sample prompts.
3. **Generation Workspace**
   - Left: editable sections (Schema, API, Auth, UI, Deployment).
   - Right: rationale panel, run status timeline, cost/time metrics.
   - Toolbar: regenerate dropdown, diff viewer toggle, export button.
4. **Version & Feedback History** – Timeline view comparing versions, inline feedback chips.
5. **Usage & Billing Dashboard** – Quotas, upgrade CTA, billing history.
6. **Settings** – Team management, API keys, integration toggles.

## Deployment Checklist
1. Configure Neon project + database; set `DATABASE_URL` with pooled connection.
2. Provision Vercel project, link repo, set env vars (`DATABASE_URL`, `NEXTAUTH_SECRET`, provider keys, Stripe secrets).
3. Run `pnpm install`, `pnpm db:migrate` (Drizzle), `pnpm dev` for local smoke.
4. Set up background worker (Vercel cron hitting `/api/jobs/run`).
5. Add Stripe webhook endpoint (`/api/webhooks/stripe`) with signing secret.
6. Configure logging/monitoring (Logtail, Sentry) and analytics (PostHog).
7. Enable CI (GitHub Actions) with `pnpm lint`, `pnpm test`, `pnpm typecheck`.
8. Document manual QA: generate sample blueprint, edit sections, export ZIP, run Neon health check.

## Summary, Open Questions & Risks
- **Summary:** Stack leverages Next.js + Neon to meet fast iteration and deploy goals; schema supports versioned blueprints, billing, usage analytics.
- **Open Questions:**
  1. Should section content stay JSON or support Markdown/MDX for richer editing?
  2. Do we need multi-tenant isolation beyond row security (separate Neon branches)?
  3. How to price heavy blueprint generations—flat fee vs usage-based add-ons?
- **Risks:**
  - Model hallucinations causing schema/API mismatch (mitigate with validators + diff tests).
  - Stripe/webhook complexity for small team (need thorough sandbox testing).
  - Cost spikes from long model runs; require caching + rate limits early.
