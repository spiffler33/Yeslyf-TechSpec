# 06 — WhatsApp Business Platform (BSP Decision: AiSensy vs Gupshup vs Wati vs Direct Cloud API)

> **Open Decision 3.** WhatsApp is Yesly's high-attention nudge channel: transactional-style reminders to retail investors (SIP due, goal drift, plan review, document pending). Sent from **Container 5** (async notification dispatcher, Celery) alongside Expo push and email. This doc covers the platform fundamentals, the direct Meta Cloud API, the three BSP candidates, pricing in INR, and a recommendation.

## Overview & role in Yesly

**WhatsApp Business Platform** is Meta's API product for programmatic messaging. Key concepts:

- **Cloud API vs On-Premises API:** the self-hosted On-Premises API is **dead**. Meta closed new sign-ups on **July 1, 2024**, shipped new features exclusively to Cloud API from v2.53 (Jan 2024), and the final supported On-Prem client version **expired October 23, 2025** — it can no longer send messages. Everything below is Cloud API (`graph.facebook.com`). Cloud API supports up to 1,000 msgs/sec per number, ~99.9% uptime per Meta.
- **WABA (WhatsApp Business Account):** the container in Meta Business Manager that owns your phone numbers, message templates, and billing. Created via Meta Business Manager (direct route) or via a BSP's embedded signup. Requires **Meta Business Verification** (GST/CIN docs for an Indian entity) to display brand name and unlock higher tiers.
- **Phone-number registration:** a dedicated number (cannot simultaneously run the consumer/SMB WhatsApp app) is added to the WABA, verified via SMS/voice OTP (`/request_code` + `/verify_code`), then registered for Cloud API (`/register` with a 6-digit PIN — this PIN is also the two-step-verification lock). Display name must match the verified business.
- **24-hour customer-service window:** when a *user* messages you (or replies), a 24h window opens during which you may send **free-form messages** (any content, no template) at **no charge** ("service" messages are free since Nov 2024). Outside this window, business-initiated messages **must** use a pre-approved **template**.
- **Template messages** — every template is classified into one of three categories:
  - **Utility** — transactional, tied to a specific user action/account: order & payment updates, reminders the user opted into, account alerts. → *Yesly's nudges should be authored to qualify here.*
  - **Marketing** — promotions, upsell, re-engagement, anything not strictly transactional. Costs ~7–8× utility in India and is subject to per-user caps (see Compliance).
  - **Authentication** — OTP/verification codes only, fixed structure with copy-code/one-tap button.
- **Template approval:** submitted via WhatsApp Manager (or API/BSP UI); automated ML review usually decides in minutes–hours (Meta SLA: up to 24–48h). **Common rejection reasons:** placeholder at start/end of body, malformed `{{1}}` params or too many placeholders (looks abusable), category mismatch (utility text that reads promotional gets reclassified or rejected), prohibited content (financial-services claims phrased as guarantees, alcohol/tobacco/weapons, competitor brand names), threatening/urgency language ("act now or lose access"), grammar/formatting issues, duplicate of an existing template. Rejected templates can be edited and resubmitted or appealed.
- **Pricing model transition:** Meta moved from **per-conversation** to **per-message** pricing **effective July 1, 2025**. You now pay per *delivered template message* (marketing/utility/authentication); service conversations are free; **utility templates delivered inside an open 24h customer-service window are free**. Volume-tier discounts (reset monthly, per country+category) apply to utility & authentication only — marketing has no volume discounts. India is billed in INR.
- **Messaging limits (business-initiated, per number, rolling 24h) — quality-rating tiers:**
  | Tier | Unique customers / 24h | How you get it |
  |---|---|---|
  | Unverified trial | 250 | default before business verification |
  | Tier 1 | 1,000 | after business verification + display-name approval |
  | Tier 2 | 10,000 | auto-upgrade: reach ~½ of current limit within 7 days with **Medium/High quality** |
  | Tier 3 | 100,000 | same auto-upgrade mechanics |
  | Tier 4 | Unlimited | same |

  Quality rating (Green/Yellow/Red in WhatsApp Manager) is driven by user blocks/reports and read rates; a Red/low rating can freeze or **downgrade** your tier and pause templates.

**Role in Yesly:** Container 5's WhatsApp channel worker takes queued nudge jobs, renders template params (name, amount, due date, deep link), calls the send API (BSP or Cloud API), records the returned message ID, and consumes status webhooks (sent/delivered/read/failed) to update the notification ledger and drive fallback (e.g., if WhatsApp fails → Expo push/email).

## How it works

```
┌──────────────────────────────┐
│ Container 5 (Celery worker)  │
│ - nudge job: user, template, │
│   params, consent check      │
│ - idempotency key            │
└──────────┬───────────────────┘
           │ HTTPS POST (template message)
           ▼
┌─────────────────────────────────────────────────────────┐
│ Option A: BSP layer                Option B: Direct     │
│ (AiSensy / Gupshup / Wati)         Meta Cloud API       │
│ - own API shape & auth             POST graph.facebook  │
│ - forwards to Meta Cloud API       .com/v23.0/{phone_   │
│   on your WABA                     number_id}/messages  │
└──────────┬──────────────────────────────────────────────┘
           ▼
┌──────────────────────────────┐
│ Meta / WhatsApp platform     │
│ - template + category check  │
│ - per-user marketing caps    │
│ - quality/tier enforcement   │
│ - billing per delivered msg  │
└──────────┬───────────────────┘
           ▼
     User's WhatsApp app (device)
           │  status transitions
           ▼
┌──────────────────────────────┐
│ Status webhooks (reverse)    │
│ sent → delivered → read      │
│        └─ failed (+errors[]) │
│ Meta → your webhook URL      │
│ (direct) or BSP → your       │
│ callback URL (BSP route)     │
│ → FastAPI /webhooks/whatsapp │
│ → update notification ledger │
│ → fallback channel if failed │
└──────────────────────────────┘
```

Sequence: dispatcher sends → API returns `wamid` (accepted, **not** delivered) → Meta attempts delivery → webhook fires `sent`, then `delivered`, then (if read receipts on) `read`; or `failed` with an `errors[]` array. Treat the ledger as: `queued → accepted(wamid) → sent → delivered → read | failed`.

## Key endpoints

Direct Meta Cloud API — base `https://graph.facebook.com/v23.0` (use latest Graph version; pin it in config):

| Method | Path | Purpose | Key request fields | Key response fields |
|---|---|---|---|---|
| POST | `/{phone-number-id}/messages` | Send template / text / media / interactive message | `messaging_product`, `to`, `type`, `template{name, language, components}` | `messages[].id` (wamid), `contacts[].wa_id`, `messages[].message_status` |
| POST | `/{phone-number-id}/messages` | Mark inbound message read | `status:"read"`, `message_id` | `success` |
| GET | `/{waba-id}/message_templates` | List templates + statuses | `fields`, `status` filter | `data[]: name, status (APPROVED/REJECTED/PAUSED), category, quality_score` |
| POST | `/{waba-id}/message_templates` | Create template programmatically | `name`, `category`, `language`, `components[]` | `id`, `status`, `category` |
| POST | `/{phone-number-id}/register` | Register number for Cloud API | `pin` (6-digit 2FA) | `success` |
| GET | `/{phone-number-id}` | Number health | `fields=quality_rating,throughput,verified_name` | `quality_rating`, `throughput.level` |
| GET/POST | your webhook URL | Verify (GET `hub.challenge`) / receive events (POST) | — | — |

BSP equivalents (each wraps the same Meta send under its own API):

| Provider | Send endpoint | Auth | Notes |
|---|---|---|---|
| AiSensy | `POST https://backend.aisensy.com/campaign/t1/api/v2` | `apiKey` (JWT) in JSON body | Sends are modeled as "API campaigns" — you pre-create a campaign bound to an approved template, then POST `campaignName + destination + templateParams[]` |
| Gupshup | `POST https://api.gupshup.io/wa/api/v1/template/msg` | `apikey` header | Form-encoded: `source`, `destination`, `template={"id":..., "params":[...]}`; async — returns `submitted` + message ID, real status via callback |
| Wati | `POST https://live-mt-server.wati.io/{tenant-id}/api/v1/sendTemplateMessage?whatsappNumber={to}` (also v3 `/api/ext/v3/messageTemplates/send`) | `Authorization: Bearer <token>` | JSON: `template_name`, `broadcast_name`, `parameters[]`; API quota is plan-gated |

**Example — direct Cloud API template send (Yesly SIP-reminder nudge):**

```http
POST https://graph.facebook.com/v23.0/{phone-number-id}/messages
Authorization: Bearer {SYSTEM_USER_TOKEN}
Content-Type: application/json
```
```json
{
  "messaging_product": "whatsapp",
  "recipient_type": "individual",
  "to": "919876543210",
  "type": "template",
  "template": {
    "name": "sip_due_reminder",
    "language": { "code": "en" },
    "components": [
      {
        "type": "body",
        "parameters": [
          { "type": "text", "text": "Vatsal" },
          { "type": "currency",
            "currency": { "fallback_value": "₹5,000", "code": "INR", "amount_1000": 5000000 } },
          { "type": "text", "text": "5 July 2026" }
        ]
      },
      {
        "type": "button", "sub_type": "url", "index": "0",
        "parameters": [ { "type": "text", "text": "goals/1234" } ]
      }
    ]
  }
}
```

**Response (202-style accept):**

```json
{
  "messaging_product": "whatsapp",
  "contacts": [
    { "input": "919876543210", "wa_id": "919876543210" }
  ],
  "messages": [
    { "id": "wamid.HBgMOTE5ODc2NTQzMjEwFQIAERgSN0E3...", "message_status": "accepted" }
  ]
}
```

`messages[].id` (the **wamid**) is the correlation key for all subsequent status webhooks — persist it on the notification row. `accepted` ≠ delivered.

## Auth

- **Direct Cloud API:** create a **System User** in Meta Business Manager (Admin role), assign it the WhatsApp app + WABA assets, and generate a **permanent (non-expiring) system-user access token** with `whatsapp_business_messaging` + `whatsapp_business_management` permissions. Use as `Authorization: Bearer …`. Do **not** use the 24h temporary token from the Getting Started page or 60-day user tokens in production. Store in AWS Secrets Manager (Mumbai); rotate by issuing a new token, swapping, revoking old.
- **Webhook security (direct):** GET verification handshake with your `hub.verify_token`; validate every POST via `X-Hub-Signature-256` (HMAC-SHA256 of raw body with your **App Secret**).
- **BSP keys:** AiSensy — long-lived JWT `apiKey` passed *in the request body* (treat as secret; body-level keys leak into logs easily — scrub). Gupshup — `apikey` header per app. Wati — per-tenant Bearer JWT. All are static keys: same Secrets Manager treatment, plus IP allowlisting where offered (Wati Business plan supports IP whitelisting).

## Webhooks/events

Subscribe to the `messages` field of the WhatsApp Business Account object (direct) or configure the BSP's callback URL. Two event families arrive on the same endpoint:

1. **Inbound user messages** (`value.messages[]`) — replies to nudges; open the free 24h service window; route to support tooling or auto-ack.
2. **Status updates** (`value.statuses[]`) — the delivery lifecycle Yesly cares about: `sent` → `delivered` → `read` → or `failed`.

**Status webhook shape (per-message pricing era):**

```json
{
  "object": "whatsapp_business_account",
  "entry": [{
    "id": "WABA_ID",
    "changes": [{
      "field": "messages",
      "value": {
        "messaging_product": "whatsapp",
        "metadata": { "display_phone_number": "91XXXXXXXXXX", "phone_number_id": "PHONE_NUMBER_ID" },
        "statuses": [{
          "id": "wamid.HBgMOTE5ODc2NTQzMjEwFQIAERgSN0E3...",
          "status": "failed",
          "timestamp": "1751873400",
          "recipient_id": "919876543210",
          "pricing": { "billable": true, "pricing_model": "PMP", "category": "utility", "type": "regular" },
          "errors": [{
            "code": 131047,
            "title": "Re-engagement message",
            "message": "Re-engagement message",
            "error_data": { "details": "Message failed to send because more than 24 hours have passed since the customer last replied to this number." },
            "href": "https://developers.facebook.com/docs/whatsapp/cloud-api/support/error-codes/"
          }]
        }]
      }
    }]
  }]
}
```

Notes:

- `pricing.pricing_model: "PMP"` = per-message pricing; `pricing.billable: false` appears on free messages (e.g., utility inside the service window).
- **Failure codes to handle explicitly:** `131047` outside 24h window (free-form attempt) · `131049` / `130472` Meta chose not to deliver — **per-user marketing cap / healthy-ecosystem limits** (expect these on marketing templates in India; do not blind-retry) · `131026` undeliverable (no WhatsApp / old client) · `132000` template param count/format mismatch · `132001` template not found or not approved in that language · `132015` template paused for quality · `131048` / `131056` spam or pair rate limit · `80007`/`4` throughput/app rate limit · `190` token expired · `100` invalid parameter.
- Webhooks are at-least-once and **can arrive out of order** (e.g., `read` before `delivered`) — make ledger updates idempotent and monotonic by wamid + status rank.
- BSP route: AiSensy/Wati surface delivery status primarily in their dashboards; programmatic status webhooks exist but are plan-gated/limited (Gupshup's DLR callbacks are the most complete of the three). If precise per-message ledger tracking matters, this favors Gupshup or direct.

## Sandbox/testing

- **Meta test assets (direct route):** every new WhatsApp app gets a **test WABA + test phone number**. You can message up to **5 recipient numbers** (each verified by OTP) **for free**, using the pre-approved `hello_world` template or your own drafts — ideal for Container 5 integration tests before business verification completes. The Getting Started page issues a temporary 24h token; swap for the system-user token.
- **Template staging:** create templates on the test WABA first; approval behavior mirrors production.
- **BSP sandboxes:** Gupshup has a sandbox mode with pre-approved templates and a shared test number; Wati and AiSensy offer trials/demo workspaces rather than true sandboxes.
- **Load/dry-run:** the tiers apply per number — a fresh production number starts at 250 (unverified) / 1K uniques per day; plan pilot cohorts accordingly and warm the number up gradually.
- **Local dev:** tunnel webhooks (ngrok/cloudflared) and replay captured `statuses` payloads in pytest fixtures; don't rely on live webhooks for CI.

## Pricing

All figures **exclude 18% GST**. Meta bills India in INR. **Per-message pricing since July 1, 2025** (per *delivered* template message).

**Meta base rates — India:**

| Category | July 1, 2025 (verified via Meta/BSP notices) | Jan 1, 2026 update (last verified via AiSensy/Helo BSP notices, as of July 2026) |
|---|---|---|
| Marketing | ~₹0.88/msg | **~₹0.86–1.09/msg** (≈10% increase; no volume discounts) |
| Utility | ~₹0.115–0.125/msg | **~₹0.13–0.145/msg** (volume tiers: ~5–30% off at scale) |
| Authentication | ~₹0.115–0.125/msg | **~₹0.13–0.145/msg** (volume tiers) — authentication-international ~₹2.30 |
| Service (user-initiated, 24h window) | Free | Free |
| Utility delivered inside an open 24h service window | **Free** | **Free** |

> **Verify before build:** the Jan-2026 numbers above come from BSP pricing notices (AiSensy quotes ₹1.09 marketing / ₹0.145 utility & auth; other aggregators quote ₹0.86 / ₹0.13 — the spread is GST/volume-tier presentation). Pull the authoritative rate card from Meta's pricing page (developers.facebook.com → WhatsApp → Pricing, downloadable CSV) at build time. Do not hard-code rates.

**BSP fees on top (last verified July 2026 — verify before signing):**

| Provider | Platform fee | Per-message markup | Model |
|---|---|---|---|
| **AiSensy** | Free Forever tier; Basic ~₹1,500/mo; Pro higher (₹~2,400+/mo) | **None** — claims zero markup, Meta pass-through via prepaid "conversation credits" | Subscription + pass-through |
| **Gupshup** | Usage-based; effectively ~₹4,000+/mo minimum spend for small accounts | **~US$0.001/msg (~₹0.09)** Gupshup fee on every outgoing+incoming message, media >64KB extra | Pass-through + small per-message fee |
| **Wati** | Growth ~US$59/mo (annual, ~₹5k), Pro ~US$99–119/mo, Business ~US$279/mo | **~20% markup** on Meta template rates (India ~20.2% per third-party audits) | Subscription + marked-up messages |
| **Direct Meta Cloud API** | ₹0 | ₹0 (pure Meta rates) | Pass-through; you pay Meta directly (INR billing, credit card/credit line on the WABA) |

**Ballpark for Yesly** (e.g., 50k utility nudges + 5k marketing/mo, Jan-2026 rates): Meta cost ≈ 50,000×₹0.145 + 5,000×₹1.09 ≈ **₹7,250 + ₹5,450 ≈ ₹12,700/mo** + GST. Wati adds ~20% (≈₹2,500) + subscription; Gupshup adds ≈₹5k fees; AiSensy adds only its subscription; direct adds nothing.

## Compliance notes

- **Opt-in (Meta policy):** you must obtain a user's prior opt-in before sending business-initiated (template) messages — collected in-app (Yesly onboarding toggle), on web, or via inbound WhatsApp message. Keep the opt-in record (timestamp, method, scope).
- **DPDP Act 2023 (India):** consent must be free, specific, informed, unambiguous — no pre-ticked boxes. **Separate consents for transactional nudges vs marketing content**; migrating a user from email/SMS to WhatsApp requires fresh consent for the WhatsApp channel. Maintain a consent ledger (date, method, purpose, withdrawal) — Container 5 must check consent state per message class before dispatch. DPDP Rules were notified in 2025; verify current enforcement timelines before launch.
- **TRAI:** TRAI's TCCCP/DLT regime governs SMS/voice, **not** WhatsApp (OTT). There is no DLT registration for WhatsApp — but this makes Meta policy + DPDP the binding constraints, and regulators have signaled interest in OTT commercial messaging; monitor.
- **SEBI RIA overlay:** anything that promotes advisory services (vs servicing an existing plan) is an advertisement under SEBI's RIA advertisement code — route such content through compliance review and classify it as **marketing**, never disguise it as utility.
- **Meta per-user marketing caps (India):** rolled out **Feb 6–13, 2024** in India, global from **May 23, 2024**: a user can receive only a limited number of marketing template messages **from all businesses combined** (widely reported as ~2 per 24h in India; exact cap is dynamic per user) unless they've engaged/replied. Excess sends are silently not delivered (webhook `failed` with `131049`/`130472`) — **you are not charged**, but reach is not guaranteed. Utility & authentication are unaffected.
- **Marketing pause history (2023–24):** Meta announced deliverability controls in late 2023; from **April 1, 2024** it began temporarily **pausing marketing templates/campaigns with persistently low read rates** until edited. Combined with the caps above, this is why marketing-on-WhatsApp is an unreliable channel in India — plan marketing reach on push/email, keep WhatsApp for utility.
- **Data residency:** Cloud API processes messages on Meta infrastructure (not guaranteed in-region); message *content* transits Meta globally. For a SEBI RIA this is acceptable for notification content, but do not put portfolio holdings/PII beyond what the nudge needs into template params. Local storage of webhook payloads stays in AWS Mumbai.

## Gotchas & effort

- **Template category reclassification:** Meta's classifier can bump a "utility" template to "marketing" (7–8× the cost, subject to caps) at creation or later. Write utility templates that reference a specific user action/account state ("your SIP of {{amount}} is due {{date}}") and avoid promotional adjectives/CTAs. Monitor `category` on the template list API.
- **Rejections & pausing:** expect a few iterations per template (placeholder placement, urgency wording). Approved templates can later be **paused/disabled** for low quality — subscribe to `message_template_status_update` webhooks (direct) and keep a fallback template.
- **Silent marketing drops:** per-user caps mean marketing sends fail without recourse; build the ledger so `131049` triggers channel fallback, not retry.
- **Number quality degradation:** blocks/reports lower quality → tier downgrade or send freeze. Mitigate: strict opt-in, easy opt-out keyword (STOP handling is on you), frequency caps in Container 5, warm up new numbers. Don't burn the number on aggressive campaigns — it's hard to recover and the number is your identity.
- **24h-window nuance:** free-form replies outside the window fail with `131047`; support-agent replies must check window state or auto-fall back to a utility template.
- **Webhook hygiene:** at-least-once + out-of-order delivery; dedupe on `wamid + status`; respond 200 fast (queue processing) or Meta backs off and eventually drops the subscription.
- **BSP lock-in:** each BSP has proprietary API shapes (AiSensy "campaigns", Gupshup form-encoded, Wati tenant URLs). Wrap the channel behind a `WhatsAppSender` interface in Container 5 so a later BSP→direct migration is a driver swap. Note: a phone number can be migrated between BSPs/direct, and templates can move with the WABA, but plan a maintenance window.
- **Estimated effort (Container 5 integration):**
  - Direct Cloud API: **~2–3 dev-weeks** (send worker + retry/backoff, webhook receiver + signature check, ledger, template CRUD sync, consent gate) **+ ~1–2 weeks elapsed ops** (business verification, display name, number procurement, template approvals).
  - Via BSP (any of the three): **~1 dev-week** integration; onboarding/verification still ~1 week elapsed (BSP hand-holds the Meta paperwork).
  - Template design/compliance review: ongoing, ~1 day per template batch.

### BSP comparison & recommendation

| | **AiSensy** | **Gupshup** | **Wati** | **Direct Cloud API** |
|---|---|---|---|---|
| Orientation | Marketing/campaign tool with API | Developer/CPaaS, enterprise (banks, Indian fintechs) | No-code team inbox / SMB CRM | Raw platform |
| Pricing | Subscription (₹1.5k+/mo), **zero markup** | ~₹0.09/msg fee + pass-through, usage minimums | Subscription ($59–279/mo) + **~20% markup** | Pure Meta rates |
| API quality for a backend dispatcher | OK but campaign-shaped (pre-create campaign per template); body-embedded API key | **Best of the three**: proper messaging API, DLR callbacks, high throughput | Weakest: API is an add-on to the inbox, plan-gated quotas | Full control, best webhooks |
| Template management | Good UI + approval assistance | Console + API | Good UI | WhatsApp Manager (Meta's own UI — adequate) |
| Analytics | Strong campaign dashboards | Enterprise reporting | Inbox/agent analytics | DIY (but Yesly has a ledger anyway) |
| Fit for transactional nudges from Celery | Workable, cheap | **Strong** | Poor fit | **Strong** |

**Recommendation for Yesly: go direct with the Meta Cloud API** (own Meta Business + WABA, system-user token), with **Gupshup as the fallback if the team wants managed onboarding/support**.

Reasoning: (1) Yesly's use case is *transactional utility nudges from a backend dispatcher* — exactly what the raw API does well; there's no campaign/inbox requirement that a BSP uniquely solves. (2) Container 5 already implements the ledger + webhook + retry pattern for Expo push, so the marginal engineering cost (~2–3 weeks) buys zero markup, first-party status webhooks, and no BSP lock-in. (3) Template authoring lives in Meta's free WhatsApp Manager — the "build your own template UI" con doesn't apply at Yesly's template count (~10–20). (4) AiSensy's zero-markup deal is attractive but its campaign-shaped API and dashboard-first status reporting fit marketing teams, not a dispatcher; Wati's 20% markup and inbox orientation rule it out. If, later, Yesly adds human advisory chat over WhatsApp (team inbox) or heavy marketing campaigns, add a BSP for *that* workload — the WABA/number can be shared or migrated. Keep marketing volume on push/email given India's per-user caps.

## Sources

- Meta — On-Premises API sunset: https://developers.facebook.com/docs/whatsapp/on-premises/sunset
- Meta — Cloud API get started (test number, 5 recipients): https://developers.facebook.com/docs/whatsapp/cloud-api/get-started/
- Meta — Cloud API messages reference: https://developers.facebook.com/docs/whatsapp/cloud-api/reference/messages
- Meta — Webhooks setup & status reference: https://developers.facebook.com/docs/whatsapp/cloud-api/guides/set-up-webhooks/ ; https://developers.facebook.com/documentation/business-messaging/whatsapp/webhooks/reference/messages/status/
- Meta — Pricing (per-message): https://developers.facebook.com/documentation/business-messaging/whatsapp/pricing
- Meta — Messaging limits: https://developers.facebook.com/documentation/business-messaging/whatsapp/messaging-limits
- Meta — Error codes: https://developers.facebook.com/documentation/business-messaging/whatsapp/support/error-codes
- AiSensy — July 2025 pricing update: https://aisensy.com/tutorials/price-update-2025
- AiSensy — Jan 1, 2026 per-message pricing update (India ₹1.09 / ₹0.145): https://m.aisensy.com/blog/whatsapp-per-message-pricing-update-effective-january-1-2026/
- AiSensy — pricing & API reference: https://aisensy.com/pricing ; https://wiki.aisensy.com/en/articles/11501889-api-reference-docs
- Gupshup — pricing model & template send API: https://support.gupshup.io/hc/en-us/articles/360012075779 ; https://docs.gupshup.io/docs/template-messages ; https://support.gupshup.io/hc/en-us/articles/38821010267673-WhatsApp-Pricing-Updates-2024-2025
- Wati — pricing & API docs: https://www.wati.io/pricing/ ; https://docs.wati.io/reference/introduction ; markup audits: https://www.ycloud.com/blog/wati-pricing ; https://chatarmin.com/en/blog/wati-pricing
- Helo.ai — Jan 2026 pricing update (India rate table): https://helo.ai/resources/blog/whatsapp-api-pricing-update-january-2026
- India marketing caps & deliverability updates: https://www.braze.com/resources/articles/meta-introduces-deliverability-updates-for-whatsapp ; https://support.exotel.com/support/solutions/articles/3000125954 ; https://clevertap.com/blog/navigating-whatsapps-new-marketing-message-limits-a-comprehensive-guide/
- Template rejections: https://m.aisensy.com/blog/whatsapp-template-approval-process/ ; https://www.twilio.com/docs/whatsapp/tutorial/message-template-approvals-statuses
- DPDP/WhatsApp compliance: https://www.complyzero.com/blog/dpdp-whatsapp-business-compliance ; https://www.dpdpa.com/blogs/whatsapp_business_dpdpa_compliance_messaging_apps.html
