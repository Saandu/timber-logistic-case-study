# Engineering notes

These notes describe the implementation as of 2026-09-12 without exposing application source, schema names beyond what the public site implies, or operational identifiers. Where a reason is given, it is the reason recorded in the codebase's own documentation at the time the decision was made.

## Architecture and state

A React 19 single-page application, built with Vite and served from Firebase Hosting, talks to Supabase: Postgres for everything stateful, Supabase Auth for email/password sign-in with confirmation and three roles (buyer, supplier, admin), Storage for supplier documents and offering images, and Edge Functions for the five things that cannot run in the browser — starting a payment, receiving the gateway's callback, issuing an invoice, sending lifecycle email, and looking up a company in the national registry. TanStack Query owns server state in the client; React Hook Form with Zod owns forms.

The Romanian legal suite (GDPR privacy policy, terms, cookie policy, ANPC consumer-protection links) and the EU Digital Services Act Article 16 notice-and-action system are first-class features, not pages bolted on at the end.

## 1. The auction lives in Postgres

When a buyer posts a request, every supplier whose delivery counties cover the request's county is matched, and each gets **one hour of their own working time** to bid — computed from that supplier's declared start, end and working days, not from a fixed 08:00–17:00 day. The per-supplier deadline is written into a dedicated table at the moment the request is posted. That snapshot is the point: a supplier cannot lengthen their own window by editing their hours afterwards, because the window they were given is already recorded. The request's overall bidding close is the latest of the snapshots.

All of this is triggers. A `pg_cron` job expires requests that reach no decision. The browser never computes a deadline it could get wrong, and there is no server process whose downtime would freeze an auction.

**Tradeoff.** Logic in triggers is harder to unit-test from the application and invisible to anyone reading only the TypeScript. The mitigation is a CI job that rebuilds the whole schema from the migration baseline in a fresh Postgres and runs SQL assertions against it — so the trigger logic is tested where it lives. A previous version of the rule split the hour across days; it was retired rather than kept as a fallback, so there is exactly one deadline rule in force.

## 2. The money path: reverse-engineered, exact, and gated

Netopia's v2 API documentation was unavailable when the integration was built, so the contract was recovered from the gateway's own published client package. The integration covers the start call from an Edge Function, the IPN callback that is the **only** code path allowed to mark a payment as paid — never the browser returning from the payment page — and Oblio invoice issuance into Romania's e-Factura/ANAF system behind it. A test invoice proved the VAT split (net plus VAT equals the gross commission to the cent) before any real series was configured.

Commission is a generated column in Postgres with exact-decimal arithmetic. Money never passes through a JavaScript number.

**Status, stated plainly.** The whole path sits behind a `PAYMENTS_ENABLED` kill switch, both in the client bundle and as an Edge Function secret, until the merchant account is approved. With the switch off the start function falls back to the gateway's sandbox endpoint, which is precisely why the gate exists: an ungated buyer would type a real card into a test form. Until it opens, the platform runs as a commission-free beta: accepting an offer is what unlocks contact details. Enabling commission changes that one transition, not the rest of the flow.

## 3. Delivery is a claim, not a verdict

A supplier marking an order delivered does not finalise anything. It opens a window in which the buyer confirms, disputes with evidence, or does nothing — in which case delivery auto-confirms after seven days. Only a confirmed delivery finalises the request and unlocks the review. A dispute resolved in the buyer's favour opens a refund automatically.

In production, the delivery claim also has to be backed by a legal transport document. The supplier provides the SUMAL code for the load; the server verifies it against the public Inspectorul Pădurii register and accepts it automatically only when the returned code and issuing CUI match the request and the verified supplier. The buyer's seven-day window starts only after that check passes. An invalid code is rejected; a registry outage or an issuer mismatch fails closed into manual review, where an administrator sees the registry snapshot and records a reason. Reusing one document across orders is refused, and verification calls are rate-limited and audited.

Supplier identity is held to the same standard. A CUI is checksum-validated in the browser, the Edge Function and the database; legal name, registry number and VAT status come from the ANAF public registry and are written server-side, so a supplier cannot overwrite them. One approved account can claim each CUI, and bidding requires a verified, approved supplier who has accepted the current version of the professional commitment.

The reason is trust asymmetry: the platform never sees the goods, so neither party's word can be the final state on its own. The three-outcome window with a timeout is the smallest mechanism that lets honest parties finish without an admin while giving a wronged buyer a path that does not depend on the supplier's cooperation.

## 4. The database is the security boundary, and CI checks the bundle

Row-level security is enabled on every table. Every state transition — posting, bidding, accepting, paying, delivering, disputing — goes through a `SECURITY DEFINER` procedure that checks the caller's role and the row's current state; there are no generic `UPDATE` policies for the client to reach. Contact details are exposed by a procedure that returns them only for the winning pair, only once the request has reached the unlocking state: acceptance during the commission-free beta, confirmed payment once commission is enabled.

The pipeline extends the same posture outward. A CI step decodes every JWT it can find in the built production bundle and fails the build if any of them carries a service role. It has never fired. The step also confirms the bundle is wired to the expected Supabase project, so a misconfigured environment cannot ship pointing at the wrong database.

Registration sits behind Cloudflare Turnstile so the supplier-approval queue is not a bot target, and the admin's approval is what turns a registered supplier into a matchable one.

## What this design does not do

- It does not handle multi-county requests; a request is for one county, which is the unit the supplier coverage model is built on.
- It does not escrow the supplier's net price. The platform takes only its commission; the net price is settled directly at delivery, which is why the delivery-confirmation window matters.
- It is not yet tested end to end in a browser. Unit tests cover pricing, estimation, validation and wood arithmetic; SQL tests cover the schema's writable paths; the flows between them are verified by hand and documented in a pre-launch checklist.
