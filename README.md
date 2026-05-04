# Case Study: Finance Tracker – P&L Dashboard for Digital Goods Sales

## Overview

Built as a client commission for a digital goods reseller. The client needed a private internal dashboard to track P&L across their operation - existing tools either didn't support their platform mix or exposed more data to third parties than they were comfortable with.

Revenue comes in through SellAuth (the storefront), inventory is sourced from LZT Market, and payments split across Stripe and NowPayments. The core problem: none of these platforms talk to each other, and getting an accurate P&L means reconciling all of them.

The app pulls data from each source on a schedule, matches transactions across platforms, calculates fees and COGS, and presents it through a dashboard with charts, filters, and per-product analytics.

**Stack:** Next.js 16 (App Router) · Flask 3 · SQLAlchemy · PostgreSQL · Docker · Railway

---

## The Core Engineering Problem

The business has a layered cost structure that no single platform knows about:

- **Revenue** lives in SellAuth (invoices, refunds, delivery status)
- **COGS** lives in LZT Market (what was paid for each account)
- **Processor fees** live in Stripe (exact to the cent, but only accessible via balance transactions)
- **Crypto fees** go through a NowPayments bridge running as a separate service

Getting an accurate net profit number means pulling all four and reconciling them. The naive approach - trust each platform's reported numbers - gives you wrong answers because platform-reported fees are often estimates, and COGS requires linking a specific sold account back to its specific purchase record.

---

## Sync Architecture

The sync system runs as an ordered pipeline of eight steps:

```
lzt → sellauth → backfill_tax → sellauth_stripe_ids → stripe → nowpayments → backfill_geo → auto_link
```

Each step is skipped if the relevant credentials aren't configured, which lets the app run in partial-data mode. Steps write to a shared progress file in the temp directory so the frontend can poll and show a live progress bar during a sync run.

**Incremental syncing** was important for keeping sync times reasonable. LZT stops paginating when it hits a page with no new `item_id`s. SellAuth (which returns newest-first) stops after two consecutive pages of zero new IDs, but still checks refund status on known records. This keeps full syncs from re-fetching thousands of already-processed invoices on every run.

Auto-sync runs on a background daemon thread, configurable to a minimum of 5 minutes. It skips itself if a manual sync is already running, coordinated via the same progress file.

---

## Stripe Reconciliation

Stripe's reported fee on a charge isn't always the final number - balance transactions give you the exact processor fee after any adjustments. The sync service fetches charges with expanded balance transactions and matches them to sales records two ways:

1. **By PaymentIntent ID** - preferred, pulled from SellAuth invoice detail in a prior step
2. **By amount + timestamp** - fallback, matches within 30 minutes and minimizes delta

The match method is stored on each sale (`stripe_match_method: pi_id | amount_time`) so you can audit confidence levels and re-run more targeted reconciliation later. Re-syncs skip records already matched by PI ID.

---

## NowPayments Bridge

The crypto payment flow was an interesting constraint. Rather than re-implementing NowPayments IPN handling in this service, the NP integration reads from a separate bridge database - a Neon Postgres instance maintained by a separate bridge app that handles webhooks and stores finished payment records alongside their SellAuth invoice IDs.

This service connects to that database as a read source, pulls confirmed payments, then calls the NowPayments API directly to get the exact outcome amount and fee. It's a two-database read pattern that avoids duplicating IPN logic while still getting accurate fee data.

---

## Inventory Linking

The auto-link step tries to connect each sale to the specific account that was delivered. SellAuth delivery strings have a loose format (`email:recovery:IGN:...`), so the linker parses these and matches against LZT account records indexed by Minecraft IGN and email from raw order data.

When a link is found, COGS gets calculated using the account's `cost_basis_usd` and profit is recomputed. The step uses `_autolink_checked` flags to avoid re-fetching API data for sales that already came back empty - otherwise a large backlog of unlinked sales would hammer the API on every sync.

---

## Timezone Handling

Financial dashboards have a quiet complexity around time: the database stores UTC, but the user wants to see "today's revenue" in their local day boundaries, not UTC midnight boundaries.

Every chart request includes a `tz_offset` from `new Date().getTimezoneOffset()` in the browser. The backend uses this to adjust epoch math when bucketing chart data into daily or monthly series, so the chart bars line up with what the user expects their "day" to be. The filter logic converts date range inputs to naive UTC before querying, which keeps the ORM queries simple.

---

## Deployment

The production deploy is a single container on Railway. The root `Dockerfile` builds the Next.js standalone output, copies it alongside the Flask backend, and runs a `start.sh` that starts gunicorn on `127.0.0.1:5001` in the background, then starts the Next.js server on the exposed port.

The Next.js server proxies `/api/*` to the loopback gunicorn instance via `next.config.ts` rewrites, so only one port is exposed externally. The browser always talks to the Next server; Next talks to Flask on the local network stack. This avoids CORS in production and keeps the Flask API off the public internet.

---

## What I'd Do Differently

**Sync progress via file is fragile.** Using a JSON file in `tempfile.gettempdir()` works fine for a single-instance deploy, but breaks on read-only filesystems or if you ever run multiple workers. A `SyncState` table in Postgres, or even a Redis key, would be more robust and not much more work.

**The `_safe_rollback` function calls itself recursively** instead of calling `db.session.rollback()`. This is a bug - a failed sync that hits that path may leave the session in a bad state. It survived testing because full syncs rarely fail mid-transaction, but it's the kind of thing that causes confusing state corruption under real load.

**Single-password auth is fine for the use case**, but storing the bcrypt hash in an environment variable and having the app mutate `os.environ` to auto-hash a plaintext value on startup is surprising behavior. Worth moving to a dedicated secrets management approach if this ever sees multiple users.
