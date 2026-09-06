---
name: improvmx
description: Set up free inbound email forwarding on a custom domain with ImprovMX — hello@yourdomain.com lands in your existing Gmail. Generates the exact MX and SPF records, merges SPF with Resend if both are present, verifies DNS with dig, and sends a test. Use when a project needs a real-looking email address without paying for Google Workspace.
user-invocable: true
allowed-tools: Read, Bash, Glob, Grep
argument-hint: [domain] [forward-to-address]
---

# ImprovMX — inbound email forwarding

Give a domain a working `hello@` address in ten minutes, for free, forwarding to an inbox the user already has.

**Argument**: `$ARGUMENTS` is the domain and optionally the destination address (e.g. `mytool.app me@gmail.com`). Ask for whichever is missing.

ImprovMX only **receives**. Outbound programmatic email (forms, magic links, receipts) is `/resend`. They pair: Resend sends the notification, the reply comes back through ImprovMX.

## Process

### 1. Read the current DNS

```bash
dig +short MX <domain>
dig +short TXT <domain> | grep -i spf
dig +short NS <domain>
```

From this you learn three things: whether MX records already exist (Google Workspace, Zoho, an old host), whether an SPF record exists that you must merge into rather than duplicate, and which provider hosts the DNS (the NS answer tells you Cloudflare vs Namecheap vs GoDaddy vs Vercel), which decides the instructions in step 3.

If MX records already point at a real mail provider, stop and say so. Adding ImprovMX on top would hijack the existing mailboxes.

### 2. Produce the records

Give the user this table, filled in:

| Type | Name | Value | Priority |
|---|---|---|---|
| MX | `@` | `mx1.improvmx.com` | 10 |
| MX | `@` | `mx2.improvmx.com` | 20 |
| TXT | `@` | `v=spf1 include:spf.improvmx.com ~all` | |

**If an SPF record already exists**, replace the TXT row with the merged version. Common merges:
- With Resend: `v=spf1 include:spf.improvmx.com include:_spf.resend.com ~all`
- With Google Workspace sending: `v=spf1 include:spf.improvmx.com include:_spf.google.com ~all`

Never leave two `v=spf1` TXT records on the same name. Receivers treat that as a permanent SPF failure and both senders go to spam.

### 3. Dashboard and DNS steps (the human part)

1. Sign up at improvmx.com. No card needed.
2. **Add domain**, enter the domain.
3. Add the records from step 2 in the DNS panel for the provider you detected:
   - **Cloudflare**: Websites → domain → DNS → Records. Set MX and TXT to **DNS only** (grey cloud). Proxied MX silently breaks delivery.
   - **Namecheap**: Domain List → Manage → Advanced DNS. Delete the default parking CNAME/URL-redirect on `@` if it's still there.
   - **GoDaddy**: My Products → DNS → Manage DNS. Allow up to 30 minutes.
   - **Vercel DNS**: `vercel dns add <domain> @ MX mx1.improvmx.com 10` etc. from the CLI.
4. Create aliases: `hello@` and `*@` (catch-all) → the destination address. Add `press@`, `support@` if the project wants them.
5. Wait five minutes, click **Verify**.

### 4. Verify

```bash
dig +short MX <domain>          # expect both improvmx hosts
dig +short TXT <domain>         # expect exactly one v=spf1 line
```

Then have the user send a test from a different account to `hello@<domain>`. First forwarded message often lands in Gmail spam; marking it "not spam" once fixes it going forward.

### 5. Replying as hello@ (optional)

Forwarding is one-way. To reply from `hello@<domain>` inside Gmail: ImprovMX dashboard → the alias → **SMTP credentials**, then Gmail → Settings → Accounts → **Send mail as**. Free plan allows 25 SMTP sends a day, which is plenty for personal replies. Do not route app email through this; that's what `/resend` is for.

### 6. Report

```
ImprovMX
- Domain: mytool.app (DNS at Cloudflare)
- MX: mx1/mx2.improvmx.com — live
- SPF: merged with Resend — single record confirmed
- Aliases: hello@, *@ → me@gmail.com
- Test: waiting on user send / received
```

## Gotchas

- **Free tier** is one domain, unlimited aliases, 25 SMTP sends/day. Premium ($9/mo) adds more domains.
- **Catch-all collects spam** eventually. Fine to start; switch to named aliases if it gets noisy.
- **DNS propagation** varies by registrar. `dnschecker.org` shows what different regions see if `dig` and the dashboard disagree.
