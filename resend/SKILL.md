---
name: resend
description: Wire transactional email into a web app with Resend — install the SDK, add the env var, scaffold a send route (Next.js API route or a plain Node helper), walk the domain DNS verification, and send a real test email. Use when an app needs to send magic links, receipts, contact-form submissions, or notifications.
user-invocable: true
allowed-tools: Read, Write, Edit, Bash, Glob, Grep
argument-hint: [project-path] [sending-domain]
---

# Resend — transactional email

Get an app sending email from its own domain, end to end, and prove it with a real delivery.

**Argument**: `$ARGUMENTS` is the project path and optionally the sending domain (e.g. `~/projects/my-app mytool.app`). Default to the current directory. Ask for the domain only if nothing in the project (`.env*`, `package.json` homepage, existing `og:url`) reveals it.

Resend only **sends**. If the user also wants `hello@domain` to receive replies, run `/improvmx` afterwards and merge the SPF records (see Gotchas).

## Process

### 1. Detect state

```bash
cat package.json | grep -E '"(next|resend|react-email|@react-email/components)"'
grep -l RESEND_API_KEY .env .env.local .env.example 2>/dev/null
ls src/app/api 2>/dev/null; ls pages/api 2>/dev/null
```

Decide the target from what you find:
- **Next.js App Router** → `src/app/api/send/route.ts` (or `app/api/send/route.ts` if there is no `src/`)
- **Next.js Pages Router** → `pages/api/send.ts`
- **Anything else (Vite, Astro, Express, plain Node)** → `lib/email.ts` helper plus a note that the send must happen server-side; the API key can never ship to the browser

### 2. Install and configure

```bash
npm install resend
```

Add to `.env.local` (create it if missing) and to `.env.example` with the value blanked:
```
RESEND_API_KEY=re_xxxxxxxx
EMAIL_FROM="Your Name <hello@yourdomain.com>"
```

Make sure `.env.local` is in `.gitignore`. If it isn't, add it before anything else.

### 3. Scaffold the send code

**App Router route** (`src/app/api/send/route.ts`):
```typescript
import { Resend } from 'resend';

const resend = new Resend(process.env.RESEND_API_KEY);

export async function POST(req: Request) {
  const { to, subject, html, replyTo } = await req.json();
  if (!to || !subject || !html) {
    return Response.json({ error: 'to, subject and html are required' }, { status: 400 });
  }
  const { data, error } = await resend.emails.send({
    from: process.env.EMAIL_FROM!,
    to,
    subject,
    html,
    replyTo,
  });
  if (error) return Response.json({ error }, { status: 500 });
  return Response.json({ id: data?.id });
}
```

**Generic helper** (`lib/email.ts`) for non-Next projects: same `resend.emails.send` call wrapped in an exported `sendEmail({ to, subject, html })` function.

If the user described a specific email (contact form, welcome, receipt), write that concrete route instead of the generic one, and add a `react-email` template under `emails/` when the body is more than a paragraph. Hand-written HTML email breaks in Outlook; `@react-email/components` renders inline-styled HTML that survives every client.

### 4. Domain verification (the human part)

Tell the user exactly what to do, in order. You cannot do the dashboard steps for them:

1. Sign up at resend.com, then **Domains → Add Domain** and enter the sending domain.
2. Resend shows three DNS records: an MX for bounces, a TXT for SPF, and a TXT for DKIM. Copy each into the registrar's DNS panel (run `/domain` if they don't know where that is).
3. On Cloudflare, every one of these must be **DNS only** (grey cloud). Proxied records break email.
4. Wait about five minutes, click **Verify**.
5. **API Keys → Create API Key**, sending-only permission, paste it into `.env.local`.

Until the domain is verified, the user can still test: Resend lets you send from `onboarding@resend.dev` to your own signup email only.

### 5. Verify with a real send

Start the dev server if it isn't running, then:
```bash
curl -s -X POST http://localhost:3000/api/send \
  -H 'content-type: application/json' \
  -d '{"to":"delivered@resend.dev","subject":"Resend wired up","html":"<p>It works.</p>"}'
```

`delivered@resend.dev` is Resend's sink address: it shows in the dashboard under Emails without delivering anywhere. A returned `id` means the whole chain works. Then send one to the user's real inbox so they can mark it "not spam" once; Gmail learns from that and later mail lands in the inbox.

### 6. Report

```
Resend
- SDK: installed (resend@x.y.z)
- Route: src/app/api/send/route.ts
- Env: RESEND_API_KEY set locally · add to Vercel → Settings → Environment Variables before deploying
- Domain: mytool.app — verified / pending DNS
- Test send: id re_abc123 → dashboard
- Next: /improvmx for inbound, or set Resend as SMTP in Supabase Auth if using magic links
```

## Gotchas

- **Only one SPF record per domain.** If ImprovMX (or Google Workspace) is also on the domain, merge into a single TXT: `v=spf1 include:spf.improvmx.com include:_spf.resend.com ~all`. Two `v=spf1` records break both services.
- **DMARC.** Resend nags for it. For a small project `v=DMARC1; p=none; rua=mailto:you@yourdomain.com` as a TXT at `_dmarc` is enough.
- **`from` must be on the verified domain.** Any local part works (`hello@`, `no-reply@`); no mailbox needs to exist.
- **Free tier**: 3,000 emails/month, 100/day. Pro is $20/month for 50,000. Marketing newsletters belong in a newsletter tool, not here.
- **Vercel**: the env var has to be added in the Vercel dashboard too. A working local send and a failing production send is almost always that.
