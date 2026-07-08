# 19. Customer Support Platform (Helpdesk / In-App Support)

**Used at:** Help & Support screen in the Yesly app (in-app chat + help center + "raise a ticket"), support@yesly email intake, WhatsApp support channel, and backend webhooks that link support tickets to user accounts.

**Shortlist:** Freshdesk (Freshworks), Zoho Desk, Intercom.
**Recommendation: Freshdesk (with Freshchat SDK for in-app chat).**

---

## 1. Comparison

| Criterion | Freshdesk | Zoho Desk | Intercom |
|---|---|---|---|
| React Native SDK | Freshchat RN SDK (`react-native-freshchat-sdk`) for in-app chat/FAQ; mature, official | ASAP RN SDK exists but historically Android-first; iOS lagged [verify current parity] | `@intercom/intercom-react-native`, official, best-in-class mobile messenger |
| Expo compatibility | Native SDK → needs dev client / prebuild; no official Expo config plugin (write a small one or use prebuild + manual native config) [verify] | Same — native modules, dev client required; no official Expo plugin | Built-in Expo config plugin shipped in the official package; cleanest Expo story |
| Ticketing | Full helpdesk: SLAs, automations, canned responses, CSAT | Full helpdesk, deep Zoho ecosystem, Blueprints for process flows | Ticketing exists but Intercom is conversation-first; classic helpdesk workflows weaker |
| Email support | Native (support@ forwarding, multiple mailboxes) | Native | Supported, but secondary to Messenger |
| WhatsApp channel | Yes — WhatsApp Business API integration (Growth plan and above; Meta conversation fees separate) [verify plan gating] | Yes — Instant Messaging channel incl. WhatsApp [verify plan gating] | Yes, on higher tiers (Advanced/Expert) [verify] |
| India data residency | **Yes — Mumbai (IND) data center**, selectable at signup | **Yes — Zoho India DC (zoho.in, Mumbai/Chennai)** | **No India region.** US / EU / AUS only, and regional hosting only on sales-led Advanced/Expert annual plans |
| DPDP posture | Data can stay in India; Freshworks is an Indian-origin company, DPDP-aware; sign DPA | Data in India on zoho.in; Zoho publicly commits to DPDP compliance; sign DPA | Cross-border transfer to US/EU required → heavier DPDP contractual work (consent notice must disclose transfer) |
| Pricing (INR/agent/mo, annual) | Free (2 agents); Growth ~₹999–1,300; Pro ~₹3,299–4,500; Enterprise ~₹5,299–7,200 [verify on freshworks.com/freshdesk/pricing INR toggle] | Free (3 agents); Express ~₹500 [verify]; Standard ~₹800; Professional ~₹1,400; Enterprise ~₹2,400 [verify on zoho.com/desk/pricing.html INR] | USD only: Essential $29, Advanced $85, Expert $132 /seat/mo annual (~₹2,500 / ₹7,300 / ₹11,300 at ₹86/$) + Fin AI $0.99/resolution [verify] |

### Verdict
**Freshdesk.** It is the only shortlisted option that combines: an India (Mumbai) data center you can pick at signup (clean DPDP story alongside the AWS Mumbai backend), a real ticketing/email helpdesk, WhatsApp as a first-class channel, INR billing, and an official React Native chat SDK (Freshchat) with JWT user authentication. Zoho Desk is the budget runner-up (cheapest, India DC, but the ASAP RN SDK has been Android-first and rougher). Intercom has the best mobile SDK and Expo plugin but no India data residency and pricing that scales badly (per-seat + per-resolution Fin fees, key features gated to Advanced/Expert) — a poor fit for a cost-sensitive, DPDP-focused SEBI RIA.

---

## 2. Integration flow (Freshdesk + Freshchat)

1. **Provision** Freshdesk account on the **IND (Mumbai)** data center at signup (cannot migrate regions later — get this right on day 1). Enable Freshchat (Freshworks Customer Service Suite or Freshchat add-on) [verify bundling].
2. **App-side SDK:** `npm i react-native-freshchat-sdk`. Expo: this is a native module — add `expo-dev-client`, run `npx expo prebuild`, add required native config (Android `google-services.json` for push, iOS APNs). No official Expo config plugin; encapsulate native tweaks in a small custom config plugin so prebuild stays reproducible.
3. **Init:** on app start, `Freshchat.init(new FreshchatConfig(APP_ID, APP_KEY, DOMAIN))` with the IND domain (e.g. `msdk.in.freshchat.com` [verify exact host]).
4. **Authenticated identity (mandatory before production):** enable **User Auth (Strict mode)** in Freshchat. Backend (FastAPI) endpoint `/support/chat-token` mints a **JWT signed with the Freshchat public/private key pair** containing `reference_id` (your internal user_id) and `freshchat_uuid`; app calls `Freshchat.restoreUserWithIdToken(jwt)` (existing user) or `setUserWithIdToken(jwt)` (create/update). Never set user identity from client-supplied values alone — prevents impersonation of another investor's support history.
5. **User properties:** push non-sensitive context (app version, KYC status flag, plan) via `Freshchat.setUserProperties`. Do NOT push PAN, bank details, or portfolio values into the support tool.
6. **Ticketing from backend:** for structured issues (e.g. failed mandate, AA consent failure), FastAPI calls Freshdesk REST `POST /api/v2/tickets` with `requester_id`/`email`, `subject`, `description`, `custom_fields` (yesly_user_id, issue_type), `tags`.
7. **Webhooks:** Freshdesk Automations → webhook to `POST https://api.yesly.in/webhooks/freshdesk` on ticket created/updated/resolved. Validate a shared secret header (Freshdesk webhooks don't sign payloads natively — put a secret in a custom header and check it) [verify current signing options]. Update in-app "My tickets" status.
8. **Channels:** connect support@yesly.in mailbox; connect WhatsApp Business number via Freshdesk WhatsApp integration (reuses the Meta BSP setup from doc 06). Publish FAQ articles to the Freshdesk knowledge base; surface them in-app via Freshchat FAQs.

---

## 3. Key API endpoints (Freshdesk REST v2 — `https://<domain>.freshdesk.com/api/v2`)

| Call | Purpose | Key I/O |
|---|---|---|
| `POST /tickets` | Create ticket from backend | In: email/requester_id, subject, description, status(2=Open), priority, custom_fields, tags → Out: ticket id |
| `GET /tickets/{id}` (+`?include=conversations`) | Fetch ticket + thread for in-app "My tickets" | Out: status, conversations[] |
| `PUT /tickets/{id}` | Update status/fields | In: status, custom_fields |
| `POST /tickets/{id}/reply` | Post reply on behalf of support flows | In: body |
| `GET /tickets?email=...` or `/search/tickets?query=` | List a user's tickets | Out: paginated tickets |
| `POST /contacts` / `GET /contacts?email=` | Create/find contact mapped to Yesly user | In: name, email, unique_external_id=user_id |
| `GET /solutions/articles/{id}` etc. | Pull KB articles (optional, Freshchat FAQ usually covers) | Out: article HTML |
| Freshchat: `POST /support/chat-token` (your FastAPI) | Mint Freshchat user JWT | In: session auth → Out: signed JWT (reference_id, freshchat_uuid) |
| Webhook (inbound) `POST /webhooks/freshdesk` | Ticket lifecycle events → app state | In: ticket id/status; verify shared-secret header |

Rate limits: plan-dependent, ~50–700 req/min [verify]; handle 429 with `Retry-After`.

---

## 4. Auth model

- **Freshdesk REST API:** HTTP Basic auth with an agent **API key** as username (`api_key:X`). Key inherits that agent's permissions — create a dedicated "API bot" agent, store the key in AWS Secrets Manager, backend-only, never in the app.
- **Freshchat SDK:** App ID + App Key identify the app (public-ish); user identity secured by **JWT (RS256) signed server-side** with keys from Freshchat's User Auth settings; strict mode rejects unauthenticated users.
- **Webhooks:** no native HMAC signature [verify] — enforce shared-secret header + HTTPS + IP allowlist if available.
- (Contrast: Intercom uses HMAC-SHA256 user-hash / JWT "identity verification"; Zoho Desk API uses OAuth2 with Zoho Accounts on the .in DC.)

---

## 5. Pitfalls

- **Pick the IND data center at account creation.** Freshworks does not let IND accounts migrate regions later; a US-region account undermines the DPDP story.
- **Identity verification before production.** Without Freshchat strict-mode JWT auth (or Intercom's user hash), anyone with the app bundle can impersonate any user_id and read their support conversations. Treat as a launch blocker.
- **Expo:** Freshchat and Zoho ASAP native SDKs do not work in Expo Go — you need `expo-dev-client` + prebuild, and CI (EAS Build) must include the native config. Intercom is the only one with an official Expo config plugin.
- **Intercom cost scaling:** per-seat pricing + $0.99/resolution Fin charges + feature gating (regional hosting, WhatsApp on Advanced/Expert only) makes cost unpredictable as ticket volume grows; and still no India region.
- **Zoho ASAP RN SDK maturity:** historically Android-only with iOS "under development" — verify current iOS parity before committing [verify].
- **WhatsApp fees are separate:** Meta per-conversation charges bill through your BSP regardless of helpdesk choice (see doc 06); the helpdesk only provides the agent inbox.
- **Data minimization (DPDP/SEBI):** keep PII in support tools to name/email/phone + ticket text; never sync PAN, bank, or holdings. Add Freshdesk to your consent notice as a processor and sign the DPA.
- **Pricing drift:** all INR figures above are indicative from third-party 2026 roundups — confirm on the official INR pricing pages before the client signs. [verify]

Sources: freshworks.com/freshdesk/pricing, support.freshdesk.com (data centers), developers.freshchat.com (RN SDK, JWT auth), developers.freshdesk.com (API v2), zoho.com/desk/pricing.html, help.zoho.com (ASAP RN SDK), intercom.com/pricing, developers.intercom.com (RN installation, identity verification, regional hosting article 6124430).
