---
name: aeo
description: Answer-engine optimisation (AEO) for a product site, run as a measurement loop. Write the prompts buyers type into ChatGPT, Perplexity, Claude and Google AI, take a baseline of whether the product shows up, diagnose each miss, fix the site, build third-party listings, then retest the same prompts. Use when a product is live but AI search never recommends it, or when the user says "AEO", "GEO", "get cited by ChatGPT" or "show up in AI answers".
user-invocable: true
allowed-tools: Read, Write, Edit, Bash, Glob, Grep, WebSearch, WebFetch
argument-hint: [project-path or product URL]
---

# AEO: get recommended by AI search

`/seo` makes a site crawlable. This skill makes it the answer. It runs as a loop, not a one-off:

**prompts → baseline → diagnose → fix the site → earn third-party mentions → retest**

AI answers are mostly built from pages that already rank or get cited: your own site, directories, comparison posts, Reddit, app-store listings. So the work is half on-site and half off-site, and you only know it worked if you measure the same prompts before and after.

**Argument**: `$ARGUMENTS` is the project path or the live URL. If neither, ask.

## 1. Gather the facts first

Read the site and the codebase, then confirm with the user (ask only for what's missing):

- **What it does**, in one sentence, in the words a buyer would use (not the marketing tagline)
- **Pricing**: free tier limits, paid price, trial
- **Who it's for** and **who it's not for**
- **Main alternatives** people compare it with, and the one honest difference from each
- **Every job the product does.** A tool that does five jobs needs to be findable for all five

These facts go into the FAQ, schema, llms.txt and listings, so they must be exact. Never invent a number, a feature or a customer count.

## 2. Write 15–25 buyer prompts

Write the prompts a buyer types **before they know the product exists**. Mix these types:

| Type | Example |
| --- | --- |
| Job to be done | "how do I turn a podcast episode into a LinkedIn carousel" |
| Best tool for X | "best tool to make Instagram carousels from YouTube videos" |
| Alternative to | "cheaper alternative to [competitor]" |
| Comparison | "[competitor A] vs [competitor B] for [job]" |
| Problem phrasing | "screenshots from a YouTube video without downloading it" |
| Brand | "what is [product]", "[product] pricing" |

Show the list to the user before running it. They know what their buyers actually say.

## 3. Take the baseline

Run every prompt with WebSearch (send them in parallel). For each, record:

- **Present?** Is the product named or linked in the results?
- **Which URL** surfaced it (homepage, a guide, a store listing, a directory)?
- **Who won instead**, and what kind of page (listicle, Reddit thread, competitor's blog)?
- **"Nobody does this"** answers, where the results say no tool exists for a job the product already does. These are the cheapest wins.

Save it to `docs/aeo/baseline-YYYY-MM-DD.md` as a table plus a one-line score ("present in 1/20"). Be upfront that WebSearch is a proxy for real ChatGPT/Perplexity answers, not the same thing. If the user can paste in a few answers from the real assistants, add them.

## 4. Diagnose each miss

Fetch the live pages the way a crawler sees them, **without JavaScript**:

```bash
# Word count a crawler gets. An SPA often returns ~20 words: just the <title> and an empty #root.
curl -sL https://[domain]/ | sed 's/<script.*<\/script>//g; s/<[^>]*>/ /g' | wc -w

# Is there FAQ / product schema?
curl -sL https://[domain]/ | grep -o '"@type": *"[A-Za-z]*"' | sort | uniq -c

# Are AI crawlers allowed?
curl -sL https://[domain]/robots.txt
curl -sL https://[domain]/llms.txt | head -30
```

Common causes, roughly in order of how often they matter:

1. **The page is empty without JS** (client-rendered React/Vite SPA). Crawlers and AI fetchers see nothing. Fix this first; nothing else counts until it's done.
2. **No page answers the prompt.** The product does the job but nothing on the site says so in the buyer's words.
3. **The facts are missing**: no visible price, no free-tier limits, no "vs X". Assistants won't recommend something they can't describe.
4. **No third-party footprint.** Nothing outside the site mentions the product, so it never makes a "best tools" answer.

## 5. Fix the site

Work through these and show the user each change:

### A. Make the content exist without JavaScript
- **Next.js / Astro / static**: usually fine already. Confirm with the curl word count.
- **Vite + React SPA**: prerender the landing page at build time. Add an SSR entry that `renderToString`s the landing component, a second `vite build --ssr`, and a small Node script that injects the HTML into `dist/index.html` inside `#root` (`createRoot` replaces it on boot). If signed-in users or deep links see a different screen, hide the static copy before first paint with a tiny inline script that adds a class to `<html>`. Re-run the curl word count: it should go from tens of words to hundreds.

```json
"build": "vite build && vite build --ssr src/prerender-entry.tsx --outDir dist-ssr && node scripts/prerender.mjs"
```

If the Vite config uses `manualChunks`, skip it for the SSR build (`defineConfig(({ isSsrBuild }) => ...)`) or the SSR build fails.

### B. Answer the buyer questions on the page
- A **visible FAQ** built from the baseline prompts: pricing, free limits, "does it work for X", "how is it different from [competitor]". Keep the questions in one data file and render the FAQ section **and** the `FAQPage` JSON-LD from it, so they can't drift apart.
- Rewrite the hero line and meta description in buyer words: input → output → for whom.

### C. Structured data
- `SoftwareApplication` (or `WebApplication`) with `offers` for **every** plan (free and paid, with real prices), `featureList`, `applicationCategory`, and `creator`.
- `FAQPage` from the same FAQ data.
- `Organization` on the brand homepage, with `sameAs` links to real profiles.
- Combine them in one `@graph`. Escape `<` as `<` when you inject JSON into HTML.

### D. One static page per job
For each "job to be done" prompt that missed, write a short guide page (around 400–700 words) titled with the buyer's phrasing: the manual method first, then where the tool does it faster. Add one honest `/compare` page covering the main alternatives, including where they're the better pick. On a Vercel SPA, serve these as static HTML from `public/` and route them **ahead of** the SPA catch-all rewrite in `vercel.json`. Link them from the landing page and the sitemap.

### E. robots.txt and llms.txt
- Allow the search and assistant crawlers explicitly: `GPTBot`, `OAI-SearchBot`, `ChatGPT-User`, `ClaudeBot`, `Claude-SearchBot`, `Claude-User`, `PerplexityBot`, `Perplexity-User`, `Google-Extended`, `Applebot-Extended`, `Amazonbot`. The `*-User` agents fetch pages live when someone asks an assistant about you. Blocking them hides you from that answer.
- `llms.txt`: what it is, who it's for, **pricing**, how it differs from the main alternatives, and links to every tool and guide page. Keep it under 500 words.

Then `/ship`, and re-run the curl checks against production.

## 6. Earn third-party mentions

Write `docs/aeo/listings.md`, paste-ready, from the same facts:

- Name, URL, 60-character tagline, 160-character blurb, a long description (how it works in numbered steps, plus the one honest difference)
- Categories/tags, "alternative to" list, pricing model
- **Where to submit, in priority order.** Start with any page that already surfaced in the baseline (often a Chrome Web Store, App Store or marketplace listing) and update it first. Then AlternativeTo (add it and "suggest as alternative" on each competitor), There's An AI For That, Product Hunt, free AI directories, and roundup posts that already rank for the prompts (pitch the author or leave a useful comment)
- **Reddit and forums**: list the threads that answer the prompts. Answer the question first, give the manual method, mention the tool once as "I built a thing for this", and never post where it isn't the answer. Agents often can't reach reddit.com, so the user may need to pull those threads themselves.

The user does the submitting and posting. Hand them the copy; don't create accounts or post for them.

## 7. Retest

Retest about **4 weeks** after the fixes are indexed. Ask the user to check Google Search Console and Bing Webmaster Tools first: a page that isn't indexed can't be cited. Then:

- Run the **exact same prompts** and save `docs/aeo/retest-YYYY-MM-DD.md` with the old and new score side by side
- Track Google rankings and AI answers **separately**. They move at different speeds, and a page can rank and still not get cited
- Note which page earned each new mention, and put the next round's effort there

## Report

```
## AEO: [Product]

Baseline: present in 1/20 prompts (docs/aeo/baseline-2026-11-24.md)
"Nobody does this" gaps: 4 (jobs the product already does)

### Shipped
✓ Landing prerendered: 18 → 778 words without JS
✓ FAQ (8 Qs) visible + FAQPage JSON-LD from one source
✓ SoftwareApplication schema with Free + Pro offers
✓ 3 guide pages + /compare, in sitemap
✓ robots.txt: 11 AI crawlers allowed · llms.txt with pricing and "vs X"

### Yours to do
☐ Submit to Search Console + Bing, request indexing for the new pages
☐ Update the store listing (copy in docs/aeo/listings.md)
☐ AlternativeTo, TAAFT, Product Hunt

### Retest
Same 20 prompts on ~[date + 4 weeks]
```

## Rules
- Measure before you fix. No baseline means no way to tell whether any of it worked.
- Every claim in the FAQ, schema, llms.txt and listings must match the live site. Wrong prices get repeated by assistants for months.
- Write the way a buyer asks, not the way the brand talks.
- Comparison pages stay honest. Say where a competitor is the better choice, because a one-sided page doesn't get cited.
- Don't stuff keywords, make doorway pages, or post fake reviews. Assistants and search engines both discount them, and they put the brand at risk.
- Visible FAQ and FAQPage JSON-LD always come from the same data.
