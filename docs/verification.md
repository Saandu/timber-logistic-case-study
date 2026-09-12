# Verification

What is asserted automatically, as of 2026-09-12, and what is not.

## On every push, in GitHub Actions

| Step | What it proves |
| --- | --- |
| Schema rebuild | The full migration baseline applies cleanly to a fresh Postgres 16, with Supabase-only extensions stubbed by a harness (`auth.uid()`, storage, vault, cron, net). |
| SQL assertions | Writable paths behave as the procedures intend: role checks, state transitions and rejections are exercised against the rebuilt schema, not mocked. |
| Build | `tsc -b` then `vite build`, then a prerender pass that writes static HTML for public routes. |
| Unit tests | **88 tests across 7 files** — pricing, estimation, validation, wood-volume arithmetic, theme and utilities — under Vitest. |
| Lint | ESLint with `--max-warnings=0`. |
| Bundle guard | Decodes every JWT found in the production bundle and fails on a service role; confirms the bundle targets the expected project. |
| Dependency audit | `npm audit` on production dependencies at high severity. |
| Deploy | Only on `main`, only the bundle that passed the steps above. |
| Post-deploy check | Fetches the live site and confirms it serves the build that was just uploaded, and that the sitemap did not regress. |

## Not automated

- **Browser flows.** Registration, posting, bidding, acceptance and payment are verified by hand. The repository keeps a pre-launch checklist that is ticked only when an item is verified, not when it is intended.
- **The payment gateway against a real merchant account.** The integration is exercised against the gateway's sandbox and the invoicing provider's test series; production is gated until the merchant account clears.
- **Load.** No performance figures are claimed.

## How the counts were taken

Table count is distinct `create table` statements across the migrations; migration and edge-function counts are directory listings; line and file counts are `wc -l` over `src/**/*.{ts,tsx}`; commit count is `git rev-list --count HEAD`. All on 2026-09-12.
