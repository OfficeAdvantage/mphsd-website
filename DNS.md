# DNS — mphsd.com

The record of what lives in this domain's DNS zone, why each record is there, and what
breaks if it goes away. **One domain carries two unrelated services** — the public website
(GitHub Pages) and the company email (Microsoft 365) — so a careless edit in the registrar
UI can take down mail while the site keeps working, or the reverse. That is the reason this
file exists.

> **Status (2026-09-10):** Website on GitHub Pages, email on Microsoft 365, Search Console
> verified two independent ways. Zone is healthy; every record below was confirmed live by
> lookup on 2026-09-10.

**When you change the zone, add a row to the [change log](#change-log) in the same commit.**
Git supplies the date and the author; the log supplies the reason, which is the part nobody
can reconstruct later.

---

## Where DNS is managed

| | |
|---|---|
| Registrar / DNS host | **GoDaddy** |
| Nameservers | `ns45.domaincontrol.com`, `ns46.domaincontrol.com` |
| Where to edit | GoDaddy → Domains → mphsd.com → **DNS → Manage Zones** |

GoDaddy is authoritative. Nothing in this repo controls DNS — the only repo file that
touches domain routing is [`CNAME`](CNAME) (contents: `mphsd.com`), which tells GitHub
Pages which hostname to serve. Changing that file without changing DNS, or the reverse,
breaks the site.

---

## Current zone

### Website — GitHub Pages

| Name | Type | Value | Purpose |
|---|---|---|---|
| `@` | A | `185.199.108.153` | GitHub Pages apex |
| `@` | A | `185.199.109.153` | GitHub Pages apex |
| `@` | A | `185.199.110.153` | GitHub Pages apex |
| `@` | A | `185.199.111.153` | GitHub Pages apex |
| `www` | CNAME | `crysvarther.github.io` | `www` → the Pages site. Still works after the repo moved to the OfficeAdvantage org (GitHub routes by hostname), but the canonical target is now `officeadvantage.github.io`; change it at a quiet moment |

All four A records are required — GitHub publishes four and expects all four present.
Removing any of them costs you redundancy; removing all of them takes the site down.
There are no AAAA records, so the site is IPv4-only. GitHub Pages does support IPv6 and
adding AAAA records is a safe, optional improvement.

### Email — Microsoft 365

**Do not edit anything in this section while troubleshooting the website.** None of it
affects the site.

| Name | Type | Value | Purpose |
|---|---|---|---|
| `@` | MX | `0 mphsd-com.mail.protection.outlook.com` | All inbound mail |
| `@` | TXT | `v=spf1 ip4:96.3.144.21 include:spf.protection.outlook.com -all` | SPF — who may send as @mphsd.com |
| `autodiscover` | CNAME | `autodiscover.outlook.com` | Outlook client auto-configuration |
| `selector1._domainkey` | CNAME | `selector1-mphsd-com._domainkey.mphsd.a-v1.dkim.mail.microsoft` | DKIM signing key 1 |
| `selector2._domainkey` | CNAME | `selector2-mphsd-com._domainkey.mphsd.a-v1.dkim.mail.microsoft` | DKIM signing key 2 (rotation) |
| `_dmarc` | TXT | `v=DMARC1; p=quarantine; adkim=r; aspf=r; rua=mailto:dmarc_rua@onsecureserver.net` | DMARC policy + reporting |
| `enterpriseregistration` | CNAME | `enterpriseregistration.windows.net` | Entra ID / Azure AD device registration |

Two notes on the mail records:

- **`ip4:96.3.144.21` in SPF** resolves to `96-3-144-21-dynamic.midco.net` — a Midco
  (regional ISP) address. It predates this project and its purpose is not documented;
  presumably an office connection or ISP relay that needed to send as the domain. It is on
  a *dynamic* range, which means it may no longer point where it did when it was added.
  Worth confirming before anyone prunes it — and worth confirming it still needs to be
  there at all.
- **DMARC is `p=quarantine`**, not `p=none`. Mail that fails both SPF and DKIM alignment
  goes to recipients' junk folders rather than being merely reported. Any new service that
  sends mail as @mphsd.com must be added to SPF (or DKIM-signed) or its mail will be
  quarantined.

### Verification

| Name | Type | Value | Purpose |
|---|---|---|---|
| `@` | TXT | `google-site-verification=XT0gl9t4IW82Yt6-FoRnn4JyonACiiEkwBZbVyhbK5Q` | Google Search Console — **domain** property (`sc-domain:mphsd.com`) |
| `_github-pages-challenge-OfficeAdvantage` | TXT | `80320b90551df2667f62893109fa27` | GitHub Pages — proves the **OfficeAdvantage** org owns the domain, so only its repos can publish a Pages site on it |

**This record must stay.** Google re-checks it periodically. Deleting it during a future
cleanup silently un-verifies the domain property and you lose that Search Console data
until it is restored.

Search Console is verified twice over, deliberately, so no single deletion locks you out:

| Property | Verified by | Notes |
|---|---|---|
| `sc-domain:mphsd.com` (domain) | the TXT record above | Covers http/https, www/non-www, all subdomains |
| `https://mphsd.com/` (URL prefix) | [`googlef47aeb1d901bc7dc.html`](googlef47aeb1d901bc7dc.html) in this repo | File-based, independent of DNS |

Both properties are live and both hold history. The URL-prefix property's verification file
is served by the site, so deleting that file from the repo would un-verify it — the same
hazard as the TXT record, in a different place.

---

## Rules

1. **Never edit the SPF record to add a verification token.** Verification tokens go in
   their own separate TXT record. Multiple TXT records at the apex are normal and fine;
   two *SPF* records is a misconfiguration that breaks mail authentication for the domain.
2. **Never touch MX, DKIM, DMARC, or autodiscover for a website change.** They are
   unrelated to the site and nothing about the site requires them to move.
3. **Add, don't replace.** Registrar UIs make "edit the existing record" the path of least
   resistance. For anything new, create a new record.
4. **Verify after every change** with the commands below, and confirm the records you
   did *not* intend to change are still intact.
5. **Log the reason here** in the same commit as the change.

## Verifying the zone

Run from any machine — these query Google's public resolver directly, bypassing local cache:

```bash
nslookup -type=TXT mphsd.com 8.8.8.8
```

```bash
nslookup -type=MX mphsd.com 8.8.8.8
```

```bash
nslookup -type=A mphsd.com 8.8.8.8
```

Expected: two TXT records (SPF + Google verification), one MX pointing at
`mphsd-com.mail.protection.outlook.com`, and four A records in `185.199.108–111.153`.

A propagation lag of minutes to an hour after a GoDaddy edit is normal; TTLs on this zone
are the GoDaddy default of one hour.

---

## Change log

Newest first. Record what changed **and why** — the "what" is recoverable from history, the
"why" is not.

### 2026-11-01 — Added GitHub Pages domain verification for the OfficeAdvantage org

**Added:** `_github-pages-challenge-OfficeAdvantage TXT 80320b90551df2667f62893109fa27`
**Changed:** nothing. **Removed:** nothing in DNS. (Removed on GitHub, not in DNS: the
`mphsd.com` verified-domain entry on the crysvarther *personal* account, whose old
`_github-pages-challenge-crysvarther` TXT record can be deleted from the zone whenever
convenient — it no longer protects anything.)

Why: the site repo moved from the crysvarther account to the OfficeAdvantage organization in
October. A domain verified under one GitHub account cannot be used by another account's Pages
site, so when the next rebuild ran (two docs merges on 2026-11-01) GitHub dropped `mphsd.com`
from the site and **mphsd.com returned 404 for about seven hours**; `officeadvantage.github.io/mphsd-website`
kept serving. Recovery: removed the personal-account verification, saved `mphsd.com` as the
custom domain on `OfficeAdvantage/mphsd-website` → Settings → Pages (as the org owner), then
added this TXT record and verified the domain under the organization the same evening, which
restores the takeover protection.

Lesson: after any repository transfer, check Settings → Pages → Custom domain the same day,
before merging anything.

SPF, MX and the Google verification TXT were confirmed unchanged after the edit.

### 2026-09-10 — Added Google Search Console domain-property verification

**Added:** `@ TXT google-site-verification=XT0gl9t4IW82Yt6-FoRnn4JyonACiiEkwBZbVyhbK5Q`
**Changed:** nothing. **Removed:** nothing.

Search Console had a `mphsd.com` domain property sitting unverified alongside the working
URL-prefix property. A domain property consolidates http/https, www/non-www and every
subdomain into one view, so it is the better long-term home for search data. Verification
requires a DNS TXT record; this is it. Verified same day, and the property backfilled its
history rather than starting empty.

Google offered to write this record itself via an OAuth flow against the GoDaddy account.
Declined — that grants Google write access to the zone that carries the company's email,
which is a poor trade for saving one paste.

SPF and MX were confirmed byte-identical before and after.

### 2026-07-21 — Website cutover to GitHub Pages

**Added/changed:** apex `A` → the four `185.199.108–111.153` GitHub Pages addresses;
`www` `CNAME` → `crysvarther.github.io`. **Mail records deliberately untouched.**

Moved the public site onto GitHub Pages, served from the `main` branch of
`crysvarther/mphsd-website`, with [`CNAME`](CNAME) in the repo binding it to the apex
hostname. Microsoft 365 email was already running on this domain and had to keep running
throughout — hence website records only.

*(Date from project notes; the pre-cutover A record values were not recorded.)*

### Before 2026-07-21 — Microsoft 365 email (undated)

The MX, SPF, DKIM (`selector1`/`selector2`), DMARC, `autodiscover` and
`enterpriseregistration` records predate this project. Who added them and when is not
recorded. They are listed above as found, and they work — treat them as load-bearing and
leave them alone absent a specific reason.
