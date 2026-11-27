# Changelog

## 2.1.0 — 2026-11-27

One new skill, built from a real AEO pass on a live product whose baseline was 1 of 20 buyer prompts.

- **`/aeo`** — answer-engine optimisation as a measurement loop: write buyer prompts, take a baseline, diagnose each miss with a no-JavaScript fetch, fix the site (prerender a Vite SPA landing page, visible FAQ + FAQPage from one data file, SoftwareApplication with every plan's price, one static page per job, an honest /compare page), write paste-ready directory listings, retest the same prompts about 4 weeks later.
- **`/seo`**: robots.txt now allows the live-fetch assistant agents (`ChatGPT-User`, `Claude-User`, `Perplexity-User`) plus `OAI-SearchBot`, `Google-Extended` and `Applebot-Extended`; flags client-rendered SPAs that serve crawlers an empty `#root`; adds a FAQPage template and pricing/"vs X" sections to llms.txt; reminds you to submit to Bing Webmaster Tools as well as Search Console.

## 2.0.0 — 2026-09-06

Six new backend skills. The kit now covers the backend the guides used to only describe.

- **`/vercel`** — first deploy: link, env sync, `vercel --prod`, domain attach, verify.
- **`/domain`** — DNS read, exact records, registrar-specific steps, propagation and SSL check, buying advice.
- **`/resend`** — transactional email with a scaffolded send route and a real test send to Resend's sink address.
- **`/improvmx`** — inbound forwarding with correct SPF merging when Resend is also present.
- **`/supabase`** — `@supabase/ssr` clients, magic-link callback, RLS-on table, storage, live-query verification. Uses the current publishable/secret key names, with the legacy anon/service_role pair noted.
- **`/stripe`** — Checkout Session, signature-verified webhook (grants access there, never on the redirect), customer portal, Stripe CLI testing, go-live checklist.

Every backend skill follows the same shape: detect, ask only for what's missing, scaffold, hand over the dashboard/DNS steps that need a human, verify with a real request, report.


## 1.1.0 — 2026-09-06

Refresh for current Claude Code. No new skills; every existing one was audited against the September 2026 harness and models.

- **responsive**: rewritten off the retired Claude Preview MCP onto the Playwright MCP plugin, with a Chrome-extension path and a `npx playwright screenshot` fallback.
- **ship**: no longer hardcodes a model name in the commit trailer (uses the one the harness supplies); project notes go to Claude auto-memory, `CLAUDE.md`, or `SHIPLOG.md` instead of an author-specific path.
- **seo**: robots.txt allowlists the current Anthropic crawlers (`ClaudeBot`, `Claude-SearchBot`) instead of the retired `Claude-Web`; dropped the dead Twitter Card Validator; sitemap dates are placeholders, not April 2026.
- **All skills**: frontmatter uses the documented `user-invocable` key (the `user_invocable` spelling was silently ignored); removed author-specific example paths and brand references; dropped the "Want me to fix any of these?" closers and hard caps on findings, which newer models handle better without.
- **prelaunch / responsive**: reports lead with launch-blockers and fold duplicate findings, rather than truncating at 5 items.

## 1.0.0 — 2026-04-21

Initial release: 8 skills (`/landing`, `/favicon`, `/og-image`, `/seo`, `/responsive`, `/prelaunch`, `/ship`, `/chrome-ext`).
