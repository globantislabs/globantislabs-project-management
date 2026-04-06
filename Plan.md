# Globantis Labs: Project Management & Team Collaboration Dashboard

## 🚀 Executive Summary
**Goal:** Build a **white-labeled, fully custom, feature-upgradable** project management dashboard for Globantis Labs.
**Philosophy:** "Minimal Code" ≠ "No Control". We use **high-level abstractions** (Next.js App Router, Supabase CLI, shadcn/ui) to get 100% code ownership with ~80% less boilerplate.
**Tech Stack:**
- **Frontend:** Next.js 14 (App Router), TypeScript, Tailwind CSS, **shadcn/ui** (Copy-paste components, full control).
- **Backend & Auth:** **Supabase** (PostgreSQL, Realtime, Auth, Storage).
- **Hosting:** **Vercel** (Optimized for Next.js) or Netlify.
- **White-labeling:** Dynamic theming via CSS variables and config files.

---

## 🛠 Phase 1: Instant Setup (5 Minutes)
*Run these commands to scaffold the entire project structure.*

```bash
# 1. Create Next.js App with TypeScript and Tailwind
npx create-next-app@latest globantis-dashboard --typescript --tailwind --eslint --app --src-dir --import-alias "@/*" --use-npm

# 2. Enter directory
cd globantis-dashboard

# 3. Initialize shadcn/ui (The secret to minimal UI coding)
# Select "Default" style, "Slate" base color, yes to CSS variables
npx shadcn-ui@latest init

# 4. Install Core Components (Dashboard, Tables, Forms, Auth)
npx shadcn-ui@latest add button card input label table avatar dropdown-menu dialog form sheet toast badge

# 5. Install Supabase Client
npm install @supabase/supabase-js @supabase/ssr lucide-react clsx tailwind-merge
```

---

## 🗄 Phase 2: Supabase Backend (SQL Only, No Backend Code)
*Go to [supabase.com](https://supabase.com), create a project, and run this in the **SQL Editor**. This sets up Auth, Tables, Row Level Security (RLS), and Realtime.*

```sql
-- 1. Enable UUID extension
create extension if not exists "uuid-ossp";

-- 2. Create Profiles Table (Extends Auth)
create table profiles (
  id uuid references auth.users not null primary key,
  full_name text,
  avatar_url text,
  role text check (role in ('admin', 'manager', 'member')) default 'member',
  updated_at timestamp with time zone default timezone('utc'::text, now())
);

-- 3. Create Projects Table
create table projects (
  id uuid default uuid_generate_v4() primary key,
  name text not null,
  description text,
  status text check (status in ('active', 'on-hold', 'completed')) default 'active',
  created_by uuid references profiles(id),
  created_at timestamp with time zone default timezone('utc'::text, now())
);

-- 4. Create Tasks Table
create table tasks (
  id uuid default uuid_generate_v4() primary key,
  project_id uuid references projects(id) on delete cascade,
  title text not null,
  status text check (status in ('todo', 'in-progress', 'review', 'done')) default 'todo',
  assignee_id uuid references profiles(id),
  due_date date,
  created_at timestamp with time zone default timezone('utc'::text, now())
);

-- 5. Enable Row Level Security (RLS) - Critical for Multi-tenant Safety
alter table profiles enable row level security;
alter table projects enable row level security;
alter table tasks enable row level security;

-- 6. Policies (Simple: Logged-in users can see everything. Adjust for strict isolation later)
create policy "Public profiles are viewable by everyone" on profiles for select using (true);
create policy "Users can update own profile" on profiles for update using (auth.uid() = id);

create policy "Projects viewable by logged in users" on projects for select using (auth.role() = 'authenticated');
create policy "Projects manageable by managers/admins" on projects for all using (
  exists (select 1 from profiles where id = auth.uid() and role in ('admin', 'manager'))
);

create policy "Tasks viewable by logged in users" on tasks for select using (auth.role() = 'authenticated');
create policy "Tasks manageable by team members" on tasks for all using (auth.role() = 'authenticated');

-- 7. Trigger to create profile on signup
create or replace function public.handle_new_user() 
returns trigger as $$
begin
  insert into public.profiles (id, full_name, avatar_url)
  values (new.id, new.raw_user_meta_data->>'full_name', new.raw_user_meta_data->>'avatar_url');
  return new;
end;
$$ language plpgsql security definer;

create trigger on_auth_user_created
  after insert on auth.users
  for each row execute procedure public.handle_new_user();
```

---

## 💻 Phase 3: Minimal Code Implementation (~200 Lines Total)

### 1. Environment Variables (`.env.local`)
```env
NEXT_PUBLIC_SUPABASE_URL=your_supabase_url
NEXT_PUBLIC_SUPABASE_ANON_KEY=your_supabase_anon_key
```

### 2. Supabase Client (`lib/supabase.ts`)
*Handles Server and Client connections securely.*
```typescript
import { createBrowserClient, createServerClient } from '@supabase/ssr'

export function createClient() {
  return createBrowserClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_ANON_KEY!
  )
}
// Note: For server actions, use createServerClient (omitted for brevity, standard pattern)
```

### 3. White-Label Config (`config/site.ts`)
*Change colors/logo here to rebrand instantly.*
```typescript
export const siteConfig = {
  name: "Globantis Labs Dashboard",
  logo: "/logo.png", // Place fetched logo in public folder
  primaryColor: "hsl(222.2 47.4% 11.2%)", // Custom Globantis Blue
  features: {
    realtime: true,
    analytics: true,
    teamChat: false // Toggle features easily
  }
}
```

### 4. Main Dashboard Page (`app/dashboard/page.tsx`)
*Uses shadcn components. Zero custom CSS needed.*
```tsx
'use client'
import { useEffect, useState } from 'react'
import { createClient } from '@/lib/supabase'
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card"
import { Button } from "@/components/ui/button"
import { Badge } from "@/components/ui/badge"

export default function Dashboard() {
  const [projects, setProjects] = useState<any[]>([])
  const supabase = createClient()

  useEffect(() => {
    // Fetch Initial Data
    const fetchProjects = async () => {
      const { data } = await supabase.from('projects').select('*')
      if (data) setProjects(data)
    }
    fetchProjects()

    // Subscribe to Realtime Changes (Upgradable Feature)
    const channel = supabase.channel('public:projects')
      .on('postgres_changes', { event: '*', schema: 'public', table: 'projects' }, 
        (payload) => {
          if (payload.eventType === 'INSERT') setProjects(prev => [...prev, payload.new])
          if (payload.eventType === 'UPDATE') setProjects(prev => prev.map(p => p.id === payload.new.id ? payload.new : p))
        })
      .subscribe()
    
    return () => { supabase.removeChannel(channel) }
  }, [])

  return (
    <div className="p-8 space-y-8">
      <h1 className="text-3xl font-bold">Project Overview</h1>
      <div className="grid gap-4 md:grid-cols-2 lg:grid-cols-3">
        {projects.map((project) => (
          <Card key={project.id}>
            <CardHeader>
              <CardTitle>{project.name}</CardTitle>
            </CardHeader>
            <CardContent>
              <p className="text-sm text-gray-500">{project.description}</p>
              <div className="mt-4 flex justify-between items-center">
                <Badge variant={project.status === 'active' ? 'default' : 'secondary'}>
                  {project.status}
                </Badge>
                <Button size="sm">Manage</Button>
              </div>
            </CardContent>
          </Card>
        ))}
      </div>
    </div>
  )
}
```

### 5. Login Page (`app/login/page.tsx`)
*Simple Email/Password Auth.*
```tsx
'use client'
import { useState } from 'react'
import { createClient } from '@/lib/supabase'
import { useRouter } from 'next/navigation'
import { Button } from "@/components/ui/button"
import { Input } from "@/components/ui/input"

export default function Login() {
  const [email, setEmail] = useState('')
  const [password, setPassword] = useState('')
  const router = useRouter()
  const supabase = createClient()

  const handleLogin = async (e: React.FormEvent) => {
    e.preventDefault()
    const { error } = await supabase.auth.signInWithPassword({ email, password })
    if (!error) router.push('/dashboard')
    else alert(error.message)
  }

  return (
    <div className="flex h-screen items-center justify-center bg-gray-50">
      <form onSubmit={handleLogin} className="space-y-4 w-96 p-8 bg-white rounded-lg shadow-md">
        <h2 className="text-2xl font-bold text-center">Globantis Labs Login</h2>
        <Input type="email" placeholder="Email" value={email} onChange={e => setEmail(e.target.value)} />
        <Input type="password" placeholder="Password" value={password} onChange={e => setPassword(e.target.value)} />
        <Button type="submit" className="w-full">Sign In</Button>
      </form>
    </div>
  )
}
```

---

## 🎨 Phase 4: White-Labeling & Theming
To ensure full control and easy rebranding:
1.  **Logo:** Download the logo from `https://globantislabs.com/wp-content/uploads/2026/01/logo-2x.png` and save as `public/logo.png`.
2.  **Colors:** Edit `app/globals.css`. The `shadcn` setup uses CSS variables. Change `--primary` to the exact Hex/HSL of Globantis branding.
    ```css
    :root {
      --primary: 221 83% 53%; /* Example Globantis Blue */
      --primary-foreground: 210 40% 98%;
      /* ... other variables */
    }
    ```
3.  **Features:** Toggle features in `config/site.ts` to hide/show modules without deleting code.

---

## 🚀 Phase 5: Deployment (Vercel)
1.  Push code to GitHub.
2.  Go to [Vercel](https://vercel.com) -> **Import Project**.
3.  Add Environment Variables (`NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_ANON_KEY`).
4.  Click **Deploy**.
5.  *Done.* Global CDN, SSL, and Auto-scaling included.

---

## 📈 Future Upgradability Path
Since you own the code, adding features is modular:
-   **Team Chat:** Add a `messages` table in Supabase + reuse the Realtime logic in `dashboard/page.tsx`.
-   **File Uploads:** Enable Supabase Storage + use `upload` component from shadcn.
-   **Analytics:** Connect Supabase to a BI tool or build a simple chart page using `recharts` (install with `npm install recharts`).

## ✅ Checklist for Fast Launch
- [ ] Run Phase 1 Commands (5 mins)
- [ ] Execute SQL in Supabase (2 mins)
- [ ] Copy/Paste Phase 3 Code (10 mins)
- [ ] Add Logo & Tweak Colors (5 mins)
- [ ] Deploy to Vercel (2 mins)
- **Total Time:** ~25 Minutes
- **Total Custom Code:** ~150 Lines
