---
name: domain
description: Point a domain (or subdomain) at a deployed app — reads the current DNS with dig, identifies the registrar, produces the exact records for Vercel (or another host), gives registrar-specific instructions for Cloudflare, Namecheap, GoDaddy, and Vercel DNS, then verifies propagation and SSL. Also advises on buying a domain when the user hasn't yet.
user-invocable: true
allowed-tools: Read, Bash, Glob, Grep
argument-hint: [domain] [target: vercel|cname-host]
---

# Domain — buy it, point it, verify it

**Argument**: `$ARGUMENTS` is the domain and optionally where it should point. Default target is Vercel. Ask for the domain if missing.

## If they haven't bought one yet

Three registrars worth using, in order:
1. **Cloudflare Registrar** — at-cost pricing, no renewal markup, clean DNS panel, free WHOIS privacy. Requires Cloudflare DNS, which is fine.
2. **Namecheap** — cheap, honest renewals, registrar-agnostic.
3. **Vercel** — buy inside the dashboard, zero DNS work. A few dollars more per year.

Avoid GoDaddy for new purchases: expensive renewals, upsells, and WHOIS privacy costs extra. Buy the `.com` if it's available; every other TLD is a compromise. Turn on auto-renew. Expect $10-15/year.

Once bought, come back to this skill.

## Process

### 1. Read current DNS

```bash
dig +short NS <domain>
dig +short A <domain>
dig +short CNAME www.<domain>
dig +short MX <domain>
dig +short TXT <domain>
```

The NS answer identifies the DNS host (`*.ns.cloudflare.com`, `dns1.registrar-servers.com` is Namecheap, `*.domaincontrol.com` is GoDaddy, `ns1.vercel-dns.com` is Vercel). Existing A/CNAME on `@` and `www` are parking records that must be **deleted**, not added to. MX and TXT are email; leave them alone.

### 2. Produce the records

**Apex + www on Vercel** (the default):

| Type | Name | Value |
|---|---|---|
| A | `@` | `76.76.21.21` |
| CNAME | `www` | the per-project value Vercel shows (looks like `abc123.vercel-dns-017.com`) |

**A subdomain only** (`app.domain.com`, `tool.domain.com`):

| Type | Name | Value |
|---|---|---|
| CNAME | `app` | the per-project value Vercel shows |

The CNAME target is unique per project now; the old shared `cname.vercel-dns.com` still resolves but the dashboard and `vercel domains add` print the exact value to use, so read it from there. For a non-Vercel host, use the CNAME target their dashboard gives.

Add the domain on the Vercel side too, or the DNS will resolve to a Vercel 404:
```bash
vercel domains add <domain>
vercel domains add www.<domain>   # Vercel redirects one to the other
```

### 3. Registrar-specific steps (the human part)

Give only the block for the provider found in step 1.

**Cloudflare**: Websites → domain → DNS → Records. Add the records. Set **Proxy status to DNS only** (grey cloud) on the A and CNAME for Vercel. Orange-cloud proxying breaks Vercel's SSL issuance and shows a Cloudflare error page.

**Namecheap**: Domain List → Manage → **Advanced DNS**. Delete the default `CNAME www → parkingpage.namecheap.com` and the `URL Redirect @` record. Add the new records. Saves instantly; propagates in minutes.

**GoDaddy**: My Products → domain → DNS → **Manage DNS**. Delete the existing A on `@` (points at GoDaddy parking) and CNAME on `www`. Add the new ones. Allow up to 30 minutes.

**Vercel DNS** (nameservers already at Vercel): nothing to do; `vercel domains add` handled it.

### 4. Verify propagation and SSL

```bash
dig +short A <domain>                 # expect 76.76.21.21
dig +short CNAME www.<domain>         # expect the vercel-dns value from step 2
dig +short A <domain> @1.1.1.1        # a second resolver, to rule out local cache
curl -sI https://<domain> | head -1   # expect HTTP/2 200 or a 307 to www
vercel domains verify <domain>        # compares live DNS with what Vercel expects
```

SSL is automatic once DNS is correct; give it up to 15 minutes. If `curl` shows a certificate error after that, the Cloudflare proxy is almost certainly still on.

### 5. Report

```
Domain
- mytool.app — DNS at Cloudflare
- Records: A @ 76.76.21.21 · CNAME www → <project>.vercel-dns-017.com (DNS only)
- Parking records: removed
- Propagation: resolved on 1.1.1.1 and 8.8.8.8
- SSL: issued · https://mytool.app → 200
- Email untouched: MX/SPF left as found
- Next: /seo with the canonical domain · /og-image at https://mytool.app/og-image.png
```

## Gotchas

- **Never add a second A record on `@`.** Replace the parking one. Two A records means half your visitors hit the parking page.
- **Email is separate from hosting.** Pointing the A record at Vercel does nothing to `hello@domain`; that's `/improvmx` (receive) and `/resend` (send).
- **Propagation is uneven.** `dnschecker.org` shows per-region answers when your `dig` says one thing and the dashboard another.
- **WHOIS privacy** is free at Cloudflare and Namecheap. Make sure it's on.
