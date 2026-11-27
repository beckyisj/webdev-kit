# Web Dev Starter Kit

A bundle of 15 [Claude Code](https://claude.com/claude-code) skills for shipping web apps fast: 9 for the frontend, launch and getting found, 6 for the backend plumbing. Built by [Becky Isjwara](https://beckyisj.com) — a non-engineer who ships weekly.

Go from blank folder to live tool in an afternoon. Each skill is a slash command (`/prelaunch`, `/ship`, etc.) that tells Claude exactly what to do.

**Background reading for each backend skill → [beckyisj.com/webdev](https://beckyisj.com/webdev)**

## What's in the box

### Frontend, launch and getting found

| Command | What it does |
| --- | --- |
| `/landing` | Generate a responsive tool landing page (hero, features, FAQ, CTA, footer). Static HTML or Next.js. |
| `/favicon` | Complete favicon set: `.ico`, `.svg`, apple-touch-icon, 192/512 PNG. Matches your brand colors. |
| `/og-image` | 1200×630 social preview image designed for thumbnail readability (WhatsApp, Slack, iMessage). |
| `/seo` | sitemap.xml, robots.txt, llms.txt, JSON-LD schemas, meta tags. Framework-aware. |
| `/responsive` | Tests your app at mobile/tablet/desktop. Flags overflow, tiny touch targets, unreadable text. |
| `/prelaunch` | 8-category code audit: a11y, responsive, SEO, error states, performance, security. |
| `/ship` | Commit, push, deploy to Vercel, save project memory. One command. |
| `/chrome-ext` | Scaffold a Manifest V3 Chrome extension from a plain-English description. |
| `/aeo` | Get recommended by ChatGPT, Perplexity, Claude and Google AI. Baselines 15–25 buyer prompts, diagnoses each miss (empty SPA, no FAQ, no pricing, no listings), fixes the site, writes your directory listings, then retests the same prompts. |

### Backend plumbing

Each of these detects what the project already has, asks only for what's missing, writes the code, tells you exactly which dashboard and DNS steps are yours, then proves it works with a real test (a test email, a `dig`, a `curl`, a Stripe CLI trigger).

| Command | What it does |
| --- | --- |
| `/vercel` | First deploy: link the project, push env vars, `vercel --prod`, attach a domain, confirm 200. |
| `/domain` | Read current DNS, produce the exact records, registrar-specific steps (Cloudflare / Namecheap / GoDaddy / Vercel), verify propagation and SSL. Buying advice if you don't have one. |
| `/resend` | Transactional email: SDK, env, a send route, domain DNS verification, a real test send. |
| `/improvmx` | Free inbound forwarding so `hello@yourdomain` reaches your Gmail. Merges SPF with Resend correctly. |
| `/supabase` | Auth (magic link + callback), Postgres table with Row-Level Security, storage bucket, `@supabase/ssr` clients. Verified with a live query. |
| `/stripe` | Checkout Session, signature-verified webhook that grants access, customer portal, Stripe CLI local testing, go-live checklist. |

## The workflow

Chain them together from blank repo to live, paid product:

1. `/landing` — generate the marketing page
2. `/favicon` — brand the browser tab
3. `/og-image` — social preview image
4. `/seo` — sitemap, robots, JSON-LD
5. `/supabase` — auth and a database, if the tool needs accounts
6. `/stripe` — payments, if it charges
7. `/resend` + `/improvmx` — email out and in
8. `/responsive` — test all breakpoints
9. `/prelaunch` — full audit
10. `/vercel` — first deploy, then `/domain` to point your domain at it
11. `/ship` — every deploy after that
12. `/aeo` — once it's live, so AI search actually recommends it

## Install

Claude Code reads skills from `~/.claude/skills/`. Clone this repo in there:

```bash
cd ~/.claude/skills
git clone https://github.com/beckyisj/webdev-kit.git
```

Each subfolder contains a `SKILL.md` that Claude reads when you invoke the command.

**Requirements:** Claude Code. The backend skills expect the relevant CLI where one exists (`vercel`, `stripe`, optionally `supabase`) and tell you how to install it. `/favicon` and `/og-image` render with macOS built-ins (`qlmanage`, `sips`); on Linux/Windows swap in ImageMagick or `rsvg-convert`. `/responsive` drives a browser through the Playwright MCP plugin (`/plugin install playwright`) or the Claude in Chrome extension, and falls back to `npx playwright screenshot` if neither is present.

Updating: `cd ~/.claude/skills/webdev-kit && git pull`.

Type `/prelaunch` (or any other skill) in your Claude Code session. Claude reads the SKILL.md and knows exactly what to do.

## Who this is for

Creators and indie hackers who want to ship tools but don't want to become full-time engineers. If you've used Claude Code or Cursor before and you're tired of googling "how to set up Stripe webhooks", this is for you.

**Skip this** if you're a senior engineer — you'll have strong opinions about every choice and probably have your own skills already.

## Backend guides

The six backend skills each have a companion article that explains the *why* and the pricing in plain English. Read these if you want to understand what the skill is doing:

- [Resend](https://beckyisj.com/webdev/guides/resend) — transactional email
- [Vercel](https://beckyisj.com/webdev/guides/vercel) — deployment
- [Domains](https://beckyisj.com/webdev/guides/domain) — GoDaddy / Namecheap / Cloudflare
- [Supabase](https://beckyisj.com/webdev/guides/supabase) — auth, database, storage
- [Stripe](https://beckyisj.com/webdev/guides/stripe) — payments

## Rather skip the DIY?

I build tools like this for creators as a service. [Book a 30-min call](https://go.beckyisj.com/30min) or [work with me](https://beckyisj.com/work-with-me).

## License

MIT. Use these, fork them, adapt them — whatever makes your workflow better.

If a skill saves you an afternoon, a ⭐ on the repo is appreciated.
