---
name: supabase
description: Add a Supabase backend to a Next.js app — env vars, the @supabase/ssr browser/server clients, an auth callback route, a first table with Row-Level Security policies, optional storage bucket, and a typed query example. Verifies the connection with a real query. Use when an app needs auth, a database, or file uploads.
user-invocable: true
allowed-tools: Read, Write, Edit, Bash, Glob, Grep
argument-hint: [project-path] [what-you-need: auth|db|storage]
---

# Supabase — auth, Postgres, storage

Set up the pieces the app actually needs and prove each one works. Don't scaffold auth for a project that only wants a table.

**Argument**: `$ARGUMENTS` is the project path and what's needed. Default to the current directory; if the need isn't stated, ask which of auth / database / storage applies.

This skill assumes Next.js App Router, which is where `@supabase/ssr` earns its keep. For Vite or plain React, use `createClient` from `@supabase/supabase-js` directly and skip the server client and callback route.

## Process

### 1. Detect state

```bash
cat package.json | grep -E '"(next|@supabase/supabase-js|@supabase/ssr)"'
grep -l SUPABASE .env.local .env.example 2>/dev/null
ls src/lib/supabase* src/utils/supabase* lib/supabase* 2>/dev/null
ls src/app/auth 2>/dev/null
which supabase || echo "no supabase cli"
```

### 2. Project and keys (the human part)

If there's no `NEXT_PUBLIC_SUPABASE_URL` yet:
1. supabase.com → **New project**. Pick the region closest to the users. It takes about two minutes to provision.
2. **Settings → API Keys**. New projects show a **publishable** key (`sb_publishable_…`) and a **secret** key (`sb_secret_…`). Older projects show the legacy `anon` and `service_role` JWTs; they behave the same way and the code below works with either pair.
3. Paste into `.env.local`; add blanked copies to `.env.example`:
```
NEXT_PUBLIC_SUPABASE_URL=https://xxxx.supabase.co
NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY=sb_publishable_...   # or the legacy anon key
SUPABASE_SECRET_KEY=sb_secret_...                          # or the legacy service_role key
```

The secret / `service_role` key bypasses every security rule. It goes in server code only, never in anything prefixed `NEXT_PUBLIC_`, never in git. Check `.gitignore` covers `.env.local` before writing the file.

### 3. Install and create the clients

```bash
npm install @supabase/supabase-js @supabase/ssr
```

`src/lib/supabase/client.ts` (browser):
```typescript
import { createBrowserClient } from '@supabase/ssr';

export function createClient() {
  return createBrowserClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY!
  );
}
```

`src/lib/supabase/server.ts` (server components, route handlers, server actions):
```typescript
import { createServerClient } from '@supabase/ssr';
import { cookies } from 'next/headers';

export async function createClient() {
  const cookieStore = await cookies();
  return createServerClient(
    process.env.NEXT_PUBLIC_SUPABASE_URL!,
    process.env.NEXT_PUBLIC_SUPABASE_PUBLISHABLE_KEY!,
    {
      cookies: {
        getAll: () => cookieStore.getAll(),
        setAll: (toSet) => {
          try {
            toSet.forEach(({ name, value, options }) => cookieStore.set(name, value, options));
          } catch {
            // called from a Server Component; middleware refreshes the session instead
          }
        },
      },
    }
  );
}
```

Using the publishable key with a plain `createClient` on the server loses the logged-in user; every query then runs as anonymous and RLS returns nothing. That's why the server client reads cookies.

### 4. Auth (if needed)

**Callback route** `src/app/auth/callback/route.ts`:
```typescript
import { NextResponse } from 'next/server';
import { createClient } from '@/lib/supabase/server';

export async function GET(request: Request) {
  const { searchParams, origin } = new URL(request.url);
  const code = searchParams.get('code');
  const next = searchParams.get('next') ?? '/';
  if (code) {
    const supabase = await createClient();
    const { error } = await supabase.auth.exchangeCodeForSession(code);
    if (!error) return NextResponse.redirect(`${origin}${next}`);
  }
  return NextResponse.redirect(`${origin}/login?error=auth`);
}
```

**Magic link sign-in** (client component):
```typescript
const { error } = await supabase.auth.signInWithOtp({
  email,
  options: { emailRedirectTo: `${location.origin}/auth/callback` },
});
```

**Middleware** `src/middleware.ts` to refresh sessions: use the `updateSession` helper from the Supabase Next.js docs verbatim; it's about 40 lines and changes with `@supabase/ssr` versions, so fetch the current one rather than reproducing a stale copy here.

Dashboard steps for the user: **Authentication → URL Configuration** → add `http://localhost:3000/auth/callback` and the production URL to Redirect URLs. During development, **Authentication → Providers → Email → Confirm email** off saves a lot of inbox-checking. For production auth emails, set Resend as the custom SMTP under **Authentication → SMTP Settings** (run `/resend` first); Supabase's default sender is rate-limited to a handful an hour and lands in spam.

### 5. Database (if needed)

Write the schema as SQL the user can paste into **SQL Editor**, or run with the CLI if `supabase` is installed and linked. Shape it to the app; this is the pattern:

```sql
create table notes (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references auth.users on delete cascade,
  title text not null,
  body text,
  created_at timestamptz default now()
);

alter table notes enable row level security;

create policy "read own" on notes for select using (auth.uid() = user_id);
create policy "insert own" on notes for insert with check (auth.uid() = user_id);
create policy "update own" on notes for update using (auth.uid() = user_id);
create policy "delete own" on notes for delete using (auth.uid() = user_id);
```

RLS on, with policies, every time. A table without RLS is readable by anyone with the publishable key, which is everyone who loads the site. For public read-only data, add `create policy "public read" on t for select using (true)` deliberately rather than leaving RLS off.

Generate types if the CLI is available:
```bash
supabase gen types typescript --project-id <ref> > src/lib/supabase/types.ts
```

### 6. Storage (if needed)

Dashboard: **Storage → New bucket**. Public bucket for avatars and images that appear in the UI; private bucket plus signed URLs for anything user-specific. Upload pattern:
```typescript
const { data, error } = await supabase.storage
  .from('uploads')
  .upload(`${user.id}/${file.name}`, file, { upsert: true });
```
Storage has its own RLS on `storage.objects`; add a policy scoping writes to `auth.uid()::text = (storage.foldername(name))[1]`.

### 7. Verify

Start the dev server and hit a real query. Simplest is a temporary server component or a route:
```typescript
const supabase = await createClient();
const { data, error, count } = await supabase.from('notes').select('*', { count: 'exact', head: true });
```
Expect `error: null`. If auth is set up, sign in with a magic link, land on the callback, and confirm `supabase.auth.getClaims()` returns the user's claims (it validates the JWT locally; `getUser()` also works but makes a network call every time). Remove the test route afterwards.

### 8. Report

```
Supabase
- Project: xxxx.supabase.co (region: eu-west-2)
- Clients: src/lib/supabase/{client,server}.ts
- Auth: magic link + /auth/callback · redirect URLs added · SMTP → Resend
- DB: notes table, RLS on, 4 policies
- Storage: bucket "uploads" (private)
- Verify: select on notes → error null · sign-in round trip OK
- Env: 3 vars local · add all three to Vercel (the NEXT_PUBLIC_ ones ship to the browser by design)
```

## Gotchas

- **RLS failures are silent.** Empty array, no error. Policies are the first thing to check when a query "returns nothing".
- **Free projects pause after 7 days idle.** Dashboard login wakes them. Pro ($25/month) doesn't pause and adds daily backups.
- **`exchangeCodeForSession` needs PKCE**, which `@supabase/ssr` enables by default. If the callback loops, the redirect URL is usually missing from URL Configuration.
- **Firebase is the alternative** and its NoSQL model is hard to migrate away from. Supabase is real Postgres; you can always take the data elsewhere.
