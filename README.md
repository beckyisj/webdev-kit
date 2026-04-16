# Web Dev Starter Kit

A bundle of 8 [Claude Code](https://claude.com/claude-code) skills for shipping web apps fast. Built by [Becky Isjwara](https://beckyisj.com) — a non-engineer who ships weekly.

Go from blank folder to live tool in an afternoon. Each skill is a slash command (`/prelaunch`, `/ship`, etc.) that tells Claude exactly what to do.

**Paired with plain-English backend guides → [beckyisj.com/webdev](https://beckyisj.com/webdev)**

## What's in the box

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

## The workflow

Chain them together from blank repo to live tool:

1. `/landing` — generate the marketing page
2. `/favicon` — brand the browser tab
3. `/og-image` — social preview image
4. `/seo` — sitemap, robots, JSON-LD
5. `/responsive` — test all breakpoints
6. `/prelaunch` — full audit
7. `/ship` — deploy

## Install

Claude Code reads skills from `~/.claude/skills/`. Clone this repo in there:

```bash
cd ~/.claude/skills
git clone https://github.com/beckyisj/webdev-kit.git
```

Each subfolder contains a `SKILL.md` that Claude reads when you invoke the command.

Type `/prelaunch` (or any other skill) in your Claude Code session. Claude reads the SKILL.md and knows exactly what to do.

## Who this is for

Creators and indie hackers who want to ship tools but don't want to become full-time engineers. If you've used Claude Code or Cursor before and you're tired of googling "how to set up Stripe webhooks", this is for you.

**Skip this** if you're a senior engineer — you'll have strong opinions about every choice and probably have your own skills already.

## Backend guides

These skills cover the frontend. For the backend plumbing (email, payments, database, domains, deployment), I wrote plain-English guides:

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
