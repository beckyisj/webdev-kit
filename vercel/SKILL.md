---
name: vercel
description: Take a web app from local to live on Vercel — link the project, sync environment variables from .env.local, deploy to production, attach a custom domain, and confirm the live URL responds. Use for a first deploy or when a project's Vercel setup is half-done. (/ship handles the routine commit-push-deploy loop once this is in place.)
user-invocable: true
allowed-tools: Read, Write, Edit, Bash, Glob, Grep
argument-hint: [project-path] [domain]
---

# Vercel — first deploy

Set up hosting once, properly, so `/ship` can do the routine deploys afterwards.

**Argument**: `$ARGUMENTS` is the project path and optionally the custom domain. Default to the current directory.

## Process

### 1. Detect state

```bash
which vercel || echo "no vercel cli"
vercel whoami 2>&1
ls .vercel/project.json vercel.json 2>/dev/null
git remote -v
cat package.json | grep -E '"(build|dev|start)"'
grep -v '^#' .env.local 2>/dev/null | cut -d= -f1
```

This tells you: is the CLI installed and logged in, is the project already linked, is there a GitHub remote (Vercel's Git integration wants one), what the framework is, and which env vars will need to exist in production.

### 2. Install and log in (once)

```bash
npm i -g vercel
vercel login
```

`vercel login` opens a browser. The user has to click through it; tell them to type `! vercel login` if they are driving this from a Claude Code session so the prompt lands in their terminal.

### 3. Link the project

```bash
vercel link --yes
```

Accept the detected framework. This writes `.vercel/project.json`; add `.vercel` to `.gitignore` if it isn't there.

### 4. Environment variables

For every key found in `.env.local` in step 1, push it to all three environments:
```bash
vercel env add <KEY> production
vercel env add <KEY> preview
vercel env add <KEY> development
```
Each command prompts for the value; the user pastes it. Never read the secret values into the conversation yourself. Keys prefixed `NEXT_PUBLIC_` or `VITE_` are bundled into the browser, so double-check none of those hold a secret.

Later, to get production vars onto a new machine: `vercel env pull .env.local`.

### 5. Deploy

```bash
vercel --prod
```

Read the output. A URL means success. If the build fails, the log is readable: nine times out of ten it is a TypeScript error that `npm run dev` tolerated, or an env var missing in production. Fix and re-run.

### 6. Custom domain (if given)

```bash
vercel domains add <domain>
```

Vercel prints the DNS record it wants: an **A record** `@ → 76.76.21.21` for the apex, and a per-project **CNAME** for `www` or any subdomain (looks like `abc123.vercel-dns-017.com`; copy it from the output, it is not the same for every project). Hand those to `/domain` for the registrar-specific steps, or if the registrar is Vercel itself, it's already done.

On Cloudflare the A record must be **DNS only** (grey cloud) or Vercel can't issue the SSL certificate.

Confirm with:
```bash
vercel domains verify <domain>
```

### 7. Verify

```bash
curl -sI https://<deployment-url> | head -1
curl -sI https://<domain> | head -1        # if a domain was added
```

Expect `HTTP/2 200`. A `307` on the apex that lands on `www` (or vice versa) is Vercel's redirect and is fine.

### 8. Report

```
Vercel
- Project: my-app (linked, framework: Next.js)
- Env: 4 vars pushed to production/preview/development
- Deploy: https://my-app-abc123.vercel.app — 200
- Domain: mytool.app — DNS pending at Namecheap (A @ 76.76.21.21)
- Next: /ship for future deploys · Settings → Git to enable preview URLs per PR
```

## Gotchas

- **Hobby tier function timeout is 10 seconds.** A route that calls an LLM or processes a file will hit it. Options: Pro (60s), or `export const maxDuration = 60` with Fluid Compute, or move the work to a background job.
- **Cold starts** on serverless functions add about a second after idle. Fine for most apps.
- **Preview deploys from PRs** need the GitHub integration: Vercel dashboard → Project → Settings → Git → Connect. `vercel --prod` from the CLI works without it.
- **Pricing**: Hobby is free (100 GB bandwidth, personal use only). Pro is $20/month per member and required for anything commercial.
- **Analytics** is one import away: `npm i @vercel/analytics`, then `<Analytics />` from `@vercel/analytics/next` in the root layout. Free for 2,500 events/month on Hobby.
