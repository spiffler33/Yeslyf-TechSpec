






# 17. Transactional SMS Gateway for India (TRAI/DLT-Compliant)

**Context:** Yesly (SEBI-registered RIA app, React Native/Expo + FastAPI, AWS Mumbai) needs a transactional SMS channel for India — OTPs, trade/advice alerts, payment confirmations. Any A2P SMS to Indian numbers is governed by TRAI's TCCCPR regulations enforced through DLT (Distributed Ledger Technology) blockchain registries run by telcos.

---

## 1. Vendor comparison

| Vendor | ₹/SMS (transactional, India) | API quality | DLT handling | Deliverability | Notes |
|---|---|---|---|---|---|
| **MSG91** | ~₹0.15–0.20 + 18% GST [verify exact slab] | Excellent — clean REST v5, dedicated OTP API with token verify, SDKs, webhooks | Guided DLT onboarding; template sync from DLT portal into panel | Strong direct telco routes; OTP-optimized route | Prepaid wallet (₹1k–50k), no monthly minimum. Widely used by Indian fintechs. |
| **Gupshup** | ~₹0.17 [verify] | Good, but API surface is older/enterprise-flavored; strongest on WhatsApp | Handles DLT scrubbing; enterprise onboarding | Very good (large operator relationships) | Better fit if WhatsApp Business is the primary channel. |
| **Textlocal** (Amazon-owned IMImobile/Webex brand) | ~₹0.20–0.25 [verify] | Simple REST, dated docs, fewer OTP-specific features | Standard DLT support | Good | Product investment has slowed post-acquisition [verify]. |
| **Kaleyra** (now Tata Communications) | ~₹0.18 [verify]; enterprise contracts | Solid enterprise API (io platform) | Full managed DLT support | Excellent (carrier-grade) | Minimum commits / sales-led; overkill for early-stage volume. |

**Recommendation: MSG91.**
- Lowest effective per-SMS cost at Yesly's expected volumes, prepaid with no commitment.
- Purpose-built OTP API (send/verify/resend with server-side token check) — pairs cleanly with FastAPI.
- Best developer experience of the four (docs, Postman collections, sandbox, webhooks).
- DLT template mapping is first-class in their panel: you import your DLT-approved templates and MSG91 validates variable counts before send, reducing silent telco rejections.
- Indian company, INR invoicing, data stays in-country — simpler for DPDP posture than routing PII through a US processor.

---

## 2. TRAI DLT prerequisites (must be done BEFORE any API integration)

1. **Principal Entity (PE) registration** on any one telco DLT portal — they interoperate: Jio TrueConnect, Airtel DLT, Vodafone Idea (VILPower), BSNL/Tata. Needs: company PAN, GST, CIN, authorized-signatory letter, and a one-time fee ~₹5,900 incl. GST [verify per portal]. Output: a 19-digit **Entity ID (PEID)**.
2. **Header (Sender ID) registration** — a 6-char alphanumeric header for transactional/service routes, e.g. `YESLYF`. Registered per category (Transactional / Service-Implicit / Service-Explicit / Promotional). Approval typically 1–3 working days.
3. **Content template registration** — every distinct message body, with variables as `{#var#}` (each var ≤30 chars). Choose category carefully:
   - **Transactional**: strictly bank-OTP-like; effectively reserved for banks. Most businesses use:
   - **Service-Implicit**: OTPs, alerts, confirmations triggered by a customer action — this is Yesly's route for OTP and advisory alerts. Delivered to DND numbers, no time window.
   - **Service-Explicit / Promotional**: marketing; requires recorded consent; promo only 10:00–21:00 IST and only to non-DND numbers.
   Output per template: a 19-digit **DLT Template ID**.
4. **Consent templates** — required only for service-explicit/promotional sends; register the consent-acquisition wording on the portal.
5. **Telemarketer chaining** — add your SMS aggregator (MSG91's registered TM ID) as your telemarketer on the DLT portal so operator **scrubbing** (real-time match of PEID + header + template + content against the ledger) passes. Unmatched messages are dropped by the operator, often silently.

---

## 3. Integration flow (Yesly + MSG91)

1. Complete DLT: PEID → header `YESLYF` → service-implicit templates (OTP, alert, confirmation) → map MSG91 as telemarketer.
2. In MSG91 panel: create account, load wallet, import DLT templates (template text + DLT Template ID), get **authkey**.
3. Backend (FastAPI): store authkey in AWS Secrets Manager (Mumbai); build a thin `sms.py` client — send via Flow API, OTP via OTP API.
4. Register a delivery-report webhook URL (FastAPI endpoint) in MSG91 panel; persist DLR status per message ID.
5. App flow: user action → FastAPI calls MSG91 with template flow_id + variables → MSG91 scrubs against DLT → telco delivers → DLR webhook updates status.
6. Monitor: alert on delivery-rate drops (usually means a template/DLT mismatch or telco filter change).

## 4. Key MSG91 API endpoints

Base: `https://control.msg91.com/api/v5/` — auth via `authkey` header on every request (no OAuth). [verify exact paths against current docs]

| Call | Purpose | Key I/O |
|---|---|---|
| `POST /api/v5/flow/` | Send transactional/service SMS (single or bulk) via a pre-created "Flow" mapped to a DLT template | In: `template_id` (MSG91 flow id), `recipients[{mobiles, var1..}]`, `sender` (=DLT header). Out: request/message id |
| `POST /api/v5/otp?mobile=91XXXXXXXXXX&template_id=...` | Generate + send OTP (MSG91 stores it server-side) | In: mobile, template_id, optional `otp_length`, `otp_expiry`. Out: request id |
| `GET /api/v5/otp/verify?mobile=...&otp=...` | Verify OTP | Out: `{"type":"success"}` or error |
| `GET /api/v5/otp/retry?mobile=...&retrytype=text|voice` | Resend OTP, optionally over voice call | — |
| Delivery webhook (configured in panel) | Push DLRs to your endpoint | Payload: message id, number, status (DELIVRD/FAILED/DND etc.), timestamps [verify payload shape] |
| `GET /api/v5/report/{request_id}` [verify] | Pull delivery report | — |

**Auth model:** single static `authkey` per account (scoped sub-keys available); pass as HTTP header. Rotate via panel; keep in Secrets Manager, never in the RN app — all sends go through FastAPI.

## 5. Pricing (INR)

- Service-implicit/transactional SMS: **~₹0.15–0.20/SMS + 18% GST** [verify current slab vs volume]. Includes the ~₹0.025 DLT scrubbing charge telcos levy [verify].
- Prepaid wallet ₹1,000+; failed and DND-delivered messages are still charged.
- One-time DLT costs: PE reg ~₹5,900; template/header registration free on most portals [verify].
- Voice-OTP retry billed separately (~₹0.30–0.60/call [verify]).

## 6. Pitfalls

- **Template mismatch = silent drop.** Content must match the DLT template character-for-character outside `{#var#}` slots; extra spaces, changed punctuation, or a variable >30 chars gets scrubbed. Test every template on all major operators (Jio/Airtel/Vi behave differently).
- **Header case sensitivity / exact match** — send `YESLYF` exactly as registered; mismatched case or wrong header-category use is rejected.
- **Route misclassification**: OTPs must go on service-implicit (or transactional if eligible), never promotional — promo route strips delivery to DND numbers and is blocked outside 10:00–21:00 IST.
- **Multi-variable templates**: telcos increasingly reject "one giant variable" templates used to smuggle arbitrary content; keep fixed text dominant.
- **DLT portal flakiness**: approvals can take days and portals differ in variable-syntax quirks; register templates on ONE portal only (they replicate).
- **Wallet exhaustion** silently fails sends — set low-balance alerts.
- **DPDP**: phone numbers are personal data; log message IDs, not message bodies with PII, and set DLR retention policy.

**Verdict:** MSG91 on the service-implicit route, with DLT registration started immediately (it is the long pole — 1–2 weeks end-to-end).
