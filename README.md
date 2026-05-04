# Case Study: Finance Tracker – P&L Dashboard

**Client commission** · Full-stack web app · Next.js · Flask · PostgreSQL

---

A digital goods reseller hired me to build a private internal dashboard to track their profit and loss. They were running sales through multiple platforms simultaneously — storefront, inventory source, and two separate payment processors — and none of these talked to each other. Every month they were manually cross-referencing spreadsheets to figure out if they actually made money.

The deliverable was a live dashboard that pulls from all four sources automatically, reconciles the data, and shows accurate P&L with charts, filters, and per-product breakdowns.

---

## The Interesting Problems

**Multi-platform reconciliation**

The fundamental challenge was that revenue, costs, and fees all lived in different systems. The storefront knew what sold. The inventory platform knew what was paid for each account. Stripe knew the exact processor fee. A crypto payment bridge knew the network fees. None of them knew about the others.

I built a sync pipeline that pulls each source in order, then stitches the records together — matching sales to the specific inventory items that were delivered, and attaching exact fee data from each processor. The alternative was trusting platform-estimated fees, which are often slightly wrong and add up over thousands of transactions.

**Stripe fee matching**

Stripe gives you an estimated fee at transaction time, but the real number lives in a separate API endpoint. Matching those back to the right sale required either a direct transaction ID (clean, exact) or a fallback using amount + timestamp within a 30-minute window. I stored which method was used on each record so the client could see confidence levels in their data and audit anything that looked off.

**Keeping syncs fast**

Full re-syncing thousands of historical records on every run would be too slow. Each integration needed an early-stop condition — the storefront returns invoices newest-first, so the sync stops after hitting two consecutive pages of already-processed records. This kept incremental syncs fast regardless of total record count.

**Timezone-aware reporting**

The database stores everything in UTC, but the client wants "today's revenue" to mean their local day — not UTC midnight. Every chart request sends the browser's timezone offset, and the backend uses it to bucket data correctly. Small detail, but without it the daily charts would be wrong for anyone not in UTC.

---

## Outcome

The client got a self-hosted dashboard deployed on a single server — password-gated, no third-party analytics or data sharing. Sync runs automatically every few minutes and the dashboard reflects near-real-time P&L across all platforms without any manual work on their end.
