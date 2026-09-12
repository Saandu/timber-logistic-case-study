# Timber Logistic

**A two-sided marketplace for timber and wood products in Romania, built as a sealed-bid reverse auction.** A buyer posts one request for a cart of products; every supplier who covers that county gets one business hour to submit a sealed offer; the buyer picks a winner and pays the platform a 3% commission. Live in production with registered suppliers.

[Live site](https://timber-logistic.ro) · [Engineering notes](docs/engineering.md) · [Verification](docs/verification.md) · [Case study on my site](https://alexandru-lungu.web.app/projects/timber-logistic)

> Client-authorized public case study, on the condition that the application source stays private. The application interface is in Romanian.

![Timber Logistic home page: a forest aerial with the headline "Pădurea, mai aproape de tine" and the marketplace call to action.](assets/home.webp)

## My contribution

**Alexandru Lungu — Full-Stack Developer, contract, June 2026 – present.**

I am the sole developer: product design, the React interface, the Postgres schema and every migration, the auction logic, the payment and invoicing integrations, the serverless functions, the CI/CD pipeline, the legal and compliance suite, and the visual identity. The client owns the business, the supplier relationships and the brand; I do not claim authorship of their commercial decisions.

## Problem and users

Wood is bought by phone in Romania: a buyer calls suppliers one by one, gets a price that depends on who is asking, and has no way to compare. The platform replaces that with a single request that every covering supplier answers blind. Suppliers cannot see each other's bids, the buyer sees all of them sorted cheapest-first once bidding closes, and contact details unlock only for the winning pair — and only once the commission is confirmed paid.

| Audience | Implemented journey |
| --- | --- |
| Buyer | Build a cart across five product categories, post one request, wait for the bidding window, compare sealed offers, accept one, pay the commission, receive contact details, confirm or dispute delivery, review the supplier. |
| Supplier | Register with company details and delivery counties, be approved by an admin, publish offerings with prices, receive requests that match, submit one sealed whole-cart offer per request within their own working hours, be notified on acceptance and payment. |
| Administrator | Approve suppliers, moderate offerings, handle EU DSA Article 16 notices with an immutable audit trail, resolve disputes and trigger refunds. |

## A two-minute look

1. Open the [marketplace](https://timber-logistic.ro/marketplace) and filter by category and county. Prices shown are indicative, derived from live supplier offerings.
2. Read [how it works](https://timber-logistic.ro/cum-functioneaza) for the auction rules as a buyer sees them.
3. The request, bidding and payment flows need an account and, for suppliers, admin approval. No shared credentials are published; the [engineering notes](docs/engineering.md) describe those flows from the implementation.

## Engineering

**React 19, TypeScript, Vite, React Router 7, TanStack Query, React Hook Form + Zod, Tailwind CSS; Supabase (Postgres, Auth, Storage, Edge Functions); Netopia card payments; Oblio e-Factura invoicing; Cloudflare Turnstile and DNS; Firebase Hosting; GitHub Actions.**

The decision the rest of the system hangs off: **the auction and the money are database features, not application features.** Bidding windows are computed by Postgres triggers from each supplier's own working hours and snapshotted the moment a request is posted, so a supplier cannot extend their own window by editing their hours afterwards. Requests that never reach a decision are expired by `pg_cron`. Commission is a generated column with exact-decimal VAT arithmetic; money never passes through a JavaScript float. Row-level security is on every table, every state change goes through `SECURITY DEFINER` procedures, and there are no generic `UPDATE` policies — a client bug can fail to show a buyer their own data, but cannot show them someone else's.

The [engineering notes](docs/engineering.md) cover four tradeoffs in detail: the auction-in-Postgres design and what it costs; reverse-engineering the payment gateway's v2 contract and gating the path; delivery as a claim rather than a verdict; and a CI step that decodes every JWT in the production bundle to catch a leaked service key.

## Quality, outcomes and status

As of 2026-09-12 the private repository holds **20 tables, 45 migrations, 5 edge functions, ~20,000 lines of TypeScript across 135 files and 158 commits.** The CI gate runs typecheck, lint, tests, build, a bundle secret scan and a post-deploy check that the live site serves the new build; pushing `main` deploys. See [verification](docs/verification.md) for what is asserted and how.

**Stated plainly:** the marketplace is live with three approved suppliers and ten offerings, and the payment path is integrated end to end — start call, IPN callback, invoice issuance into e-Factura with a test invoice proving the VAT split to the cent. It is held behind a kill switch until the merchant account is approved. Until then no deal can complete, because contact details unlock only on a confirmed payment. The gate is deliberate: an ungated buyer would otherwise type a real card into the gateway's sandbox.

No transaction volume, revenue or conversion figures are claimed.

## Source and rights

**The application source remains private because this is client work.** This repository contains only case-study writing and a screenshot of the public home page; it contains no application source, credentials, customer data or supplier data.

© 2026 Alexandru Lungu. All rights reserved for case-study material owned by Alexandru Lungu. Client software, branding and content remain the property of their respective rights holders. See [LICENSE](LICENSE).

**Professional profile:** [alexandru-lungu.web.app](https://alexandru-lungu.web.app) · [GitHub](https://github.com/Saandu)
