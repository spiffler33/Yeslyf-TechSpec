# 03 — Payments: Razorpay (primary) / Cashfree (alternative)

> Research date: July 2026. All facts below were checked against official sources (razorpay.com docs/pricing, RBI framework coverage, NPCI, Apple/Google developer policy) where possible. Items that could not be verified from public sources are explicitly flagged as **[unverified]**.

---

## Overview & role in Yesly

Yesly (House of Alpha, SEBI RIA reg. INA000018364) needs a payment layer for:

1. **One-time tier purchases** — Essential at ₹2,999 and Growth at ₹4,999, paid once via UPI/cards/netbanking. Modeled as Razorpay **Orders → Payments** with checkout in the React Native app.
2. **Tier-3 monthly subscription** — recurring billing via **UPI Autopay e-mandate**, using Razorpay **Plans + Subscriptions**. Chart 08 (subscription state machine) should map 1:1 onto Razorpay's subscription states (see Webhooks section).
3. **GST invoices** — generated at the Razorpay layer via the **Razorpay Invoices** product (auto-invoicing per subscription billing cycle; GST-compliant invoices for one-time purchases).
4. **Refunds** — full/partial via the Refunds API. The 7/14/30-day refund-window decision is unconstrained by Razorpay: normal refunds are possible on payments **up to 6 months old**, so any of the three windows works technically (see Refunds below).

Backend: Python FastAPI (AWS Mumbai) creates orders/subscriptions server-side and consumes webhooks. App: React Native/Expo opens Razorpay Checkout via `react-native-razorpay` (native module — needs an Expo dev client/prebuild, **not** Expo Go).

**Named alternative:** Cashfree Payments — compared at the end of the Pricing section.

---

## How it works

### A. One-time tier purchase (Essential / Growth)

```
App (RN)                    FastAPI backend                   Razorpay
   |                              |                               |
   |-- POST /purchase/tier ------>|                               |
   |                              |-- POST /v1/orders ----------->|   (amount=299900|499900, currency=INR,
   |                              |<-- order_id (order_xxx) ------|    receipt=internal_ref, notes={user_id,tier})
   |<-- {order_id, key_id} -------|                               |
   |                              |                               |
   |-- RazorpayCheckout.open({key, order_id, amount, prefill}) -->|   (checkout UI; UPI intent / card / netbanking)
   |<-- {razorpay_payment_id, razorpay_order_id,                  |
   |     razorpay_signature} -----------------------------------  |
   |                              |                               |
   |-- POST /purchase/verify ---->|                               |
   |                              |  verify: HMAC_SHA256(         |
   |                              |    order_id + "|" + payment_id,
   |                              |    key_secret) == signature   |
   |                              |  (constant-time compare)      |
   |<-- tier unlocked ------------|                               |
   |                              |<== webhook: payment.captured / order.paid ==|
   |                              |   (source of truth; reconcile vs client callback)
```

- Payment **capture**: enable auto-capture in Dashboard (one-time "Payment Capture" setting). Payments made **without an order_id cannot be captured and are auto-refunded** — always create the order server-side first.
- Treat the **webhook** (`payment.captured` / `order.paid`) as the source of truth for entitlement, not the client callback (the app can be killed mid-flow; UPI app-switch returns are unreliable).

### B. Tier-3 monthly subscription (UPI Autopay e-mandate)

```
Backend: POST /v1/plans  (period=monthly, interval=1, item={amount, currency})
Backend: POST /v1/subscriptions (plan_id, total_count, customer_notify, start_at, notes)
   → subscription_id (state: created)
App: open checkout with subscription_id
   → customer approves UPI Autopay mandate in their UPI app (₹ auth txn / mandate registration)
   → state: authenticated
On charge_at (each cycle):
   → NPCI/issuer sends 24-hr pre-debit notification (RBI mandate)
   → auto-debit executes (no AFA if amount ≤ ₹15,000)
   → state: active, webhook subscription.charged (payload includes payment + invoice)
Failed charge:
   → state: pending, webhook subscription.pending (Razorpay auto-retries — "Smart Retries" dunning)
   → retries exhausted → state: halted, webhook subscription.halted
   → recovery: customer re-authenticates / old invoice charged manually → back to active
       (previous missed charges are NOT re-attempted after recovery)
Terminal/other: cancelled (API/Dashboard), completed (end of total_count),
               expired (not authenticated by expire_by), paused/resumed (from active only)
```

**Full state set (Chart 08 mapping):** `created → authenticated → active ⇄ pending → halted`, plus `paused/resumed`, `cancelled`, `completed`, `expired`. Razorpay docs do **not** publish the exact retry count/schedule between pending and halted **[unverified — docs only say "we continue to retry while pending" until "all retries are exhausted"]**.

### C. Refund flow

```
Backend: POST /v1/payments/{payment_id}/refund  {amount?: partial, speed: "normal"|"optimum"}
   → refund entity (status: pending → processed | failed)
   → webhook: refund.created / refund.processed / refund.failed / refund.speed_changed
Normal speed: 5–7 working days to customer. Optimum: instant where supported, silently
falls back to normal otherwise (check speed_processed). Normal refund impossible if the
payment is > 6 months old — irrelevant for a 7/14/30-day window.
```

---

## Key endpoints / SDK calls

| Name | Purpose | Key request fields | Key response fields |
|---|---|---|---|
| `POST /v1/orders` | Create order before checkout (one-time tiers) | `amount` (paise, e.g. 299900), `currency` ("INR"), `receipt` (unique, ≤40 chars), `notes` (≤15 k/v pairs) | `id` (order_xxx), `status` (created/attempted/paid), `amount_paid`, `amount_due`, `attempts`, `created_at` |
| `RazorpayCheckout.open()` (react-native-razorpay) | Open native checkout in RN app | `key` (key_id), `order_id`, `amount`, `currency`, `name`, `description`, `prefill` {email, contact}, `theme` | success: `razorpay_payment_id`, `razorpay_order_id`, `razorpay_signature`; error: code + description |
| `POST /v1/plans` | Define Tier-3 billing plan | `period` (monthly), `interval` (1), `item` {name, amount, currency} | `id` (plan_xxx) |
| `POST /v1/subscriptions` | Create subscription (UPI Autopay) | `plan_id`, `total_count` (cycles), `quantity`, `customer_notify`, `start_at`, `expire_by`, `notes`, `addons` | `id` (sub_xxx), `status`, `charge_at`, `short_url` (hosted auth page) |
| `POST /v1/subscriptions/{id}/cancel` | Cancel Tier-3 | `cancel_at_cycle_end` (0 = immediate, 1 = end of cycle) | `status: cancelled` |
| `POST /v1/subscriptions/{id}/pause` / `/resume` | Pause/resume (from active only) | `pause_at` | `status: paused/active` |
| `PATCH /v1/subscriptions/{id}` | Update plan/qty/schedule | `plan_id`, `schedule_change_at` | pending-update details |
| `POST /v1/payments/{id}/refund` | Full/partial refund | `amount` (omit = full), `speed` ("normal"/"optimum"), `notes`, `receipt` | `id` (rfnd_xxx), `status` (pending/processed/failed), `speed_requested`, `speed_processed` |
| `POST /v1/payment_links` | Payment Link (fallback/desktop/email collection) | `amount`, `currency`, `description`, `customer` {name, contact, email}, `notify` {sms, email}, `callback_url`, `expire_by`, `reminder_enable`, `reference_id` | `id` (plink_xxx), `short_url`, `status` |
| `POST /v1/invoices` | GST invoice (Invoices product) | line items with GST/discount/shipping, customer details | invoice id, `short_url`, status; also auto-generated per subscription cycle |

**Example — create order (request → response):**

```json
POST /v1/orders
{ "amount": 299900, "currency": "INR", "receipt": "yesly_essential_u123",
  "notes": { "user_id": "u123", "tier": "essential" } }
```
```json
{ "id": "order_RB58MiP5SPFYyM", "entity": "order", "amount": 299900,
  "amount_paid": 0, "amount_due": 299900, "currency": "INR",
  "receipt": "yesly_essential_u123", "status": "created",
  "attempts": 0, "created_at": 1756455561,
  "notes": { "user_id": "u123", "tier": "essential" } }
```

**Signature verification (Python, FastAPI side):**

```python
import hmac, hashlib

def verify_payment_signature(order_id: str, payment_id: str, signature: str, key_secret: str) -> bool:
    msg = f"{order_id}|{payment_id}".encode()
    expected = hmac.new(key_secret.encode(), msg, hashlib.sha256).hexdigest()
    return hmac.compare_digest(expected, signature)
```

(For subscriptions, the checkout signature is HMAC-SHA256 over `payment_id + "|" + subscription_id` — verify against Razorpay's `razorpay-python` SDK `utility.verify_subscription_payment_signature`, which handles both variants.)

---

## Auth

- **API auth:** HTTP Basic — `key_id:key_secret` (e.g. `curl -u rzp_live_xxx:secret`). Key pairs generated in Dashboard; separate **test** (`rzp_test_...`) and **live** (`rzp_live_...`) pairs.
- `key_id` is public (shipped to the app for checkout); `key_secret` stays server-side only (AWS Secrets Manager / SSM in Mumbai). Never embed `key_secret` in the RN bundle.
- **Webhook auth:** separate webhook secret (set when configuring the webhook), used for `X-Razorpay-Signature` HMAC — distinct from `key_secret`.
- Official `razorpay-python` SDK wraps Basic auth + signature utilities.

---

## Webhooks

**Events Yesly should subscribe to:**

| Event | Yesly action |
|---|---|
| `payment.captured` | Unlock tier (one-time purchase source of truth) |
| `payment.failed` | Log; prompt retry in app |
| `order.paid` | Idempotent confirmation of order settlement state |
| `payment.authorized` | Only relevant if manual capture is used (avoid; use auto-capture) |
| `subscription.authenticated` | Mandate approved — Chart 08: awaiting first charge |
| `subscription.activated` | Tier-3 entitlement ON |
| `subscription.charged` | Renewal success; attach invoice; extend entitlement |
| `subscription.pending` | Charge failed, retries ongoing — grace-period messaging |
| `subscription.halted` | Retries exhausted — start recovery flow (re-auth / manual invoice charge); decide entitlement cutoff |
| `subscription.cancelled` / `completed` / `paused` / `resumed` / `updated` | Sync Chart 08 state |
| `refund.created` / `refund.processed` / `refund.failed` | Refund lifecycle + user notification |

**Verification:** every delivery carries `X-Razorpay-Signature` = HMAC-SHA256(**raw request body**, webhook secret). Razorpay explicitly warns: *do not parse or re-serialize the body before computing the HMAC* — verify the raw bytes. Use constant-time comparison.

**Idempotency / replay:** each event carries a unique `x-razorpay-event-id` header — persist processed IDs (e.g. Postgres unique index or DynamoDB conditional put) and skip duplicates. Razorpay retries failed deliveries, so duplicates and out-of-order arrivals will happen; make every handler idempotent and state-transition-guarded (e.g. don't re-extend entitlement on a replayed `subscription.charged`). Note: if you rotate the webhook secret, retried in-flight webhooks are still signed with the **old** secret.

**Ops constraints:** webhook endpoint must be on port 80/443; whitelist Razorpay webhook IPs; respond 2xx quickly (do work async via queue). Razorpay's guidance: rely on webhooks for automation, use API polling (`GET /v1/orders/{id}/payments`) only for user-facing instant confirmation.

---

## Sandbox

- **Test mode** via `rzp_test_` keys — full API + Dashboard parity, no real money ("simulated transaction"), mock bank page with Success/Failure buttons.
- **Test UPI IDs:** `success@razorpay` (success flow), `failure@razorpay` (failure flow). Caveat from docs: in test mode, *cancelling* a UPI payment still results in success — UPI cancellation can only be tested in live mode.
- **Test cards:** published Indian + international test card numbers; **subscription test tokens are valid only 3 days**, which limits how long a test-mode recurring cycle can be exercised.
- Webhooks fire in test mode; point them at a dev endpoint (ngrok/API Gateway dev stage) to test signature verification and idempotency.
- UPI Autopay mandate testing in test mode is not documented in the public test-details pages **[unverified — plan a small live-mode pilot with real ₹ amounts for mandate UX validation]**.

---

## Pricing

Verified on razorpay.com/pricing, July 2026. **All fees attract 18% GST.**

| Item | Rate (as published) |
|---|---|
| Standard domestic (all instruments) | **2%** per successful transaction |
| UPI / RuPay debit | Zero MDR, but **2% Razorpay platform fee applies** (note: UPI is no longer free on Razorpay's standard plan) |
| Visa/MC/Amex/Diners cards | 2% |
| Corporate/business cards | 2.15% |
| International cards | up to 3% |
| Card subscriptions (recurring) | 0.9% + platform fee per transaction |
| **UPI Autopay / e-mandate / NACH recurring** | **"Pricing on request" — not public [unverified]**; get a quote before committing Tier-3 economics |
| Payment Links / Pages / Buttons | 0.2% + payment gateway fees |
| Instant refunds | ₹7.99–₹14.99 per refund (by amount); normal refunds free |
| Setup / AMC / refund processing | ₹0 |
| Instant settlement | Available on demand; **fee not published [unverified]** |
| Enterprise pricing | Negotiable above ~₹5,00,000/month volume |

**Ballpark for Yesly:** a ₹4,999 Growth purchase costs ≈ ₹100 + 18% GST ≈ **₹118 (~2.36% all-in)** at standard rates. Negotiate — flat 2% on UPI is above market and Razorpay routinely discounts for registered businesses.

**Settlement:** standard **T+2 working days** (domestic), extended by bank holidays; INR only; requires completed KYC/activation. Instant Settlements (minutes instead of T+2) available on request. Partial settlements occur automatically when refunds reduce the live balance.

**Cashfree (named alternative), verified July 2026:**
- Standard **1.95%**; promo **1.6% flat** for merchants signing up 18 Sep 2025 – 31 Jul 2026 (12 months, conditions: ≤₹1 Cr/month GTV, UPI ≥40% of GTV). International cards 2.69% (promo) / 2.99%.
- Subscriptions: UPI Autopay e-mandate supported (auto-debit up to ₹15,000 per charge without AFA) — same NPCI rails as Razorpay.
- Settlement: typically 24–48 hours (effectively T+1 in practice), instant-settlement options available.
- Net: Cashfree is **cheaper on headline rates and faster on settlement**; Razorpay has the more mature Subscriptions/Invoices product surface, better docs, and the more battle-tested RN SDK. Reasonable strategy: build on Razorpay, keep the gateway behind an internal interface so Cashfree can be a drop-in for cost negotiation leverage.

---

## Compliance notes (India / SEBI / RBI)

**RBI recurring-payment rules — Digital Payments E-Mandate Framework, 2026** (issued **21 April 2026**, effective immediately; consolidates 8 circulars from 2019–2024 across cards, PPIs, and UPI):

- **AFA-free limit:** subsequent recurring debits up to **₹15,000 per transaction** need no additional factor authentication after mandate setup. Enhanced limit of **₹1,00,000** applies only to specific categories: **mutual fund subscriptions, insurance premiums, credit-card bill payments**. Yesly's Tier-3 advisory-fee subscription is **not** one of these categories, so the **₹15,000 cap applies** — fine for a monthly advisory fee, but caps future annual-billing-on-mandate ideas.
- **Pre-debit notification:** issuer must notify the customer **≥24 hours before each debit** (merchant name, amount, debit date/time, mandate reference, reason). Still mandatory under the 2026 framework; only FASTag/NCMC auto-replenishment mandates are exempt. Handled by the issuer/NPCI rails via Razorpay — Yesly doesn't build this, but must design UX around it (customers can and do cancel after the notification).
- Customers must be able to cancel/pause mandates at issuer/UPI-app level (outside Yesly's control) — `subscription.cancelled`/`halted` webhooks are the only way Yesly finds out. Post-debit alerts and grievance redressal are issuer obligations.

**SEBI / RIA:** SEBI's fee-collection ecosystem — the **Centralized Fee Collection Mechanism (CeFCoM, via BSE)** — is currently **optional** for RIAs **[verify current SEBI stance at build time; SEBI has periodically signaled it may become mandatory]**. Collecting via Razorpay is permissible today; keep advisory-fee invoices/records aligned with SEBI RIA fee regulations (fee caps under the AUA/fixed-fee modes) — that logic lives in Yesly's backend, not Razorpay.

**GST invoicing at the Razorpay layer:** the **Razorpay Invoices** product is **active as of July 2026 — no deprecation notice found** (docs and marketing pages current; searches surfaced no sunset announcement). It supports "GST-compliant invoices": add GST, discounts, shipping and it computes totals; subscriptions **auto-generate an invoice per billing cycle**. Caveat: public docs do not spell out GSTIN/HSN-SAC field-level behavior **[unverified — validate that generated invoices meet House of Alpha's CA requirements for SAC 9971/financial-service invoicing; otherwise generate GST invoices in Yesly's backend from `payment.captured`/`subscription.charged` webhooks and treat Razorpay invoices as payment collateral only]**.

**Refund window (7/14/30-day decision):** Razorpay imposes no meaningful constraint — normal refunds work on payments up to 6 months old, refund processing is free, and partial refunds are supported. Decide on product/SEBI-comms grounds. Note refunds do **not** return the ~2% platform fee **[unverified — confirm fee treatment on refunds with Razorpay sales; historically Razorpay does not refund its fee]**.

**App-store / IAP position (verified against current policies, July 2026):**

- **Google Play:** the Payments policy **exempts financial products/services from Google Play Billing** — apps selling "insurance, stock trades, **investment consulting**, or tax preparation" *should not* use Play's billing system; Play defines financial products as those "related to the management or investment of money… including personalized advice." A SEBI-registered RIA's advisory tiers fit squarely. **Verdict: Yesly can (and per policy, should) use Razorpay in-app on Android.** Comply with Play's separate **Financial Services/Financial Features policy** (declarations for finance apps, India personal-loan rules don't apply, but the finance-app declaration form does).
- **Apple App Store:** more ambiguous.
  - **3.1.3(e)** — goods/services **consumed outside the app** must use non-IAP payments. **3.1.3(d)** — **real-time person-to-person services** (docs example list includes tutoring, medical consultations, fitness training; 1:1 only) may use non-IAP; one-to-few/one-to-many must use IAP.
  - **3.1.1** — unlocking "features, functionality, or digital content within the app" (subscriptions, premium content, feature unlocks, SaaS access) **requires IAP**.
  - **Risk for Yesly:** if Essential/Growth/Tier-3 primarily unlock **in-app digital content/features** (dashboards, reports, premium screens), Apple review can classify them as digital goods → IAP required (15–30% commission). If the tiers are positioned and substantiated as **human advisory services** (1:1 advisor access, real-world financial advice consumed outside the app, with in-app content as ancillary), 3.1.3(d)/(e) supports Razorpay. Indian fintech precedent (broker/MF apps) leans toward Apple tolerating external payment for regulated financial services, but this is **App Review discretion, not a written India carve-out [unverified as a guarantee]**.
  - 2025–2026 external-link changes (US storefront link-outs, regional entitlements) do **not** currently extend to the India storefront **[verified as of the current guidelines text: link-out relief is US/designated-storefront scoped]**.
  - **Mitigations:** anchor Tier-3 to scheduled 1:1 advisor interactions; keep purchase flows web-based (Payment Links) as a fallback if Apple rejects; do not reference external payment from within iOS UI in ways 3.1.1 prohibits.

**Data/infra:** payment data stays with Razorpay (PCI-DSS is theirs); store only IDs (order/payment/subscription/refund) + amounts in Yesly's Mumbai RDS — aligns with RBI data-localization expectations since Razorpay processes in India.

---

## Gotchas & effort

**Gotchas**

1. **UPI Autopay mandate friction:** setup is a multi-app dance (Yesly → checkout → UPI app → back). Drop-off at mandate approval is materially higher than one-time UPI. Show explicit "you'll approve a mandate in your UPI app" messaging; handle the user never returning (subscription stuck in `created`/`expired`).
2. **Mandate execution failures are common in India** (insufficient balance at 0-hour execution, issuer downtime, customer blocks after the 24-hr pre-debit SMS). Design the `pending → halted` journey deliberately: grace period length, entitlement cutoff, in-app recovery CTA. Razorpay does **not** publish retry counts — don't hardcode assumptions; drive purely off webhooks.
3. **`halted` recovery quirk:** after recovery to `active`, **missed charges are not re-attempted** — if Yesly wants back-payment, it must charge the old invoice manually or eat the gap.
4. **Webhook idempotency/replay:** dedupe on `x-razorpay-event-id`; guard state transitions (events arrive out of order); verify HMAC on the **raw body** (FastAPI: use `await request.body()`, not the parsed model — JSON re-serialization breaks the signature). Old-secret signing during rotation.
5. **Signature verification pitfalls:** compare with `hmac.compare_digest`; verify on the server, never in the app; remember the subscription-checkout signature uses `payment_id|subscription_id` (different from the order flow's `order_id|payment_id`).
6. **Auto-capture vs manual:** use auto-capture. Uncaptured authorized payments auto-refund; orderless payments cannot be captured at all. Late-authorized UPI payments (customer approves after checkout timeout) arrive **only** via webhook — without webhook handling you'll take money and grant nothing.
7. **Client success ≠ entitlement:** never unlock a tier from the RN success callback alone; verify signature server-side AND reconcile with `payment.captured`.
8. **Expo:** `react-native-razorpay` is a native module — requires `expo prebuild`/dev-client + EAS builds; it will not run in Expo Go. Budget CI changes.
9. **UPI cancellation untestable in test mode**; subscription test tokens expire in 3 days — plan a live-mode pilot for the full mandate lifecycle.
10. **Pricing trap:** UPI now carries Razorpay's 2% platform fee on the standard plan, and UPI-Autopay recurring pricing is quote-only — get commercial terms in writing before finalizing Tier-3 price points.

**Effort estimate**

| Workstream | Effort |
|---|---|
| One-time purchase (orders, RN checkout, verify, webhooks, entitlement) | ~1.5–2 dev-weeks |
| Subscriptions/UPI Autopay (plans, mandate UX, full Chart-08 state sync, dunning UX) | ~2–3 dev-weeks |
| Refunds + support tooling | ~0.5 week |
| GST invoicing (Razorpay Invoices integration or in-house generation) | ~0.5–1 week |
| Apple review contingency (web purchase fallback via Payment Links) | ~0.5 week buffer |

---

## Sources

- https://razorpay.com/docs/api/orders/create/ — Create Order API, fields, example JSON
- https://razorpay.com/docs/payments/subscriptions/ — Subscriptions overview, UPI Autopay/e-mandate, charge_at, Smart Retries, auto-invoicing
- https://razorpay.com/docs/payments/subscriptions/states/ — nine subscription states, pending/halted behavior, halted recovery
- https://razorpay.com/docs/api/payments/subscriptions/ — Plans/Subscriptions endpoints, cancel/pause/resume/update, webhook event list
- https://razorpay.com/docs/webhooks/ — webhook fundamentals, ports/IP allowlist, automation guidance
- https://razorpay.com/docs/webhooks/validate-test/ — X-Razorpay-Signature HMAC-SHA256 on raw body, x-razorpay-event-id dedupe, secret-rotation caveat
- https://razorpay.com/docs/api/refunds/create-instant/ — refund endpoint, speed normal/optimum, response fields, 5–7 day timeline, 6-month age limit
- https://razorpay.com/docs/payments/settlements/ — T+2 settlement, instant settlements, partial settlements
- https://razorpay.com/pricing/ — 2%/2.15%/3% rates, UPI platform fee, card subscriptions 0.9%, payment links 0.2%, instant refund fees, 18% GST
- https://razorpay.com/docs/payments/invoices/ — Invoices product, GST-compliant invoicing (active, no deprecation notice)
- https://razorpay.com/docs/api/payments/payment-links/ — Payment Links API
- https://razorpay.com/docs/payments/payment-gateway/react-native-integration/standard/ — RN SDK prerequisites (RN 0.60+)
- https://razorpay.com/docs/payments/payments/test-card-upi-details/ and https://razorpay.com/docs/payments/payments/test-upi-details/ — test cards, success@razorpay / failure@razorpay, 3-day subscription test tokens
- https://www.scconline.com/blog/post/2026/04/24/rbi-issues-digital-payments-e-mandate-framework-2026/ — RBI Digital Payments E-Mandate Framework, 2026 (21 Apr 2026)
- https://www.businesstoday.in/amp/personal-finance/news/story/rbi-auto-debit-rules-explained-what-new-changes-mean-for-your-upi-and-card-payments-528507-2026-05-02 — ₹15,000 AFA-free cap, ₹1 lakh MF/insurance/credit-card categories, 24-hr pre-debit notification, FASTag/NCMC exemption
- https://www.npci.org.in/product/autopay — NPCI UPI AutoPay product page
- https://www.cashfree.com/payment-gateway-charges/ — Cashfree 1.95% standard / 1.6% promo (to 31 Jul 2026), international rates, settlement 24–48 hrs, UPI Autopay ≤₹15,000
- https://support.google.com/googleplay/android-developer/answer/10281818 — Google Play Payments policy: financial products (stock trades, investment consulting) must NOT use Play Billing
- https://developer.apple.com/app-store/review/guidelines/ — Apple 3.1.1 (IAP for digital features), 3.1.3(d) person-to-person real-time services, 3.1.3(e) goods/services consumed outside the app, external-link entitlement scope
