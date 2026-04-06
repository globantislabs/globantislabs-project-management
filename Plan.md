# 🚀 Globantis Labs Project Management & Team Collaboration Dashboard
## Production-Grade "Hard Mode" Implementation Plan

**Status:** Ready for Production  
**Philosophy:** Maximum control, enterprise security, scalable architecture, zero technical debt.  
**Tech Stack:** Next.js 14 (App Router) + TypeScript + Tailwind CSS + shadcn/ui + Supabase + Vercel  
**Estimated Effort:** High (Full Custom Control)  
**Time to MVP:** 2-3 Days  
**Total Code Estimate:** ~1,500 - 2,000 lines (Production Quality)

---

## 🏗️ Architecture Overview

### Core Principles
1.  **Type Safety First:** Strict TypeScript everywhere (DB types generated automatically).
2.  **Security by Default:** Row Level Security (RLS) enforced on all tables.
3.  **Scalability:** Server Components for data fetching, Client Components for interactivity.
4.  **White-Label Ready:** Dynamic theming engine based on organization settings.
5.  **Observability:** Structured logging and error boundaries.

### Directory Structure (Production Standard)
```text
/workspace
├── supabase/
│   ├── migrations/          # SQL migration files
│   ├── config.toml          # Supabase CLI config
│   └── seed.sql             # Initial data
├── src/
│   ├── app/                 # Next.js App Router
│   │   ├── (auth)/          # Login/Signup routes
│   │   ├── (dashboard)/     # Protected dashboard routes
│   │   ├── api/             # Webhooks & Edge functions
│   │   └── layout.tsx       # Root layout with providers
│   ├── components/
│   │   ├── ui/              # shadcn/ui primitives
│   │   ├── dashboard/       # Feature-specific components
│   │   ├── forms/           # Reusable form components
│   │   └── layout/          # Sidebar, Header, Nav
│   ├── lib/
│   │   ├── supabase/        # Supabase clients (server/client)
│   │   ├── utils.ts         # CN helper, formatters
│   │   └── constants.ts     # App-wide constants
│   ├── hooks/               # Custom React hooks
│   ├── stores/              # Zustand state management
│   ├── types/               # Global TypeScript types
│   └── middleware.ts        # Auth protection middleware
├── .env.local               # Environment variables
├── next.config.js
├── tailwind.config.ts
└── package.json
```

---

## 📋 Phase 1: Foundation & Infrastructure (Day 1)

### 1.1 Initialization
Run these commands to set up the project skeleton:

```bash
npx create-next-app@latest globantis-dashboard --typescript --tailwind --eslint --app --src-dir --import-alias "@/*"
cd globantis-dashboard

# Install Core Dependencies
npm install @supabase/supabase-js @supabase/ssr zustand lucide-react date-fns clsx tailwind-merge
npm install -D @supabase/supabase-js @types/node

# Initialize shadcn/ui (Choose "New York" style, Slate color)
npx shadcn-ui@latest init
npx shadcn-ui@latest add button card input label avatar dropdown-menu dialog table toast skeleton badge tabs form select

# Install Supabase CLI for local dev
npm install -g supabase
```

### 1.2 Environment Configuration
Create `.env.local`:
```env
NEXT_PUBLIC_SUPABASE_URL=your_supabase_project_url
NEXT_PUBLIC_SUPABASE_ANON_KEY=your_supabase_anon_key
SUPABASE_SERVICE_ROLE_KEY=your_service_role_key # Server side only
NEXT_PUBLIC_APP_URL=http://localhost:3000
```

### 1.3 Database Schema (Production Grade SQL)
*File: `supabase/migrations/001_init_schema.sql`*

This schema includes RLS, triggers for updated_at, and proper indexing.

```sql
-- Enable UUID extension
create extension if not exists "uuid-ossp";

-- 1. Profiles Table (Extends Auth)
create table public.profiles (
  id uuid references auth.users on delete cascade primary key,
  email text unique not null,
  full_name text,
  avatar_url text,
  role text default 'member' check (role in ('admin', 'manager', 'member')),
  created_at timestamptz default now(),
  updated_at timestamptz default now()
);

-- 2. Organizations (For Multi-tenant/White-label support)
create table public.organizations (
  id uuid default uuid_generate_v4() primary key,
  name text not null,
  slug text unique not null,
  logo_url text,
  brand_color text default '#0F172A', -- Globantis Dark
  created_at timestamptz default now()
);

-- 3. Organization Members
create table public.organization_members (
  organization_id uuid references public.organizations on delete cascade,
  user_id uuid references public.profiles on delete cascade,
  role text default 'member' check (role in ('owner', 'admin', 'member')),
  joined_at timestamptz default now(),
  primary key (organization_id, user_id)
);

-- 4. Projects
create table public.projects (
  id uuid default uuid_generate_v4() primary key,
  organization_id uuid references public.organizations on delete cascade not null,
  name text not null,
  description text,
  status text default 'active' check (status in ('active', 'archived', 'on_hold')),
  created_by uuid references public.profiles,
  created_at timestamptz default now(),
  updated_at timestamptz default now()
);

-- 5. Tasks
create table public.tasks (
  id uuid default uuid_generate_v4() primary key,
  project_id uuid references public.projects on delete cascade not null,
  title text not null,
  description text,
  status text default 'todo' check (status in ('todo', 'in_progress', 'review', 'done')),
  priority text default 'medium' check (priority in ('low', 'medium', 'high', 'urgent')),
  assignee_id uuid references public.profiles,
  due_date timestamptz,
  created_at timestamptz default now(),
  updated_at timestamptz default now()
);

-- Indexes for performance
create index idx_tasks_project on public.tasks(project_id);
create index idx_tasks_assignee on public.tasks(assignee_id);
create index idx_org_members_user on public.organization_members(user_id);

-- Triggers for updated_at
create or replace function update_updated_at_column()
returns trigger as $$
begin
  new.updated_at = now();
  return new;
end;
$$ language plpgsql;

create trigger update_profiles_updated_at before update on public.profiles
  for each row execute procedure update_updated_at_column();
create trigger update_projects_updated_at before update on public.projects
  for each row execute procedure update_updated_at_column();
create trigger update_tasks_updated_at before update on public.tasks
  for each row execute procedure update_updated_at_column();

-- ROW LEVEL SECURITY (RLS) - CRITICAL FOR PRODUCTION
alter table public.profiles enable row level security;
alter table public.organizations enable row level security;
alter table public.organization_members enable row level security;
alter table public.projects enable row level security;
alter table public.tasks enable row level security;

-- Policies (Simplified for brevity, expand in production)
-- Profiles: Users can see their own
create policy "Users can view own profile" on public.profiles
  for select using (auth.uid() = id);
create policy "Users can update own profile" on public.profiles
  for update using (auth.uid() = id);

-- Orgs: Members can view their orgs
create policy "Members view orgs" on public.organizations
  for select using (
    exists (select 1 from public.organization_members where organization_id = id and user_id = auth.uid())
  );

-- Projects: Members of org can view projects
create policy "Org members view projects" on public.projects
  for select using (
    exists (
      select 1 from public.organization_members om
      where om.organization_id = project.organization_id
      and om.user_id = auth.uid()
    )
  );

-- Tasks: Same as projects
create policy "Org members view tasks" on public.tasks
  for select using (
    exists (
      select 1 from public.organization_members om
      join public.projects p on p.id = task.project_id
      where om.organization_id = p.organization_id
      and om.user_id = auth.uid()
    )
  );
```

### 1.4 Supabase Client Setup (SSR Ready)
*File: `src/lib/supabase/server.ts`*
```typescript
import { createServerClient, type CookieOptions } from '@supabase/ssr'
import { cookies } from 'next/headers'

export function createClient() {
  const cookieStore = cookies()

  return createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        get(name: string) {
          return cookieStore.get(name)?.value
        },
        set(name: string, value: string, options: CookieOptions) {
          try {
            cookieStore.set({ name, value, ...options })
          } catch (error) {
            // Handle cookie setting errors in Server Components
          }
        },
        remove(name: string, options: CookieOptions) {
          try {
            cookieStore.set({ name, value: '', ...options })
          } catch (error) {
            // Handle cookie removal errors
          }
        },
      },
    }
  )
}
```

*File: `src/lib/supabase/client.ts`*
```typescript
import { createBrowserClient } from '@supabase/ssr'

export function createClient() {
  return createBrowserClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
  )
}
```

---

## 🎨 Phase 2: White-Label Theming Engine (Day 1 PM)

### 2.1 Dynamic Theme Provider
We will use CSS variables injected from the database organization settings.

*File: `src/components/layout/theme-provider.tsx`*
```typescript
'use client'

import { useEffect, useState } from 'react'
import { createClient } from '@/lib/supabase/client'

interface OrgTheme {
  brandColor: string
  logoUrl: string
}

export function ThemeProvider({ children, orgSlug }: { children: React.ReactNode, orgSlug: string }) {
  const [theme, setTheme] = useState<OrgTheme | null>(null)
  const supabase = createClient()

  useEffect(() => {
    async function fetchTheme() {
      const { data } = await supabase
        .from('organizations')
        .select('brand_color, logo_url')
        .eq('slug', orgSlug)
        .single()
      
      if (data) {
        setTheme({
          brandColor: data.brand_color || '#0F172A',
          logoUrl: data.logo_url || 'https://globantislabs.com/wp-content/uploads/2026/01/logo-2x.png'
        })
        
        // Apply CSS Variables
        const root = document.documentElement
        root.style.setProperty('--primary', data.brand_color || '#0F172A')
      }
    }
    fetchTheme()
  }, [orgSlug])

  if (!theme) return <div className="flex h-screen items-center justify-center">Loading Workspace...</div>

  return (
    <div className="min-h-screen bg-background text-foreground" style={{ '--primary': theme.brandColor } as React.CSSProperties}>
      {children}
    </div>
  )
}
```

---

## 🔐 Phase 3: Authentication & Middleware (Day 2 AM)

### 3.1 Middleware Protection
*File: `src/middleware.ts`*
```typescript
import { createServerClient, type CookieOptions } from '@supabase/ssr'
import { NextResponse, type NextRequest } from 'next/server'

export async function middleware(request: NextRequest) {
  let response = NextResponse.next({
    request: { headers: request.headers },
  })

  const supabase = createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!,
    {
      cookies: {
        get(name: string) { return request.cookies.get(name)?.value },
        set(name: string, value: string, options: CookieOptions) {
          request.cookies.set({ name, value, ...options })
          response = NextResponse.next({ request: { headers: request.headers } })
          response.cookies.set({ name, value, ...options })
        },
        remove(name: string, options: CookieOptions) {
          request.cookies.set({ name, value: '', ...options })
          response = NextResponse.next({ request: { headers: request.headers } })
          response.cookies.set({ name, value: '', ...options })
        },
      },
    }
  )

  const { data: { session } } = await supabase.auth.getSession()

  // Protect Dashboard Routes
  if (request.nextUrl.pathname.startsWith('/dashboard') && !session) {
    return NextResponse.redirect(new URL('/login', request.url))
  }

  // Redirect logged-in users away from login
  if (request.nextUrl.pathname === '/login' && session) {
    return NextResponse.redirect(new URL('/dashboard', request.url))
  }

  return response
}

export const config = {
  matcher: ['/((?!_next/static|_next/image|favicon.ico|.*\\.(?:svg|png|jpg|jpeg|gif|webp)$).*)'],
}
```

---

## 🖥️ Phase 4: Dashboard Core Features (Day 2 PM - Day 3)

### 4.1 Layout Structure
*File: `src/app/(dashboard)/layout.tsx`*
- Sidebar (Collapsible, mobile responsive).
- Top Bar (User profile, Org switcher, Notifications).
- Main Content Area.

### 4.2 Project List & Kanban Board
- **Data Fetching:** Use Server Components for initial load.
- **Realtime Updates:** Use Supabase Realtime subscriptions.
- **Drag & Drop:** Use `@dnd-kit/core` and `@dnd-kit/sortable`.

### 4.3 Task Details Modal
- Use shadcn `Dialog` component.
- Form validation with `react-hook-form` and `zod`.
- Optimistic updates for instant UI feedback.

---

## 🚀 Phase 5: Deployment & CI/CD (Day 3 PM)

### 5.1 Vercel Configuration
1. Connect GitHub repo to Vercel.
2. Add Environment Variables in Vercel Dashboard.
3. Configure Build Command: `next build`.

### 5.2 Database Migrations in CI
- Use GitHub Actions to run `supabase db push` on PR merge.

---

## 🔮 Future Upgradability

1. **AI Integration:** Summarize projects with Vercel AI SDK.
2. **File Storage:** Supabase Storage for attachments.
3. **Real-time Chat:** Comment threads on tasks.
4. **Analytics:** Dashboard with `recharts`.
5. **Webhooks:** Slack/GitHub integrations.

---

## 📝 Developer Checklist

- [ ] Initialize Next.js project with TypeScript.
- [ ] Set up Supabase project and run migration SQL.
- [ ] Configure `.env.local` and verify DB connection.
- [ ] Implement Auth (Login/Logout/Middleware).
- [ ] Build White-label Theme Provider.
- [ ] Create Dashboard Layout (Sidebar/Header).
- [ ] Develop Project List View.
- [ ] Develop Kanban Board with Drag-and-Drop.
- [ ] Add Realtime subscriptions for live updates.
- [ ] Write basic Unit Tests (Vitest).
- [ ] Deploy to Vercel.
- [ ] Run Lighthouse audit (Target: 90+).

---

**Note to Team:** This plan prioritizes long-term maintainability and security. The "Hard Mode" approach ensures we own the IP, can scale infinitely, and provide a truly white-labeled experience for Globantis Labs.
