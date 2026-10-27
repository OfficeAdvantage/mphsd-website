# Analytics & KPIs — mphsd.com

Everything Wix Analytics gives you, built from free Google tools. The tracking code is
already wired into the site generator — it just needs two IDs pasted into `build.py`
(a one-time, ~10-minute job that requires signing in to Google, so it has to be done by
a person with the company Google account).

> **Status (2026-09-01):** GA4 (`G-4LYW71CZQR`) is live, Search Console is verified (via the
> `googlef47aeb1d901bc7dc.html` file, so `GSC_VERIFICATION` stays `""`), and the Google Business
> Profile is verified and wired into the site. Still to do: **step 4 below** (connect the profile
> to GA4) and the Looker Studio dashboard.
>
> **Update (2026-10-27):** the dashboard now exists as a Claude dashboard,
> **Mitchell Plumbing & Heating Analytics** (https://claude.ai/artifact/SyHwT4jrRPevauKBadwd8D),
> which replaces the Looker Studio build below. It runs on sample files until the BigQuery
> exports in **"Automating the Claude dashboard with BigQuery"** (near the end of this file)
> are set up — do that first, since the exports don't backfill. There is also the HDO
> multi-client dashboard (`analytics-dashboard/` beside `site/`), which shows live numbers
> today via Google sign-in.

---

## What's already built in

Once `GA4_ID` is set in `build.py`, every page automatically reports:

| Event | Fires when a visitor… | Why it matters |
|---|---|---|
| `page_view` | views any page | traffic, top pages, where visitors come from |
| `phone_call` | taps/clicks any phone number | **your #1 conversion** — includes which page they were on |
| `generate_lead` | successfully submits the contact form | quote requests, with the request type (estimate/service/account) |
| `sms_click` | taps the "text us photos" number | photo-quote leads |
| `email_click` | clicks the email address | email leads |
| `outbound_click` | clicks to Facebook, Google, Lochinvar, etc. | which off-site profiles get traffic |
| `cta_click` | clicks any button/CTA (Free Estimate, Call Now, card links…) | which buttons earn their placement — the label rides along as `cta_label` |

If `GA4_ID` is left `""`, zero tracking code is emitted — the site stays tracker-free.

**One-time GA4 click to unlock button labels:** GA4 stores the `cta_label` parameter but
won't report on it until it's registered. In GA4: **Admin → Custom definitions → Create
custom dimension** → Dimension name `CTA label`, scope **Event**, event parameter
`cta_label` → Save. (Applies from that day forward. Without this, `cta_click` still
counts total button clicks — you just can't split by which button.)

**AI referral traffic needs no extra setup** — visitors who arrive from ChatGPT,
Perplexity, Gemini, Claude, Copilot, etc. show up automatically with source domains like
`chatgpt.com` / `perplexity.ai` under Referrals. The HDO Analytics dashboard breaks
these out as their own "AI assistants" source line.

## One-time setup (do these in order)

### 1. Google Analytics 4 (~5 min)
1. Go to https://analytics.google.com and sign in with the company Google account.
2. Admin (gear icon) → **Create Property** → name it "Mitchell Plumbing & Heating".
3. Add a **Web** data stream for `https://mphsd.com`.
4. Copy the **Measurement ID** (looks like `G-ABC123XYZ`).
5. Paste it into `GA4_ID` in `build.py`, then:
   ```
   cd site
   python pages.py
   ```
   Commit & push. Data starts appearing within a day.
6. After a phone click and a form submit have happened at least once: in GA4 go to
   **Admin → Events**, and flip the **"Mark as key event"** toggle on `phone_call` and
   `generate_lead`. That makes them show up as conversions everywhere in GA4.

### 2. Google Search Console (~3 min) — do this the day the site goes live
1. Go to https://search.google.com/search-console → **Add property** → **URL prefix**
   → `https://mphsd.com`.
2. Choose the **HTML tag** verification method. It shows a line like
   `<meta name="google-site-verification" content="TOKEN">`.
3. Paste just the TOKEN into `GSC_VERIFICATION` in `build.py`, re-run `python pages.py`,
   commit & push, then click **Verify** back in Search Console.
4. In Search Console: **Sitemaps** → submit `https://mphsd.com/sitemap.xml`.

Search data takes a few weeks to accumulate — which is why this should happen at launch,
not later.

### 3. Google Business Profile (biggest local-SEO lever) — ✅ done
The profile is verified. Its insights show calls, direction requests, and profile views
from Google Maps — for a local trade business this often drives more calls than the
website itself.

Wired into the site as of 2026-09-01:
- `GBP_URL` = `https://g.page/r/CQziyzk1Su77EBM` — the profile page. Feeds the schema.org
  `sameAs` links and the footer Google icon.
- `GBP_REVIEW_URL` = the same URL + `/review` — the write-a-review deep link, used **only**
  by the "Leave a Google Review" button on `reviews.html`. Keep these separate: `sameAs` and
  the footer icon should point at the profile, not drop every visitor into a review dialog.

The profile currently has **zero reviews** — see `marketing/google-business-profile-setup.md`
for the remaining dashboard work (categories, photos, services, review requests).

### 4. Connect the Business Profile to GA4 (~2 min)
Since June 2026 GA4 pulls Business Profile metrics natively — no tags, no separate login.
**GA4 → Admin → Product links → Google Business Profile links → Link.**

- Requires Editor/Administrator on the GA4 property **and** Owner/Manager on the profile,
  signed in as the **same Google account**. Mismatched accounts is the most common reason the
  profile never appears in the picker.
- Adds a Business Profile report collection to the left-hand Reports menu with seven metrics:
  interactions, website clicks, calls, directions, messages, bookings, menus.
- ⏱ **Rolling six-month window, and it does not backfill** — data only accrues from the day
  you link, so linking late permanently costs you that history.
- Limits worth knowing: these metrics are view-only — they can't be used in Explorations,
  comparisons, or filters — and the integration isn't supported on GA4 subproperties.
- If the menu item is missing: check the account mismatch above, and confirm you're on a
  standard property rather than a subproperty.

### 5. The KPI dashboard (~20 min, free) — full build spec below
[Looker Studio](https://lookerstudio.google.com) connects to both GA4 and Search Console
and gives you a single bookmarkable "how's the website doing" page that refreshes itself
every time you open it. Build it once using the paint-by-numbers spec in the next
section; after that it's a live graphical dashboard on demand — the equivalent of the
Wix Analytics home screen, except you own it.

---

## Looker Studio dashboard — build spec

Prerequisites: steps 1 and 2 above are done, and at least a few days of data have
accumulated (charts on an empty property look broken — they aren't).

### Setup (once)
1. Go to https://lookerstudio.google.com → **Blank Report**.
2. It asks to add data. Add **two** sources:
   - **Google Analytics** connector → pick the *Mitchell Plumbing & Heating* GA4 property
   - **Search Console** connector → pick the `https://mphsd.com` property →
     choose **"Site Impression"** table → **"web"** search type
3. Rename the report (top-left): `MPHSD — Website KPIs`.
4. Top menu **Theme and layout** → pick any theme you like (this is purely cosmetic).

Every widget below is added with **Insert →** (chart type), then set the **Data source**,
**Dimension**, and **Metric** in the right-hand panel. Set the report-level default date
range to **Last 28 days** with comparison to **Previous period** (File → Report settings).

### Row 1 — "Did the website make the phone ring?" (4 scorecards, GA4 source)
| # | Widget | Metric | Settings |
|---|---|---|---|
| 1 | Scorecard | **Key events** filtered to `phone_call` | Add a chart filter: Event name = `phone_call`. Label it "Phone calls from site" |
| 2 | Scorecard | **Key events** filtered to `generate_lead` | Chart filter: Event name = `generate_lead`. Label "Quote requests" |
| 3 | Scorecard | **Total users** | Label "Visitors" |
| 4 | Scorecard | **Sessions** | Label "Visits" |

For each: turn on **comparison to previous period** (right panel → Date range compare) so
every number shows a green/red % change arrow. These four tiles are the whole business
case for the website — keep them top-left where the eye lands first.

### Row 2 — trend + where visitors come from (GA4 source)
| Widget | Dimension | Metric | Notes |
|---|---|---|---|
| Time series chart | Date | Sessions; add 2nd metric: Key events | One line = traffic, one = conversions. The gap between them is opportunity |
| Pie chart | Session default channel group | Sessions | Shows Organic Search vs Direct vs Social vs Referral — i.e., "is SEO working?" |

### Row 3 — what they do on the site (GA4 source)
| Widget | Dimension | Metric | Notes |
|---|---|---|---|
| Table | Page path and screen class | Views; Total users | Sort by Views desc, top 10. Which services/towns pull traffic |
| Table | Event name | Event count | Chart filter: Event name in `phone_call, generate_lead, sms_click, email_click, outbound_click`. All conversions side-by-side |

### Row 4 — Google Search performance (Search Console source)
| Widget | Dimension | Metric | Notes |
|---|---|---|---|
| Table | Query | Impressions; Url Clicks; Average Position | Sort by Impressions desc, top 15. What people type into Google — feeds blog topics |
| Time series | Date | Impressions; Url Clicks | The long-term SEO trend line. Expect this to climb for months after launch |

### Finish
- **Share** (top right) → give view access to whoever at the company should see it, or
  turn on link sharing → bookmark the link. Opening it always shows current data.
- Optional: Share → **Schedule delivery** → email a PDF snapshot to the owner monthly.
- The **date picker** widget (Insert → Date range control, drop it top-right) lets anyone
  flip the whole dashboard between "last 7 days / 28 days / this year vs last year" on
  the fly.

> Field names occasionally get renamed by Google. If a metric listed above isn't shown
> verbatim, use the search box in the metric picker — e.g. "key events" was called
> "conversions" before 2024.

### Managing multiple client sites from one dashboard

Looker Studio has a **Data control** widget made exactly for this: **Insert → Data
control**, drop it at the top of the report next to the date picker. It renders as a
dropdown listing every GA4 property your signed-in Google account can access — pick a
different company and the entire dashboard (all GA4-sourced charts) switches to that
site's data instantly.

- To add a new client: create their GA4 property (or get viewer access to an existing
  one via GA4 → Admin → Property access management). It appears in the dropdown
  automatically — no report editing needed.
- The Search Console widgets (Row 4) are the one exception: a data control only swaps
  sources from one connector at a time. Either add a second data control bound to the
  Search Console connector, or keep Row 4 per-site on separate report pages.
- Same trick works for viewers you share with: they see only properties *their* account
  can access.

## The KPIs worth watching monthly

1. **Phone calls + form leads** (GA4 key events) — the only numbers that pay the bills
2. **Calls / directions from Google Business Profile** — the Maps side of the funnel.
   Once step 4 is done these sit in GA4's Business Profile reports alongside the website
   numbers, instead of needing a separate trip to the Business Profile dashboard.
3. **Search impressions, clicks, and average position** (Search Console) — is SEO working
4. **Top queries** (Search Console) — what people actually search; feeds blog topics
5. **Top pages + traffic sources** (GA4) — which services and towns pull visitors

Ignore raw pageviews and "sessions" as goals — they're inputs, not results.

## Automating the Claude dashboard with BigQuery (~30 min once, then hands-off)

There is now a Claude dashboard for this site, **Mitchell Plumbing & Heating Analytics**
(https://claude.ai/artifact/SyHwT4jrRPevauKBadwd8D). It currently runs on sample CSV files.
The hands-off way to feed it is to let Google copy the raw data into **BigQuery** every day
(a built-in, free, no-code export from both GA4 and Search Console), connect BigQuery to
Claude once, and then have Claude rewrite the dashboard's sources as live queries. After
that the dashboard refreshes itself every time it is opened, and nobody exports anything.

> ⏱ **Neither export backfills.** GA4 and Search Console only start copying data from the
> day the link is made. Every week this waits is a week of history the dashboard will
> never have. Do steps A–C as soon as possible, even if step D waits.

Everything below has to be done **signed in to the company Google account** that owns the
GA4 property `G-4LYW71CZQR` and the Search Console property `sc-domain:mphsd.com` (the same
account that set up the HDO analytics dashboard's Cloud project). Google renames menu items
now and then; if a label below isn't there verbatim, use the search box at the top of the
Cloud console or the help pages linked at the end.

### A. Google Cloud project (~10 min)

1. Open https://console.cloud.google.com. In the project picker (top bar) choose the project
   that was created for the HDO analytics dashboard (named **HDO Analytics Dashboard**). If it
   doesn't exist, **New project** → name `MPHSD Analytics` → Create, then select it.
2. **Note the Project ID** (Dashboard → Project info, or the project picker). It looks like
   `hdo-analytics-dashboard-123456`. The *ID*, not the *number* — Search Console asks for the ID.
3. Menu ☰ → **APIs & Services → Library**. Search for and **Enable** each of:
   - **BigQuery API**
   - **BigQuery Storage API**
4. Menu ☰ → **Billing** → **Link a billing account** (create one with a company card if none
   exists). Why: Search Console's export refuses to write to a project without billing, and
   without billing BigQuery's free "sandbox" deletes every table after 60 days. **Expected
   charge: $0.** BigQuery's always-free tier is 10 GB of storage and 1 TB of queries per month;
   a site with a few dozen visits a day uses well under 1% of that. To be safe, Billing →
   **Budgets & alerts** → Create budget → $5/month → email alert.
5. Menu ☰ → **IAM & Admin → IAM** → **Grant access**:
   - New principal: `search-console-data-export@system.gserviceaccount.com`
   - Roles: **BigQuery Job User** and **BigQuery Data Editor** (add both, then Save).
   This is the robot account Search Console uses to write its tables. GA4 adds its own
   (`firebase-measurement@system.gserviceaccount.com`) automatically when you link in step B;
   if GA4 later complains about permissions, give that account **BigQuery User** here.

### B. GA4 → BigQuery daily export (~5 min)

1. Open https://analytics.google.com, property **Mitchell Plumbing & Heating** (`G-4LYW71CZQR`).
2. **Admin** (gear, bottom left) → under *Product links* → **BigQuery links** → **Link**.
   You need Editor or above on the GA4 property and Owner on the Cloud project — the company
   account has both if step A was done with it.
3. **Choose a BigQuery project** → pick the project from step A → Confirm.
4. **Data location:** `United States (us)` (multi-region). Keep this; it can't be changed later
   and Search Console will go in the same place.
5. Next → *Configure data streams and events*: leave all streams included, exclude nothing.
6. **Include advertising identifiers for mobile app streams:** leave off (there is no app).
7. **Frequency:** tick **Daily** only. (Streaming is not needed and can cost money.)
8. Next → **Submit**. The link shows as active immediately; the first table arrives within
   24 hours as `analytics_<property number>.events_YYYYMMDD`.
9. While you are here: **Admin → Property settings → Property details** → copy the numeric
   **Property ID** (not the `G-` measurement ID) into the note at the end of this section.
   The BigQuery dataset is named after it.

### C. Search Console → BigQuery bulk export (~5 min)

1. Open https://search.google.com/search-console, property **mphsd.com** (the domain property).
   You must be an *Owner* of the property, not just a user.
2. Left menu → **Settings** → **Bulk data export** → **Continue** (or *Start*).
3. **Cloud project ID:** paste the Project ID from step A.2.
4. **Dataset name:** leave the default `searchconsole`.
5. **Dataset location:** `US` — the same as GA4 in step B.4.
6. Continue → Search Console checks the permissions from step A.5 → **Set up export**.
   If it reports a permissions error, go back to IAM and confirm both roles are on the
   `search-console-data-export@…` principal, then retry.
7. First tables appear within 48 hours: `searchconsole.searchdata_site_impression`,
   `searchconsole.searchdata_url_impression` and an `ExportLog` table. Daily after that.
   Settings → Bulk data export shows the export's status and last successful day.

### D. Connect BigQuery to Claude and switch the dashboard (~5 min, then Claude's job)

1. In Claude (claude.ai) → **Settings → Connectors** → browse the directory → add
   **Google Cloud BigQuery** → **Connect** → sign in with the same company Google account →
   allow read access. (If the directory shows two BigQuery connectors, the Google Cloud one
   is the right one.)
2. Open the Claude session that holds the dashboard, or any new session in this project,
   make sure the BigQuery connector is enabled for it, and say:
   *"BigQuery is connected. Switch the Mitchell Plumbing & Heating Analytics dashboard from
   the sample files to live queries on the GA4 and Search Console exports."*
3. Claude then writes one SQL query per dashboard source (daily sessions and lead events,
   sessions by source/medium, views by page, Search Console by day, top queries, top pages,
   demand by town), keeps the same column names so the page does not change, removes the
   "Sample" labels, and checks each query. From then on the numbers refresh on every open.
4. In the dashboard itself, **Settings → Queries run as**: choose **You** if people at
   Mitchell Plumbing should see the numbers without having their own BigQuery access, or
   leave **Each viewer** if only HDO staff with the connector will open it.

### What stays manual, and the cheap fixes

| Card | Why it can't come from BigQuery | What to do |
|---|---|---|
| Google Business Profile (calls, directions, website clicks, reviews) | No BigQuery export exists for the profile | Do **step 4 above** (link the profile to GA4) so calls/directions at least appear in GA4's own reports; then either a 5-minute monthly CSV from Business Profile → Performance, or ask Claude to add a Business Profile API connector later |
| AI visibility (does Claude/Gemini/Perplexity recommend the business?) | No API reports what assistants answer | Run the AI Visibility tab in the HDO dashboard quarterly and paste the results to Claude, or let Claude run a monthly scheduled task that asks the five questions with web search (Claude column only) |
| PageSpeed scores | The key-less PageSpeed API has a daily quota of 0 | Re-run at https://pagespeed.web.dev after each deploy and tell Claude the scores, or create a free PageSpeed API key in the Cloud project (APIs & Services → Credentials) and Claude can re-run it on a schedule |

### Record here when done

| Item | Value |
|---|---|
| Cloud project ID | _(step A.2)_ |
| GA4 numeric property ID / dataset | `analytics_` _(step B.9)_ |
| GA4 export linked on | _(date — history starts here)_ |
| Search Console export started on | _(date — history starts here)_ |
| BigQuery connected to Claude on | _(date)_ |
| Dashboard switched to live queries on | _(date)_ |

Google's own guides, in case a screen looks different: [GA4 — Set up BigQuery Export](https://support.google.com/analytics/answer/9823238),
[Search Console — Start a new bulk data export](https://support.google.com/webmasters/answer/12917675),
[Search Console — Manage and monitor bulk data exports](https://support.google.com/webmasters/answer/12919198).

---

## Later, if the business wants more

- **Uptime monitoring** — free at https://uptimerobot.com (alerts if the site goes down)
- **Call tracking** (CallRail, ~$45+/mo) — swap-in tracking phone number that attributes
  calls to ads vs. website vs. truck wrap. Only worth it once there's ad spend.
- **Rank tracking** — a monthly manual check of "plumber mitchell sd" etc. is free;
  paid trackers only matter if competing for many keywords.
