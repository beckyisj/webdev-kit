# Changelog

## 1.1.0 — 2026-09-06

Refresh for current Claude Code. No new skills; every existing one was audited against the September 2026 harness and models.

- **responsive**: rewritten off the retired Claude Preview MCP onto the Playwright MCP plugin, with a Chrome-extension path and a `npx playwright screenshot` fallback.
- **ship**: no longer hardcodes a model name in the commit trailer (uses the one the harness supplies); project notes go to Claude auto-memory, `CLAUDE.md`, or `SHIPLOG.md` instead of an author-specific path.
- **seo**: robots.txt allowlists the current Anthropic crawlers (`ClaudeBot`, `Claude-SearchBot`) instead of the retired `Claude-Web`; dropped the dead Twitter Card Validator; sitemap dates are placeholders, not April 2026.
- **All skills**: frontmatter uses the documented `user-invocable` key (the `user_invocable` spelling was silently ignored); removed author-specific example paths and brand references; dropped the "Want me to fix any of these?" closers and hard caps on findings, which newer models handle better without.
- **prelaunch / responsive**: reports lead with launch-blockers and fold duplicate findings, rather than truncating at 5 items.

## 1.0.0 — 2026-04-21

Initial release: 8 skills (`/landing`, `/favicon`, `/og-image`, `/seo`, `/responsive`, `/prelaunch`, `/ship`, `/chrome-ext`).
