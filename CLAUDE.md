# BeeconPay — Project Overview

BeeconPay is a subscription-based, card-controlled access platform for sports viewing centres in Nigeria. Fans subscribe to a specific centre for a period (daily/weekly/monthly) and gain entry by tapping a Beecon-issued NFC/RFID card at the door. Owners run centres, set their own prices, and earn payouts via an in-app wallet; Beecon takes a commission on every subscription.

> Repo folder is still called `becon/` for historical reasons. The product name is **BeeconPay** — never write "Bison" into new code or docs.

## Roles

- **Fan** — discovers centres, subscribes, tops up wallet, taps card for entry
- **Owner** — manages centre, sets pricing, registers walk-in (cash) users, manages wallet/payouts, views entries
- **Reader** — standby NFC tap app running at the centre door, validates entry
- **Admin** — platform oversight, KYC review, fee config, analytics

## Project Structure

```
becon/
├── backend/              Express 5 + TypeScript + Prisma + PostgreSQL (production)
├── admin_dashboard/      React + Vite + Tailwind admin panel (deployed: admin.beeconpay.com)
├── mobile/               Flutter app — fan, owner, reader roles
├── owner dashboard web/  Web dashboard for owners (not started)
├── landing page/         Marketing site (not started)
├── ui code mobile/       Exported design system component code
└── ui inspiration/       Reference images and mockups
```

Each of `backend/`, `admin_dashboard/`, `mobile/` is its own git repo under `beecon-technologies/` on GitHub.

## Tech Decisions

- **Backend:** Express 5 + TypeScript + **Prisma** (PostgreSQL). Zod for validation.
- **Auth:** JWT (access 15min / refresh 30d) + phone/email + OTP
- **Payments:** Paystack (top-ups, withdrawals)
- **Email:** Resend (KYC notifications, etc.)
- **Storage:** Cloudflare R2 (KYC documents) — private bucket, served via backend `/admin/document-proxy`
- **Mobile:** Flutter + Riverpod + GoRouter
- **Admin:** React + Vite (preview server in Docker), Tailwind, custom UI kit
- **Design:** Dark theme, **blue** accent (`#2563EB` / `#60A5FA`) — NOT emerald/green
- **Deployment:** Dokploy (Docker Compose + Traefik), dokploy-network external

## Key Docs

- `backend/docs/backend architecture.md` — API reference + DB schema
- `backend/docs/prd.md` — product requirements
- This file's **Core Business Rules** section below — business invariants that must not be broken

---

## Core Business Rules (READ BEFORE CHANGING PAYMENT, SUBSCRIPTION, OR ENTRY LOGIC)

These are invariants from the product spec. Never weaken or contradict them without explicit user approval.

### Subscription model
1. **Time-based only.** Plans are `daily`, `weekly`, `monthly`. There is no pay-per-match, per-event, or per-ticket charging.
2. **Tied to one centre.** Every subscription has a `centre_id`. A fan can hold subscriptions at multiple centres simultaneously, but each subscription is scoped to one centre.
3. **Decentralized pricing.** Each centre owner sets their own daily/weekly/monthly prices. The platform does not fix prices. `@@unique([centre_id, type])` enforces one plan of each type per centre.

### Payment flows
4. **App subscription (fan-initiated):** fan wallet is debited the full plan price → Beecon commission is withheld → `netPayout = price − commission` is credited to the owner wallet. Both debit and credit must create Transaction rows.
5. **Cash subscription (owner-initiated for walk-in users):** fan pays cash at the centre. Beecon does **not** deduct the full amount from the owner wallet — only the commission percentage. Commission is charged against the owner wallet balance (so the wallet must have enough to cover it before activation).
6. **Every subscription — app or cash — must create a Transaction record tied to the centre.** Transactions are the source of truth for revenue tracking.

### Wallets
7. **Two wallet surfaces share one schema:** fan wallets (top-up via Paystack, spend on subscriptions) and owner wallets (receive net payouts, pay commissions on cash subs, withdraw to bank).
8. **Withdrawals go through Paystack bank transfer.**

### Cards vs subscriptions (CRITICAL)
9. **Card = identity. Subscription = access permission.**
   - A card belongs to a **user**, not to a centre. One physical card works at every centre.
   - A subscription belongs to a `(user, centre)` pair.
   - Never tie a card to a single centre. Never grant entry without a card.
10. **Card issuance:** fans request cards via the app and pick them up at a chosen centre; owners issue cards directly to walk-in users.

### Entry / access control
11. **No card → no access.** The app alone cannot grant entry. A QR-only or app-only entry flow violates the spec.
12. **Entry verification logic (on card tap):**
    - Reader reads card → identifies user
    - Check: does the user have an active, non-expired subscription **at this centre**?
    - Yes → log entry, grant access. No → deny.
13. **Reader devices must work offline.** The backend periodically syncs an active-subscriber list to each reader; the reader validates taps against its local cache and queues entry logs for later upload.

### Auto-renewal
14. Fans can toggle auto-renew on any subscription. On expiry, a scheduled job charges the fan's wallet (if sufficient) and creates a new subscription. If the wallet is empty, auto-renew silently fails and the subscription lapses.

### Forbidden (will never exist)
- Pay-per-match / per-event charging
- Entry without a card
- A card bound to a single centre
- Full wallet deduction on cash subscriptions (only commission comes out of the owner wallet)
- App-only entry with no physical tap

---

## Current Implementation Gaps (as of 2026-04-11)

These are known deltas between the spec above and the codebase. Do not assume they're fixed — verify before relying on them.

| # | Gap | Status | Notes |
|---|---|---|---|
| 1 | `POST /entries/verify` endpoint | ❌ missing | Reader screens exist as UI only; no backend route verifies an NFC tap against a centre's subscribers |
| 2 | `Card` model as a first-class entity | ❌ missing | `nfc_token` is stored on `Subscription`, conflating card identity with access permission. A separate `Card` model scoped to `user_id` is required |
| 3 | Card issuance flow (fan request + owner-issued) | ❌ missing | No endpoints or UI to request or issue cards |
| 4 | Auto-renewal | ❌ missing | No `auto_renew` field on `Subscription`, no scheduled job |
| 5 | Walk-in user creation for cash subs | ❌ missing | `cashSubscription()` requires an existing `fan_id` — owners can't onboard a non-smartphone user from scratch |
| 6 | Offline sync for reader devices | ❌ missing | No `/centres/:id/subscribers/sync` endpoint, no local cache on reader, no entry-log upload queue |
| 7 | `Transaction.centre_id` FK | ⚠️ partial | Transactions are wallet-centric; querying "all transactions for centre X" requires joining through subscriptions. Add a direct `centre_id` (and optional `subscription_id`) FK on `Transaction` |
| 8 | Reviews/ratings | ❌ missing (deferred) | Spec says "build to support reviews in the future" — no model yet, not a current priority |

## Conventions

- API responses follow `{ success, message, data }` envelope
- Errors use custom classes from `backend/src/core/utils/errors.ts`
- RBAC via `authorize(...roles)` middleware
- Backend feature modules live in `backend/src/apps/<module>/{controllers,services,models,routes,validators}`
- Prisma is the single source of truth for DB schema; migrations run automatically on deploy
- Mobile features live in `mobile/lib/features/<feature>/{screens,widgets,controllers,models}`; all API/network calls go through `lib/services/`
- Admin dashboard API calls go through `admin_dashboard/src/api/` wrapping a shared Axios client

---

# Working Style (LLM Behavioral Guidelines)

Behavioral guidelines to reduce common LLM coding mistakes. Merge with the project-specific instructions above as needed.

**Tradeoff:** These guidelines bias toward caution over speed. For trivial tasks, use judgment.

## 1. Think Before Coding

**Don't assume. Don't hide confusion. Surface tradeoffs.**

Before implementing:
- State your assumptions explicitly. If uncertain, ask.
- If multiple interpretations exist, present them - don't pick silently.
- If a simpler approach exists, say so. Push back when warranted.
- If something is unclear, stop. Name what's confusing. Ask.

## 2. Simplicity First

**Minimum code that solves the problem. Nothing speculative.**

- No features beyond what was asked.
- No abstractions for single-use code.
- No "flexibility" or "configurability" that wasn't requested.
- No error handling for impossible scenarios.
- If you write 200 lines and it could be 50, rewrite it.

Ask yourself: "Would a senior engineer say this is overcomplicated?" If yes, simplify.

## 3. Surgical Changes

**Touch only what you must. Clean up only your own mess.**

When editing existing code:
- Don't "improve" adjacent code, comments, or formatting.
- Don't refactor things that aren't broken.
- Match existing style, even if you'd do it differently.
- If you notice unrelated dead code, mention it - don't delete it.

When your changes create orphans:
- Remove imports/variables/functions that YOUR changes made unused.
- Don't remove pre-existing dead code unless asked.

The test: Every changed line should trace directly to the user's request.

## 4. Goal-Driven Execution

**Define success criteria. Loop until verified.**

Transform tasks into verifiable goals:
- "Add validation" → "Write tests for invalid inputs, then make them pass"
- "Fix the bug" → "Write a test that reproduces it, then make it pass"
- "Refactor X" → "Ensure tests pass before and after"

For multi-step tasks, state a brief plan:
```
1. [Step] → verify: [check]
2. [Step] → verify: [check]
3. [Step] → verify: [check]
```

Strong success criteria let you loop independently. Weak criteria ("make it work") require constant clarification.

---

**These guidelines are working if:** fewer unnecessary changes in diffs, fewer rewrites due to overcomplication, and clarifying questions come before implementation rather than after mistakes.
