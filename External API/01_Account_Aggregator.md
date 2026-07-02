# Account Aggregator (AA) Framework & Vendor APIs

> Research date: July 2026. Covers the ReBIT/RBI AA framework, a Setu deep-dive (front-runner), vendor comparison, compliance for a SEBI-registered RIA, and integration gotchas.

---

## 1. Overview & role in Yesly

Yesly (House of Alpha, SEBI RIA reg. INA000018364) uses the Account Aggregator framework at **Step 22 of the onboarding flow, gated to Tier 2+ users**. The in-app experience is a ~7-step consent journey:

1. Pick FIPs (banks / MF / demat / insurance providers to link)
2. Enter mobile number (must be the number registered with those FIPs)
3. Backend creates a consent → gets a consent handle/ID from the AA
4. User verifies OTP with the AA (creates/logs into their AA handle, e.g. `98xxxxxx01@setu`)
5. Review consent details (purpose, data types, date range, expiry, frequency)
6. Approve (or reject)
7. Done → redirect back into the app

After approval, the backend performs an **initial data fetch** (create data session → fetch decrypted FI data → normalise → store in RDS Postgres), then runs **scheduled refreshes that reuse the same ACTIVE consent** (a `PERIODIC` consent lets you open new data sessions without re-running the 7-step UX, within the consented frequency), and handles **revoke / pause / expiry** events pushed via webhook (stop refreshing, purge data per data-life rules, prompt re-consent).

Why AA instead of screen-scraping or statement upload: it is the RBI-regulated, consent-based rail; data arrives digitally signed from the source FIP; and as a SEBI-regulated entity Yesly is eligible to be an FIU (see Compliance).

---

## 2. How the AA framework works (ReBIT spec)

### Roles

| Role | Who | Regulator |
|---|---|---|
| **AA (NBFC-AA)** | Consent manager + data pipe. Licensed NBFC-AA under RBI Master Direction DNBR.PD.009/03.10.119/2016-17 (Sep 2, 2016). Is "data-blind": transports encrypted data, cannot read or store it. | RBI |
| **FIP** (Financial Information Provider) | Banks, NBFCs, AMCs/RTAs, depositories, insurers, GSTN, CRA (NPS) — the data sources | RBI / SEBI / IRDAI / PFRDA |
| **FIU** (Financial Information User) | Any entity "registered with and regulated by any financial sector regulator" that consumes data. **Yesly/House of Alpha qualifies as a SEBI-registered RIA.** | RBI / SEBI / IRDAI / PFRDA |
| **TSP** (Technical Service Provider) | Unregulated tech vendors (e.g. Setu, Finarkein) that run the FIU module for a regulated entity | — (contracted by FIU) |
| **Sahamati** | Industry alliance: central registry, FIU/FIP certification, fair-use consent templates, FIP health dashboards | Self-regulatory body |

As of 2026 the ecosystem has ~17 operational AAs, ~179 FIPs and 950+ FIUs live (Sahamati directory, via CASParser's 2026 state-of-AA report).

### Consent artefact structure

The consent artefact is a digitally signed JSON object (signed by the AA, verifiable by the FIP) with these key fields (ReBIT AA API spec v2.0.0):

- **Purpose** — `code` + `refUri` + `text` + `Category`. Standard codes: **101** Wealth management service, **102** Customer spending patterns / budget / reporting, **103** Aggregated statement, **104** Explicit consent to monitor accounts, **105** Explicit one-time consent for the data. *Yesly's fit: 101 (periodic, wealth management) — this is also the code Sahamati's fair-use templates prescribe for advisory/wealth use.*
- **fetchType** — `ONETIME` or `PERIODIC`
- **Frequency** — `unit` (HOUR/DAY/MONTH/YEAR) + `value`; caps how often FI can be fetched (e.g. once per day)
- **FIDataRange** — `from`/`to` timestamps bounding the transaction history requested
- **consentExpiry / consentStart** — validity window of the consent itself
- **DataLife** — how long the FIU may retain/use fetched data (`unit` DAY/MONTH/YEAR/INF + `value`)
- **consentMode** — `VIEW` / `STORE` / `QUERY` / `STREAM` (STORE = FIU may persist data, subject to DataLife)
- **consentTypes** — `PROFILE`, `SUMMARY`, `TRANSACTIONS`
- **fiTypes** — e.g. `DEPOSIT`, `TERM_DEPOSIT`, `RECURRING_DEPOSIT`, `MUTUAL_FUNDS`, `EQUITIES`, `ETF`, `SIP`, `BONDS`, `DEBENTURES`, `GOVT_SECURITIES`, `INSURANCE_POLICIES`, `NPS`, `GSTR1_3B` … (~23 FI types defined by ReBIT FI schemas)

### Full lifecycle (text flow diagram)

```
CONSENT FLOW
FIU backend                AA                         FIP                    User (app/webview)
    |  POST /Consent  ---->  |                          |                        |
    |  <--- ConsentHandle    |                          |                        |
    |  (status PENDING)      |                          |                        |
    |                        |  <-- OTP login, account discovery & linking ----  |
    |                        |  (AA discovers accounts at FIPs via mobile no.)   |
    |                        |  <------- user reviews & APPROVES consent ------  |
    |                        |-- signed consent artefact stored; notifies FIP    |
    |  <--- ConsentStatusNotification (ACTIVE) + consentId                       |
    |  POST /Consent/fetch ->|  (returns signed consent artefact JSON)           |

DATA FLOW (repeatable while consent ACTIVE, within Frequency)
    |  generate ephemeral Curve25519 key pair + 32-byte nonce                    |
    |  POST /FI/request ----> |  --- forwards consent + FIU KeyMaterial ------>  |
    |  <--- sessionId         |                          | FIP verifies consent  |
    |                        |                           | signature, prepares   |
    |                        |  <-- encrypted FI data -- | data, encrypts with   |
    |  <--- FINotification (session READY) ---           | ECDH shared secret    |
    |  GET /FI/fetch/{sessionId} -> AA returns encrypted payload + FIP KeyMaterial
    |  FIU: ECDH(own private key, FIP public key) -> shared secret               |
    |       HKDF(shared secret, nonces) -> 256-bit session key                   |
    |       AES-256-GCM decrypt -> FI XML/JSON per ReBIT FI schema for each FIType

LIFECYCLE EVENTS
    User may PAUSE / REVOKE consent at the AA at any time; consent auto-EXPIREs
    at consentExpiry. AA pushes ConsentStatusNotification to FIU; FIU must stop
    fetching and honour DataLife for already-fetched data.
```

Security plumbing per ReBIT: every API body is signed with a **detached JWS** sent in the **`x-jws-signature`** header (both requests and responses); data encryption is **ECDHE on Curve25519 + HKDF + AES-256-GCM** (reference implementation: the "Rahasya" library / Forward-Secrecy DHE). ([ReBIT AA API spec v2.0.0](https://specifications.rebit.org.in/artefacts/NBFC-AA_API_Specification_v2.0.0.pdf), [api.rebit.org.in](https://api.rebit.org.in/), [Finvu JWS guide](https://github.com/finvu/finvu-rebit-aa-jws))

Notifications in raw ReBIT terms: the FIU must expose `POST /Consent/Notification` (ConsentStatusNotification: `consentId`, `consentHandle`, `consentStatus` = ACTIVE/PAUSED/REVOKED/EXPIRED) and `POST /FI/Notification` (FINotification: `sessionId`, `sessionStatus` = READY/EXPIRED/FAILED). **A TSP like Setu absorbs all of this** — including JWS signing and payload decryption — and re-exposes simple REST + webhooks.

---

## 3. Setu AA deep-dive (front-runner)

Setu (Pine Labs group) operates both an NBFC-AA and an FIU TSP module. Yesly would integrate Setu's **FIU APIs**: Setu handles ReBIT signing, encryption/decryption, multi-AA routing (`@setu`, `@onemoney`, …), and gives back plain decrypted JSON. Docs: [docs.setu.co/data/account-aggregator](https://docs.setu.co/data/account-aggregator/overview).

**Base URLs** — Sandbox: `https://fiu-sandbox.setu.co` · Production: `https://fiu.setu.co`

### Endpoints table

| Method & path | Purpose | Key request fields | Key response fields |
|---|---|---|---|
| `POST /consents` | Create consent (start of Step 22 flow) | `consentDuration {unit, value}`, `vua` (e.g. `9999999999@setu`), `dataRange {from, to}` (mandatory when consentTypes includes TRANSACTIONS), `consentMode`, `fetchType`, `consentTypes[]`, `fiTypes[]`, `purpose` (code 101–105), `frequency {unit, value}`, `dataLife {unit, value}`, `redirectUrl`, `context[]`, `additionalParams.tags[]` | `id` (consent UUID), `url` (hosted AA webview for OTP + review + approve), `status: PENDING`, `detail.*` echo |
| `GET /consents/:id` | Poll consent status | — | `status` (PENDING → ACTIVE / REJECTED / REVOKED / PAUSED / EXPIRED), `detail.consentExpiry`, `usage {lastUsed, count}` |
| `POST /sessions` | Create data session (initial fetch & every scheduled refresh — reuses the same `consentId`) | `consentId`, `dataRange {from, to}` (⊆ consent's FIDataRange), `format: "json"` or `"xml"` | `id` (session UUID), `status: PENDING` |
| `GET /sessions/:id` | Fetch FI data once session ready | — | session `status` (PENDING / PARTIAL / COMPLETED / EXPIRED / FAILED); `fips[] → accounts[] → {linkRefNumber, maskedAccNumber, status (READY/DELIVERED/PENDING/TIMEOUT/DENIED), data.account {profile, summary, transactions}}` — **already decrypted** |

There is also an optional **auto-fetch** mode: Setu automatically creates the session on consent approval and pushes the decrypted data to your webhook as an `FI_DATA_READY` event with an `fiData` array — useful for the initial fetch after Step 22 approval.

### Example: create consent (Setu docs)

```json
POST https://fiu-sandbox.setu.co/consents
{
  "consentDuration": { "unit": "MONTH", "value": "24" },
  "vua": "9999999999@setu",
  "dataRange": { "from": "2024-04-01T00:00:00Z", "to": "2026-07-01T00:00:00Z" },
  "consentMode": "STORE",
  "fetchType": "PERIODIC",
  "consentTypes": ["PROFILE", "SUMMARY", "TRANSACTIONS"],
  "fiTypes": ["DEPOSIT", "MUTUAL_FUNDS", "EQUITIES", "INSURANCE_POLICIES", "NPS"],
  "purpose": {
    "code": "101",
    "refUri": "https://api.rebit.org.in/aa/purpose/101.xml",
    "text": "Wealth management service",
    "category": { "type": "string" }
  },
  "frequency": { "unit": "DAY", "value": 1 },
  "dataLife": { "unit": "MONTH", "value": 12 },
  "redirectUrl": "https://app.yesly.in/aa/callback",
  "context": []
}
```

```json
201 Created
{
  "id": "6d285134-c764-49ab-b32d-ead003161587",
  "url": "https://fiu.setu.co/v2/consents/webview/6d285134-c764-49ab-b32d-ead003161587",
  "status": "PENDING",
  "detail": {
    "consentExpiry": "2028-07-01T05:36:43.011Z",
    "fiTypes": ["DEPOSIT", "MUTUAL_FUNDS", "EQUITIES", "INSURANCE_POLICIES", "NPS"],
    "consentTypes": ["TRANSACTIONS", "PROFILE", "SUMMARY"]
  }
}
```

The React Native app opens `url` in a webview/browser (steps 4–6 of Step 22: OTP, review, approve), then the AA redirects to `redirectUrl`.

*(Field names/values above are from Setu's consent-flow and consent-object docs; the exact combination is illustrative for Yesly's use case.)*

### Example: fetch data

```json
GET /sessions/f608b1c5-60cb-445d-8355-76a9cc053535
{
  "status": "COMPLETED",
  "fips": [{
    "fipID": "Setu-FIP",
    "accounts": [{
      "linkRefNumber": "b2329f47-0a6f-4131-adb5-9ef7b4c1ca6a",
      "maskedAccNumber": "XXXXXX4373",
      "status": "DELIVERED",
      "data": { "account": {
        "profile": { "...holder name, mobile, PAN (as provided by FIP)..." },
        "summary": { "...balance, type, ifsc..." },
        "transactions": { "transaction": [
          { "amount": 2500, "type": "DEBIT",
            "transactionTimestamp": "2026-06-14T10:03:12Z",
            "narration": "UPI/..." }
        ]}
      }}
    }]
  }]
}
```

Non-DEPOSIT FI types (MUTUAL_FUNDS, EQUITIES, INSURANCE_POLICIES, NPS) return the same envelope with type-specific `summary`/`transactions` blocks following the ReBIT FI schemas (holdings with ISIN/folio, units, NAV, etc.).

### Auth (Setu)

- Onboard via **The Bridge** dashboard ([bridge.setu.co](https://bridge.setu.co/v2/signup)): register the FIU, create an "Account Aggregator" Data product, configure consent template, branding, discovery methods and notification endpoint.
- Credentials: **`x-client-id` + `x-client-secret`** (issued per environment) exchanged for a Bearer `access_token` (OAuth2 client-credentials style), plus **`x-product-instance-id`** header on every call identifying the configured AA product.
- Setu handles the ReBIT-level **detached JWS `x-jws-signature`** signing and verification and the ECDH key management internally — Yesly's FastAPI backend never touches Curve25519 keys. (If you ever go AA-direct without a TSP, you own JWS + Rahasya-style decryption yourself.)

### Webhooks (Setu)

Configure one webhook `base_url` in Bridge. Two event families (JSON POSTs):

- **`CONSENT_STATUS_UPDATE`** — `{ "consentId": "...", "type": "CONSENT_STATUS_UPDATE", "data": { "status": "ACTIVE" }, "timestamp": "...", "success": true }`. Drives Step 22 completion and later REVOKED/PAUSED/EXPIRED handling.
- **`SESSION_STATUS_UPDATE`** — `{ "dataSessionId": "...", "consentId": "...", "type": "SESSION_STATUS_UPDATE", "data": { "status": "COMPLETED" } }`. Signals "data ready, call GET /sessions/:id".
- With auto-fetch on: **`FI_DATA_READY`** carrying the decrypted `fiData` array directly.

⚠️ As of the current docs, **Setu documents no webhook signature verification and no retries** ("Webhook retries will be added in future iterations"). Mitigation: treat webhooks as hints; run a reconciliation poller (`GET /consents/:id`, `GET /sessions/:id`) and idempotent handlers; restrict the webhook endpoint (allowlist/secret path).

### Sandbox / testing (Setu)

- Full sandbox at `fiu-sandbox.setu.co`, driven from the Bridge dashboard; free to use during development.
- **Mock FIPs** (e.g. `Setu-FIP`) with **mock data for all ~23 FI data types**; test VUAs like `9999999999@setu` simulate discovery/linking/OTP without real bank accounts — this is how Yesly QA tests Step 22 end-to-end.
- Docs suggest Beeceptor for quick webhook testing; a Postman collection for the FIU APIs is published ([Setu FIU APIs Postman](https://documenter.getpostman.com/view/16080598/TzzBoun5)).
- Production go-live requires Setu review of your consent template + FIU onboarding compliance (regulated-entity checks), after which prod credentials and a dedicated sub-domain are issued.

### Pricing (Setu)

- **No comprehensive public rate card.** Sandbox is free; production is a sales-led contract.
- Third-party reviews (productgrowth.in, 2026) report **indicative ₹10–₹25 per successful data fetch, with volume discounts**; the market norm across AAs/TSPs is per-successful-fetch (sometimes split consent-fee + fetch-fee). Treat any number as unverified until quoted by Setu — AA↔FIU pricing is bilateral; Sahamati does not fix prices.
- Budget note for Yesly: a PERIODIC daily-refresh consent at ~₹10–25/fetch would be ~₹300–750/user/month if fetched daily — argues for weekly/monthly refresh cadence or fetch-on-app-open throttling.

---

## 4. Vendor comparison

| | **Setu AA (+TSP)** | **Anumati (Perfios)** | **OneMoney (+MoneyOne TSP)** | **CAMSFinServ** | **Finvu (Cookiejar)** |
|---|---|---|---|---|---|
| Licence | NBFC-AA + FIU TSP (Pine Labs) | NBFC-AA (Perfios) | First NBFC-AA licensee; MoneyOne = TSP arm | NBFC-AA (CAMS group) | NBFC-AA pure-play |
| FIP coverage (early-2026, reported) | Routes to multiple AAs incl. `@onemoney`; own FIP network smaller | **~80+ FIPs — broadest**; strong banking + GST + depositories | ~65+ FIPs; long production track record | ~70+ FIPs; **deepest MF coverage** (CAMS is largest MF RTA) | ~60+ FIPs |
| Integration style | Clean REST + hosted consent webview, decrypted JSON, webhooks; best-documented public docs; Bridge self-serve sandbox | Perfios-style enterprise API; often bundled with Perfios analytics/BSA; sales-led | ReBIT-flavoured FinPro FIU APIs (docs.moneyone.in); enterprise onboarding | ReBIT-flavoured APIs; sales-led; docs not fully public | ReBIT-style APIs; public sandbox on GitHub Pages (finvu.github.io/sandbox); helper libs (JWS repo) |
| SDKs | Web/webview redirect flow (works in RN via browser/webview); Postman collection | Mobile SDKs + web redirect via Perfios | Mobile SDK + web (`sdk_consent` APIs documented) | Web redirect; SDK availability unclear publicly | Web redirect + client libs; JWS reference code |
| Pricing | Per-successful-fetch, ~₹10–25 reported (unofficial) | Not public; enterprise contract | Not public; enterprise contract | Not public | Not public |
| Strengths | **Developer experience, sandbox with mock FIPs for 23 FI types, decryption handled, multi-AA routing** | Coverage breadth; analytics stack (Perfios BSA) on top | Maturity, reliability at scale, dual AA+TSP | Mutual-fund data depth — relevant to a wealth app | Focused AA partner, transparent tech, good engineering docs |
| Weaknesses | Own-AA FIP coverage narrower than Anumati/CAMS (mitigated by multi-AA routing); webhooks lack retries/signatures | Docs behind sales wall; heavier enterprise motion | Docs partly behind onboarding | Least self-serve; docs opacity | Smaller coverage; you do more ReBIT-level work yourself |

(Coverage figures are third-party reported — [HyperVerge FIU selection guide](https://hyperverge.co/blog/best-account-aggregators/), [CASParser state-of-AA 2026](https://casparser.in/blog/state-of-account-aggregator-2026/) — and shift monthly; check [Sahamati FIP-AA mapping](https://sahamati.org.in/fip-aa-mapping/) before committing.)

**Recommendation for Yesly:** Setu as primary (TSP + AA with multi-AA routing gives coverage without multi-vendor integration), evaluate CAMSFinServ as a secondary if MF-holdings coverage via Setu proves thin. A multi-AA strategy through one TSP beats integrating two raw AAs.

---

## 5. Compliance notes (India)

- **RBI Master Direction NBFC-AA (DNBR.PD.009/03.10.119/2016-17, Sep 2, 2016, updated)** — the constitutive regulation: AA licensing, the FIU definition ("entity registered with and regulated by any financial sector regulator"), explicit-consent requirement, AA data-blindness. [RBI Master Direction](https://www.rbi.org.in/Scripts/BS_ViewMasDirections.aspx?id=10598)
- **Yesly's FIU eligibility:** House of Alpha is SEBI-registered (RIA INA000018364) → qualifies as an FIU under the RBI definition. Practical onboarding: sign Sahamati ecosystem terms, get the FIU module **certified** (Sahamati-empanelled certifiers) and listed in the **Central Registry** — Setu as TSP does the technical heavy lifting, but the FIU registration and regulatory responsibility sit with House of Alpha, not Setu.
- **SEBI circular SEBI/HO/MRD/DCAP/P/CIR/2022/110 (Aug 19, 2022)** — "Participation as Financial Information Providers in Account Aggregator framework": brought SEBI-regulated FIPs into AA (depositories; AMCs via their RTAs), which is what makes `MUTUAL_FUNDS` and `EQUITIES` (demat) FI types actually flow. Secondary summaries of the circular/consultation note the FIU-side use cases SEBI envisaged: *an RIA seeking client financial-asset information via AA to devise a financial plan; a portfolio manager seeking portfolio information to manage the client's portfolio* — exactly Yesly's use case. (Circular text: [sebi.gov.in](https://www.sebi.gov.in/legal/circulars/aug-2022/participation-as-financial-information-providers-in-account-aggregator-framework_62157.html); FIU use-case language per [SCC Online summary](https://www.scconline.com/blog/post/2022/08/20/sebi-issues-circular-on-participation-of-financial-account-aggregators-in-account-aggregator-framework/) and TaxGuru's consultation-paper coverage — verify against the circular PDF before citing in compliance docs.)
- **Can Yesly store fetched data?** Yes, conditionally: request `consentMode: STORE`, and retention is bounded by the consent's **DataLife** (e.g. 12 months) and by **purpose limitation** (code 101 wealth management — data usable only for that). On consent REVOKED/EXPIRED you may retain already-fetched data until DataLife lapses but must not fetch again; on DataLife expiry, purge from RDS/S3. Avoid `DataLife: INF` — Sahamati fair-use templates and DPDP data-minimisation both cut against it.
- **DPDP Act 2023 + DPDP Rules:** Yesly is a Data Fiduciary for fetched FI data — needs notice, purpose limitation, storage limitation, erasure on purpose exhaustion, grievance mechanism; penalties up to ₹250 crore for fiduciary breaches (₹50 crore band for consent-manager violations). There is an unresolved overlap between RBI's AA-consent regime and DPDP's Consent Manager regime (AAs are the blueprint but registration/primacy questions remain open — see [Sahamati's reconciliation note](https://sahamati.org.in/reconciling-the-account-aggregator-and-consent-manager-frameworks/), [SCC Online, Jun 2026](https://www.scconline.com/blog/post/2026/06/26/account-aggregator-consent-manager-paradox-dpdp-rules-fintech-sector/)). Practically: AA consent artefact ≠ full DPDP compliance; keep your own DPDP notice + consent records alongside.
- **Sahamati fair-use templates:** consent parameters (frequency, duration, data range) are expected to match the published template for your purpose code; over-asking (e.g. hourly fetch for advisory) can fail certification review. [Fair Use Template Library](https://sahamati.org.in/aa-consent-template-library/)
- Data residency: FI data lands in Yesly's AWS Mumbai (RDS/S3) — consistent with Indian financial-data localisation expectations; encrypt at rest and scope access.

---

## 6. Gotchas & effort estimate

### Consent-handle lifecycle

- ReBIT two-phase reality: `POST /Consent` returns a **ConsentHandle** (its own status: PENDING → READY/FAILED); only after user approval does a **consentId** + signed artefact exist. Setu collapses this into one consent `id`, with statuses **PENDING → ACTIVE / REJECTED**, then **PAUSED / REVOKED / EXPIRED** (+ session-level FAILED). Model all of these in Postgres; PAUSED is user-initiated at the AA and resumable — don't treat it as REVOKED.
- Scheduled refresh must check consent status *before* creating a session, respect the consented `frequency` (429-style rejections otherwise), and keep `dataRange` within the consent's FIDataRange.

### Known ecosystem pain points

- **Mobile-number coupling (biggest UX risk):** AA discovery keys off the mobile number registered at the FIP. If the user's Yesly/Cognito number ≠ bank-registered number → zero accounts discovered. Also: two accounts under one number at the same FIP sometimes surface only one (documented by Fold for current+savings combos). Design Step 22 to ask "the mobile number your bank has", not just reuse the login number.
- **FIP downtime & flakiness:** highly variable, worst among PSU banks; Fold publicly reported ~95% discovery-failure rates at one large private bank for months. Sahamati publishes FIP health dashboards ([Saans API health dashboard](https://sahamati.org.in/saans-api-health-dashboard/), [FIP-wise health](https://sahamati.org.in/dashboard-3-fip-wise-health-summary/)); SLA working group targets 95–99% success, <5% missed notifications — reality is spikier. Build retries with backoff, PARTIAL-session handling (some accounts DELIVERED, others TIMEOUT), and user-visible "bank X is down, we'll retry" states.
- **Data quality variance:** narration formats, missing/incorrect timestamps, duplicate transactions, balance-vs-transaction inconsistencies differ per FIP; MF/equities payloads differ in depth by RTA/depository. Budget for a normalisation + dedupe layer before anything touches Yesly's planning engine.
- **Decryption failures:** if you ever bypass the TSP, ECDH/HKDF/AES-GCM mistakes (nonce misuse, key-material serialisation) are a classic failure class — a key reason to let Setu deliver decrypted JSON.
- **Webhook fragility (Setu-specific):** no retries, no documented signatures → poll as backstop, idempotency keys on handlers.
- **Testing:** sandbox mock FIPs never reproduce production FIP flakiness or real data messiness. Plan a closed beta with employees' real accounts before Tier-2 rollout.
- **Consent UX drop-off:** the 7-step flow (new AA handle + OTP + review) has material abandonment industry-wide; pre-select FIPs, explain the AA webview hand-off in-app, and handle the redirect-back (deep link into the Expo app) carefully on both iOS and Android.

### Effort estimate (Yesly, FastAPI + RN/Expo, via Setu)

| Workstream | Estimate |
|---|---|
| Setu onboarding, Bridge setup, consent template + FIU registration/certification paperwork | 1–2 weeks elapsed (mostly waiting) |
| Backend: consent create/status, webview hand-off + deep-link return, webhook receiver + poller | 1.5–2 weeks |
| Data sessions: initial fetch, scheduler for periodic refresh, lifecycle (revoke/pause/expiry) handling | 1–1.5 weeks |
| FI data normalisation into Yesly's portfolio model (DEPOSIT + MF + EQUITIES first; INSURANCE/NPS later) | 2–3 weeks (the long pole) |
| QA on sandbox + real-account beta + FIP-failure hardening | 1–2 weeks |
| **Total** | **~6–9 engineer-weeks** for a production-worthy v1 via Setu; going AA-direct (raw ReBIT: JWS, ECDH, multi-AA) would be 3–4× that. |

---

## 7. Sources

- [ReBIT — NBFC-AA API Specification v2.0.0 (PDF)](https://specifications.rebit.org.in/artefacts/NBFC-AA_API_Specification_v2.0.0.pdf) · [ReBIT API portal & FI schemas](https://api.rebit.org.in/) · [ReBIT release notes (ECDHE/Curve25519)](https://api.rebit.org.in/assets/ReleaseNotes.txt)
- [RBI — Master Direction NBFC-Account Aggregator (Reserve Bank) Directions, 2016](https://www.rbi.org.in/Scripts/BS_ViewMasDirections.aspx?id=10598)
- [Setu Docs — AA overview](https://docs.setu.co/data/account-aggregator/overview) · [Quickstart](https://docs.setu.co/data/account-aggregator/quickstart) · [Consent flow](https://docs.setu.co/data/account-aggregator/api-integration/consent-flow) · [Consent object](https://docs.setu.co/data/account-aggregator/consent-object) · [Data APIs](https://docs.setu.co/data/account-aggregator/api-integration/data-apis) · [Notifications](https://docs.setu.co/data/account-aggregator/api-integration/notifications) · [FI data types](https://docs.setu.co/data/account-aggregator/fi-data-types) · [Setu FIU APIs Postman collection](https://documenter.getpostman.com/view/16080598/TzzBoun5)
- [SEBI — Circular SEBI/HO/MRD/DCAP/P/CIR/2022/110: Participation as FIPs in AA framework (19 Aug 2022)](https://www.sebi.gov.in/legal/circulars/aug-2022/participation-as-financial-information-providers-in-account-aggregator-framework_62157.html) · [SCC Online summary](https://www.scconline.com/blog/post/2022/08/20/sebi-issues-circular-on-participation-of-financial-account-aggregators-in-account-aggregator-framework/) · [TaxGuru — SEBI AA consultation paper coverage](https://taxguru.in/sebi/sebi-consultation-paper-account-aggregator-framework-public-comments.html)
- [Sahamati — What is a consent artefact](https://sahamati.org.in/what-is-consent-artefact/) · [Fair Use Template Library](https://sahamati.org.in/aa-consent-template-library/) · [Periodic-consent example artefact](https://sahamati.org.in/consent-artefact-periodic-consent-for-monitoring-of-loan-account/) · [FIP-AA mapping](https://sahamati.org.in/fip-aa-mapping/) · [Saans FIP API health dashboard](https://sahamati.org.in/saans-api-health-dashboard/) · [FIP-wise health summary](https://sahamati.org.in/dashboard-3-fip-wise-health-summary/) · [AA SLA working group 2023](https://sahamati.org.in/aa-sla-working-group-recommendation-2023/) · [Reconciling AA and DPDP Consent Manager frameworks](https://sahamati.org.in/reconciling-the-account-aggregator-and-consent-manager-frameworks/) · [How to join the AA network](https://sahamati.org.in/how-to-join-the-account-aggregator-network-to-share-and-access-financial-data/)
- [Finvu — ReBIT JWS reference implementation](https://github.com/finvu/finvu-rebit-aa-jws) · [Finvu sandbox docs](https://finvu.github.io/sandbox/)
- [MoneyOne — FinPro FIU / FinShare FIP tech docs](https://docs.moneyone.in/tech/) · [OneMoney consent SDK API](https://www.onemoney.in/docs/api/sdk_consent.html)
- [Anumati by Perfios](https://www.anumati.co.in/)
- [HyperVerge — FIU selection guide / AA comparison](https://hyperverge.co/blog/best-account-aggregators/) · [CASParser — State of AA 2026](https://casparser.in/blog/state-of-account-aggregator-2026/) · [productgrowth.in — Setu review (indicative pricing)](https://productgrowth.in/tools/banking-api/setu/)
- [Fold — AA challenges & path forward (FIP reliability)](https://fold.money/blog/account-aggregator-challenges-path-forward) · [Fold — Known bank issues](https://help.fold.money/hc/en-us/articles/17395308835986-Known-Issues-Banks)
- [SCC Online — AA / Consent-Manager paradox under DPDP Rules (Jun 2026)](https://www.scconline.com/blog/post/2026/06/26/account-aggregator-consent-manager-paradox-dpdp-rules-fintech-sector/) · [AZB Partners — Consent Managers under DPDP](https://www.azbpartners.com/bank/consent-managers-under-indias-dpdp-act-and-dpdp-rules/)
