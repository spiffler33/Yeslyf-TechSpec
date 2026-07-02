# KYC + eSign — Digital KYC & Advisory-Agreement Signing (Digio front-runner)

> Research date: July 2026. Yesly = SEBI-registered RIA (House of Alpha, INA000018364). Stack: React Native/Expo app, Python FastAPI backend, AWS Mumbai.
> Facts marked **[verify]** could not be confirmed against a primary source during research (Digio's docs site is JS-rendered and not fully crawlable) — confirm against the live docs / vendor sales call before build.

---

## 1. Overview & role in Yesly (Step 21)

Step 21 fires when a user buys the **Tier 2 Investment Plan**: before Yesly can render personalised investment advice and charge an advisory fee, SEBI's IA Regulations require (a) completed client KYC and (b) a **signed investment advisory agreement** (Reg. 19(1)(d), inserted by the IA (Amendment) Regulations 2020: *"neither any investment advice is rendered nor any fee is charged until the client has signed the agreement and been given a copy"*).

The flow: **PAN → Aadhaar (OTP / DigiLocker) → bank penny-drop → selfie/liveness → Aadhaar-eSign the advisory agreement**. Three consecutive failures at any stage route to the grievance/manual-review flow. Verified documents are stored field-encrypted in S3 (Mumbai).

**Open question — is KYC required at Tier 1?** The regulatory trigger is not "tier", it is *investment advice for consideration*. Under the IA Regulations + PMLA (RIAs are SEBI intermediaries/reporting entities), KYC + signed agreement are required **before any personalised investment advice is given or any advisory fee charged** — regardless of price point. If Tier 1 is generic financial education / model content with no personalised recommendation and no advisory fee, KYC is not triggered. If Tier 1 contains any client-specific advice (risk-profiled recommendations count), KYC + agreement are required at Tier 1 too. **This is an interpretation — route to compliance counsel; the safe line is: KYC gate wherever risk-profiling → recommendation happens.**

One vendor can cover the whole step: **Digio** does PAN verification, DigiLocker/offline-Aadhaar KYC, penny-drop, selfie/face-match, *and* Aadhaar eSign (plus KRA fetch/upload) behind one contract and one auth scheme. That is the main reason it is the front-runner.

---

## 2. How the KYC flow works (text flow diagram)

```
User taps "Buy Investment Plan (Tier 2)"
        │
        ▼
[Backend] POST Digio "create KYC request" (template = yesly_ria_kyc)
        │  returns KYC request id (KID...) + gateway access token (GWT...)
        ▼
[App] Launch Digio KYC gateway (React Native / WebView SDK) with KID + token
        │
        ├── 1. PAN ──────────── user enters PAN (or uploads image → OCR)
        │        Digio verifies against NSDL/Protean + name match
        │
        ├── 2. Aadhaar ──────── EITHER DigiLocker pull (OAuth consent → signed
        │        Aadhaar XML + PAN from issuer)  OR  offline Aadhaar XML
        │        (user downloads from UIDAI with share code) → name/DOB/address/photo
        │
        ├── 3. Bank ─────────── penny drop: ₹1 IMPS to user's account number+IFSC
        │        → returns beneficiary name as per bank + fuzzy name-match score
        │        (fallbacks: reverse penny drop via UPI, penny-less lookup)
        │
        └── 4. Selfie ───────── liveness-checked selfie, face-matched against
                 Aadhaar/PAN photo → confidence score
        │
        ▼
[Digio → Backend] webhook: KYC request approved / failed (per-action results)
        │  Backend: 3 fails on any action → mark kyc_status=BLOCKED → grievance flow
        ▼
[Backend] Generate advisory agreement PDF (client details, fees, T&Cs per
          SEBI Sept-2020 guidelines Annexure) → POST Digio eSign "upload PDF"
        │  returns document id (DID...) + per-signer sign URL / token
        ▼
[App] Open Digio sign gateway → user consents → OTP to *Aadhaar-registered*
      mobile → UIDAI auth → CA issues one-time DSC → PDF signed (user), then
      Yesly officer counter-signs (DSC or Aadhaar eSign)
        │
        ▼
[Digio → Backend] webhook doc.signed / completed → download signed PDF +
      audit trail → field-encrypt → S3 (Mumbai) → unlock Tier 2 advice
        │
        ▼
[Backend] Upload/validate KYC with KRA (CVL/CAMS/NDML) — RIA obligation
```

Failure loop: each action allows retry; after **3 fails** → create grievance ticket (human review: name-mismatch override, alternate bank account, manual VIPV per SEBI's April 2020 KYC-tech circular), freeze purchase, notify user with reason + SLA.

---

## 3. Digio deep-dive (front-runner)

Digio (Digiotech Solutions Pvt. Ltd., Bengaluru) is a licensed ASP for Aadhaar eSign and one of the most widely used KYC/eSign stacks in Indian broking/MF onboarding. Products: **DigiKYC** (ID verification, DigiLocker, offline Aadhaar, bank verification, face match), **DigiSign** (Aadhaar/DSC/electronic eSign + eStamp), **DigiStudio** (no-code workflow templates: you assemble PAN → Aadhaar → penny-drop → selfie as one "journey" and invoke it with a single API), NACH/payments, and a **KRA module** (fetch + upload/update KYC details with KRAs — directly useful for the RIA obligation, see §10).

Docs: `https://documentation.digio.in/` (DigiKYC, DigiSign, DigiStudio sections). The docs are gated behind a JS app — endpoint paths below are from Digio's public docs/SDK repos/integration writeups and prior integrations; **[verify]** exact paths at build time.

### 3.1 Endpoints table

| Capability | Endpoint (production base `https://api.digio.in`) | Method | Notes |
|---|---|---|---|
| Create KYC request (workflow/template) | `/client/kyc/v2/request/with_template` | POST | Template built in DigiStudio; returns `KID...` id + `GWT...` access token **[verify path]** |
| Fetch KYC request result | `/client/kyc/v2/{kid}/response` | POST | Per-action statuses + extracted data **[verify]** |
| Download KYC media (selfie, Aadhaar XML, PAN image) | `/client/kyc/v2/media/{id}` | GET | **[verify]** |
| PAN verification (standalone) | `/v3/client/kyc/fetch_id_data/PAN` | POST | Body: `{"id_no":"ABCPE1234F","name":"...","dob":"..."}`; verifies vs income-tax DB + name match **[verify]** |
| Bank account penny drop | `/client/verify/bank_account` (confirmed product; path **[verify]**) | POST | ₹1 IMPS; returns bank-registered name + fuzzy match; also offers **penny-less**, **reverse penny drop (UPI)**, **VPA verification** |
| Aadhaar offline XML / DigiLocker | via KYC workflow actions (`digilocker`, `aadhaar_offline`) | — | DigiLocker pull returns issuer-signed Aadhaar/PAN; offline XML validated with share code |
| Face match / selfie liveness | via KYC workflow action (`selfie` / `ipv`) | — | Matches selfie vs ID photo, liveness check, confidence score |
| KRA fetch | `/digikyc/kra/.../fetch_details` (exists in docs; path **[verify]**) | POST | Fetch client KYC record from KRA by PAN |
| KRA upload/update | `/digikyc/kra/.../upload_details` (exists in docs; path **[verify]**) | POST | Upload/modify KYC record to KRA |
| eSign — upload PDF & create sign request | `/v2/client/document/uploadpdf` (base64 JSON) or `/v2/client/document/upload` (multipart) | POST | See §4 |
| eSign — get document status | `/v2/client/document/{document_id}` | GET | `agreement_status`: requested / completed / expired / failed **[verify field names]** |
| eSign — download signed PDF | `/v2/client/document/download?document_id=DID...` | GET | Signed PDF with embedded DSC + audit page |

### 3.2 Example JSON — create KYC request (template flow) **[verify field names]**

```json
POST /client/kyc/v2/request/with_template
Authorization: Basic base64(client_id:client_secret)

{
  "customer_identifier": "user@yesly.in",
  "customer_name": "Asha Rao",
  "template_name": "yesly_ria_kyc",
  "notify_customer": false,
  "generate_access_token": true,
  "reference_id": "yesly-usr-8842-step21"
}

→ 200
{
  "id": "KID2407XXXXXXXX",
  "status": "requested",
  "access_token": { "id": "GWT2407XXXXXXXX", "valid_till": "..." }
}
```

The app then launches Digio's gateway (Android/iOS/React-Native/WebView SDKs — `github.com/digio-tech/gateway_kyc`, SDK takes `KID` + identifier + `GWT` token, env `SANDBOX`/`PRODUCTION`) and the user completes all actions in Digio's white-labelled UI. Results come back by webhook + polling.

### 3.3 Example JSON — penny drop **[verify]**

```json
POST /client/verify/bank_account
{ "beneficiary_account_no": "50100XXXXXXX", "beneficiary_ifsc": "HDFC0000123",
  "beneficiary_name": "Asha Rao" }

→ { "verified": true,
    "beneficiary_name_with_bank": "ASHA R RAO",
    "fuzzy_match_score": 0.87, "id": "..." }
```

Store the raw bank-name string; run your own name-match threshold policy (see §13).

---

## 4. eSign workflow deep-dive (Digio DigiSign)

Flow (confirmed by Digio's Aadhaar-eSign docs + webhook docs):

1. **Backend** generates the advisory-agreement PDF and calls `POST /v2/client/document/uploadpdf`:

```json
{
  "file_name": "yesly-advisory-agreement-usr8842.pdf",
  "file_data": "<base64 PDF>",
  "signers": [
    { "identifier": "user@yesly.in", "name": "Asha Rao",
      "sign_type": "aadhaar", "reason": "Investment advisory agreement" },
    { "identifier": "principal@houseofalpha.in", "name": "House of Alpha (RIA)",
      "sign_type": "dsc", "reason": "RIA counter-signature" }
  ],
  "expire_in_days": 7,
  "display_on_page": "last",
  "notify_signers": false,
  "send_sign_link": false,
  "generate_access_token": true
}
→ { "id": "DID2407XXXXXXXX", "agreement_status": "requested",
    "signing_parties": [ ... ], "access_token": { "id": "..." } }
```
   Field names per public integration docs; **[verify]** `display_on_page`, coordinate-based placement (`sign_coordinates`) options against current docs. Note (confirmed): a signer gets **one identifier — email OR phone**; if both are sent, email wins and phone is ignored.
2. **App** opens Digio's sign gateway (SDK/WebView with `DID` + token, or the hosted `send_sign_link` URL).
3. **User signing ceremony**: user views the PDF → consents → chooses Aadhaar eSign → enters Aadhaar/VID → **OTP goes to the Aadhaar-registered mobile** (UIDAI, not Yesly's OTP) → on success the CA (Digio partners with a CCA-licensed CA/eSign ESP) generates a key pair and issues a **one-time Digital Signature Certificate** in the user's name and applies it to the PDF hash. Biometric (fingerprint/IRIS) variants exist for assisted flows.
4. **Counter-sign**: Yesly's principal officer signs with an organisational DSC or Aadhaar eSign (SEBI expects a bilateral executed agreement).
5. **Webhook** fires when **all** signers have acted (confirmed behaviour) → backend downloads the signed PDF (embedded signatures + audit trail: timestamps, IPs, auth mode), field-encrypts, stores in S3, flips `agreement_status=EXECUTED`, serves a copy to the user in-app (SEBI requires the client to get a copy).
6. **Expiry/decline**: sign requests expire (`expire_in_days`); handle `expired`/`declined` webhooks by regenerating.

The signed PDF is a standard PKCS#7/PAdES-signed document — validatable in Adobe Reader against the CCA India root.

---

## 5. Auth

- **Digio**: HTTP **Basic auth** — base64 of `client_id:client_secret` in the `Authorization` header, over TLS. Separate credential pairs for sandbox and production, issued from the Digio dashboard. Client-side SDK sessions use the short-lived `GWT` access tokens generated server-side (so secrets never ship in the app) — this fits Yesly's FastAPI-backend-holds-secrets model.
- **IDfy**: header pair `account-id` + `api-key` on every request.
- **HyperVerge**: header pair `appId` + `appKey` (+ `transactionId`).
- Keep all secrets in AWS Secrets Manager (Mumbai); never in the Expo bundle.

---

## 6. Webhooks

- Configure in Digio dashboard → Profile → Webhooks; subscribe to KYC events and `doc.signed`/completed events (confirmed: eSign webhook fires only after **all signers** complete).
- eSign payload includes: signer list, `documentSignId`/`id`, document status, `signedAt` timestamp, reference number — and (in at least the standard config) the **signed document as base64** in the payload. Prefer re-downloading via the download endpoint instead of trusting payload base64; verify webhook authenticity (Digio supports signature headers/shared-secret — **[verify]** exact mechanism) and make handlers idempotent.
- KYC webhooks report per-action approval/rejection so the backend can implement the 3-strikes → grievance logic without polling.
- Always run a reconciliation poller (webhooks get dropped; Digio also allows status GET by id).

---

## 7. Sandbox

- **Digio**: full sandbox at `https://ext.digio.in` (API) with dashboard at `ext-enterprise.digio.in`; production `https://api.digio.in` / `app.digio.in` **[verify exact hosts at onboarding]**. Sandbox Aadhaar eSign uses a simulated UIDAI flow (no real Aadhaar needed); penny drop in sandbox returns mocked bank responses. Access requires sales onboarding (no self-serve signup) — plan 1–2 weeks for commercial + credentials.
- **IDfy**: sandbox keys via sales; async task model identical in test.
- **HyperVerge**: trial `appId/appKey` via sales; SDK test mode.

---

## 8. Pricing (public ballparks only — all negotiable, confirm in quotes)

No vendor publishes a rate card; figures below are from public analyses (productgrowth.in, signyu.com) and market ballparks. **Treat as directional.**

| Item | Public ballpark (INR) | Confidence |
|---|---|---|
| Aadhaar eSign (per signature) | ₹8–20 volume-based (₹15–30 at low volume across ASPs; discounts >5k/month) | Medium (third-party analyses) |
| PAN verification (per check) | ~₹2–5 | Low — market ballpark, not published |
| Penny drop (per check) | ~₹2–5 (reverse penny drop similar; penny-less cheaper) | Low — market ballpark |
| Aadhaar offline XML / DigiLocker pull | ~₹3–10 | Low |
| Face match + liveness | ~₹2–10 | Low |
| Full KYC journey (all actions, per completed user) | roughly ₹20–50 | Low — sum of parts |
| eSign + eStamp combined | ₹50–120/doc (Digio facilitation ~₹30–50 + state stamp duty) — only if stamping is used | Medium |
| Platform/minimum commitments | Vendors typically want monthly minimums / prepaid credits; Digio has no published setup fee | Low |

Per successfully KYC'd + signed Tier 2 user, budget **~₹30–70 all-in** (excluding stamp duty, see §11). Unknowns to get in writing: charge on *failed* attempts (most vendors bill per API hit, i.e., your 3-retry design has cost), KRA fetch/upload fees, webhook/SDK fees (usually nil).

---

## 9. Vendor comparison — Digio vs IDfy vs HyperVerge

| Dimension | **Digio** | **IDfy** | **HyperVerge** |
|---|---|---|---|
| Coverage for Step 21 | **Everything incl. Aadhaar eSign + eStamp + KRA module** — single vendor | Very broad KYC ("breadth"): PAN, bank penny drop, DigiLocker, PAN–Aadhaar link check, AML, BGV; **has eSign API** | KYC verification only — **no eSign/eStamp**; would need a second vendor for the agreement |
| Signature strength | Licensed eSign ASP; Aadhaar OTP + biometric, DSC, multi-party workflows, audit trail | eSign offered but less central to product | — |
| Face/liveness | Good (score-based) — third-party rating ~4.1/5 KYC accuracy | Good | **Best-in-class**: iBeta Level-2 certified passive liveness, 99.5%+ face-match claims; strongest anti-spoof (masks/deepfakes) |
| DigiLocker depth | Native DigiLocker action inside KYC journeys; Digio is itself a DigiLocker authorised partner | DigiLocker integration offered | DigiLocker via workflow product |
| API style | REST, Basic auth, template/journey model (DigiStudio) + hosted SDK UI | REST, **async task model** (`POST eve.idfy.com/v3/tasks/async/...` → `request_id` → webhook/poll) — clean for FastAPI background jobs | REST + strong mobile SDKs (HyperKYC); API-first, most developer-friendly per third-party reviews |
| SLA / failure handling | Enterprise SLA in contract (uptime terms not public); per-action retry in journeys | Enterprise SLA; async model degrades gracefully (poll with backoff, ~2-min timeout guidance) | Enterprise SLA; SDK does client-side quality checks → fewer garbage submissions |
| Pricing model | Per transaction, negotiated; INR billing | Per verification, negotiated; enterprise sales | Per verification/MAU-style, negotiated; enterprise sales |
| Fit for Yesly | **Recommended primary** — one contract covers KYC + eSign + KRA; regulated-BFSI pedigree (RBI/SEBI/IRDAI use-cases) | Strong fallback / second source for verifications; good if you want best breadth of fraud checks | Add-on only if selfie-spoof fraud becomes a real problem, or as liveness engine inside another flow |

Recommendation: **Digio as primary for the whole step; keep IDfy as contractual backup for penny-drop/PAN (multi-vendor failover is common because bank/IMPS-side penny-drop outages are vendor-independent).** HyperVerge only if liveness quality becomes a differentiator.

---

## 10. DigiLocker, KRA and CKYC

### DigiLocker
Government of India's document wallet (MeitY). Citizens hold **issuer-signed** digital documents (Aadhaar, PAN, DL). Integration is an **OAuth 2.0-style consent flow** (now under the MeriPehchaan SSO umbrella): register as an Authorized Partner/Requester → get `client_id`/`client_secret` + redirect URI → user is redirected to DigiLocker, authenticates (Aadhaar OTP etc.), **consents at document/scope level** → your server exchanges the auth code for a token → pulls the requested issued documents (Aadhaar XML, PAN). Spec: "Digital Locker Authorized Partner API Specification v2.0". In practice **you don't integrate DigiLocker directly** — Digio/IDfy are already authorised partners and expose it as a KYC action; direct partnership is a months-long onboarding with MeitY. SEBI's 24 April 2020 circular ("Clarification on KYC Process and Use of Technology for KYC") explicitly blesses DigiLocker-sourced documents, Aadhaar offline XML, eSign and video-IPV for securities-market KYC.

### KRA (KYC Registration Agencies) — the RIA obligation
- Governed by **SEBI {KRA} Regulations, 2011** and the **Master Circular on KYC norms for the securities market** (SEBI/HO/MIRSD/SECFATF/P/CIR/2023/169, 12 Oct 2023, as amended). Five KRAs: **CVL** (CDSL Ventures), **CAMS**, **NDML** (NSDL), DOTEX, Karvy.
- As a SEBI-registered intermediary, an RIA must, for every advisory client: **(1) first fetch** the client's KYC record from a KRA by PAN; **(2) if a *validated/registered* record exists**, rely on it (no re-doing KYC — that's the point of KRAs); **(3) if none exists or it's outdated**, perform KYC (the Digio flow above) and **upload the KYC record to a KRA** within the prescribed timeline (SEBI norms: ~3 working days). KRAs then independently **validate** records (PAN–Aadhaar linkage + official DB checks → "Validated" status) and notify the client by email/SMS.
- So Yesly's Step 21 needs a **KRA leg**: fetch-first, upload-after. Digio's DigiKYC has KRA **Fetch Details** and **Upload/Update Details** APIs (confirmed to exist in its docs), which saves a direct CVL/CAMS integration; Yesly still needs its **own KRA membership/credentials** (KRAs bill intermediaries directly, per-fetch/per-upload fees ~₹15–35 **[unverified ballpark]**).
- Practical note: many Tier 2 users who already invest in MFs/stocks will be "KYC Validated" at a KRA — for them the flow can collapse to *KRA fetch + penny drop (bank not in KRA record) + agreement eSign*, cutting cost and drop-off. Design the journey to branch on KRA status.

### CKYC (CERSAI / CKYCRR)
Central KYC Records Registry under PML Rules 2005, run by CERSAI. Reporting entities under PMLA upload standardised KYC records and get a 14-digit KIN. Since Aug 2024 SEBI has moved to a model where **KRAs themselves upload securities-market KYC records to CKYCRR** (SEBI circular on "Uploading of KYC information by KRAs to CKYCRR", with a 6-month migration window from 1 Aug 2024) — so for a SEBI RIA the practical integration surface is the **KRA**, not CERSAI directly. Keep CKYC in view only if compliance counsel says House of Alpha has an independent CKYCRR upload duty; the KRA-upload route is the operative one for securities clients. **[Interpretation — confirm with counsel.]**

---

## 11. Legal validity of e-signatures for the RIA advisory agreement (the OTP-vs-eSign answer)

**IT Act 2000 framework:**
- **Section 3**: "digital signature" — asymmetric-crypto DSC issued by a CCA-licensed Certifying Authority.
- **Section 3A + Second Schedule**: "electronic signature" — other notified reliable techniques; **Aadhaar eSign is notified in the Second Schedule**, so it is a statutorily recognised electronic signature.
- **Section 5**: where law requires a signature, an electronic signature *affixed in the manner prescribed by the Central Government* satisfies that requirement.
- **Evidence law** (Indian Evidence Act / Bharatiya Sakshya Adhiniyam 2023): recognised electronic signatures (DSC, Aadhaar eSign) carry **statutory presumptions of validity**; click-wrap/email-OTP acceptance does **not** — it can form a contract under general contract law, but you must prove who clicked, with no presumption, and its status as a "signature" is disputable.

**SEBI's position for RIAs:**
- IA Regulations Reg. 19(1)(d) (2020 amendment): advisory agreement must be **signed** by the client before any advice/fee; client must get a copy. (Existing clients had to be papered by 1 Apr 2021.)
- SEBI's **Guidelines for Investment Advisers** circular (SEBI/HO/IMD/DF1/CIR/P/2020/182, 23 Sept 2020) prescribes mandatory agreement contents and contemplates execution through legally acceptable electronic modes including **Aadhaar e-sign**.
- **SEBI informal guidance to Paytm Money Ltd (2021)** on exactly this question: **mere electronic consent (email confirmation / click-through / plain OTP acknowledgement) is NOT sufficient** to satisfy the "signed agreement" requirement. Acceptable modes: **(1) wet signature (incl. scanned exchange), (2) Aadhaar eSign, (3) DSC**. (Secondary source: cskruti.com summary of the guidance; obtain the guidance letter itself for the compliance file.)
- Contrast: SEBI's informal guidance to **Purnartha Investment Advisers (20 May 2021)** accepted electronic signatures/stylus acknowledgements for **portfolio-manager fee annexures**, citing IT Act s.5 — i.e., SEBI is comfortable with *recognised* e-signatures generally, but that guidance was about PMS fee acknowledgement, not the RIA agreement.

**Answer for Yesly:** an email/SMS-OTP "I agree" is **not defensible** as the signed RIA advisory agreement — it is neither a recognised electronic signature under the IT Act nor accepted per SEBI's informal guidance. **Use Aadhaar eSign (OTP-based) via Digio for the client, counter-signed by House of Alpha (DSC recommended).** OTP click-acceptance remains fine for *other* consents (T&Cs, DPDP consent, risk-profile confirmation) — just not for the advisory agreement itself.

**Stamp duty:** the Indian Stamp Act (and state Acts) makes most written agreements stampable; an **unstamped/under-stamped agreement is inadmissible in evidence** (s.35) until duty + penalty (up to 10×) is paid — it doesn't void the contract but cripples enforcement. Advisory/service agreements typically attract only the state's nominal "agreement" duty (₹100–500 in most states, e.g., Art. 5(h) type entries — **rate is state-specific; confirm for the client's state**). Duty is payable **at/before execution**, and e-stamping of e-signed docs is done via SHCIL/authorised vendors — Digio bundles eStamp (~₹30–50 facilitation + duty). Pragmatic approach used by many RIA platforms: execute on plain e-sign and accept the (small, curable) stamping risk, or franking/e-stamp in the RIA's home state — **decision for counsel; budget ₹0–550/agreement depending on the call**.

---

## 12. Compliance notes

- **Sequence lock**: enforce in backend state-machine — no advice payload and no fee charge until `kyc_status=APPROVED` **and** `agreement_status=EXECUTED`; timestamp both (SEBI inspection evidence).
- **Record retention**: SEBI IA Regulations require records (KYC, agreements, advice, rationale) kept **min. 5 years** (electronic form OK); PMLA requires KYC records **5 years from end of the client relationship** — so retention clock ≥ 5 years *post-offboarding*, not post-signing. S3 lifecycle: retain + legal-hold, no auto-delete.
- **Aadhaar handling**: do **not** store the full Aadhaar number (Aadhaar Act/UIDAI regs) — store the offline-XML reference id + masked number (last 4 digits) only; the Digio-hosted flow keeps raw Aadhaar out of Yesly's systems (keep it that way — don't proxy Aadhaar numbers through FastAPI logs).
- **DPDP Act 2023 + DPDP Rules 2025** (notified 14 Nov 2025; substantive fiduciary obligations phase in by ~May 2027): itemised consent notice before collecting ID/biometric data, purpose limitation (KYC data can't be reused for marketing without separate revocable consent), 72-hour breach notification, security safeguards (field-level encryption in S3 helps), data-principal rights (access/erasure — reconcile erasure requests against the SEBI/PMLA retention duty: regulatory retention wins, document the refusal ground). Penalties to ₹250 crore.
- **Grievance flow**: SEBI RIA norms require a grievance-redressal mechanism incl. SCORES/ODR registration — the 3-fails route should create an auditable ticket with defined SLA and appear in grievance MIS.
- **Outsourcing/vendor diligence**: keep Digio's DPA, ISO/SOC certificates, and India data-residency confirmation on file (SEBI intermediary outsourcing guidelines).

---

## 13. Gotchas & effort

**Gotchas**
1. **Aadhaar–mobile linkage is a hard dependency** for OTP eSign and OTP-based DigiLocker: no linked/active mobile → no OTP → signature impossible. There is no in-flow fix (user must update mobile at an Aadhaar centre, takes days). Detect early (offline XML flow reveals a mobile-hash presence) and design the grievance path for it; biometric eSign is the assisted-mode fallback but is impractical for a pure-app product.
2. **Name mismatch across PAN/Aadhaar/bank** is the #1 KYC drop-off (initials vs expanded names, "ASHA R RAO" vs "Asha Rao", maiden names). Penny-drop returns fuzzy scores — set a threshold (e.g., auto-pass ≥0.8, manual review 0.6–0.8, fail <0.6) rather than exact match, and log the score for audit.
3. **Penny-drop is flaky by bank**: IMPS downtime windows (esp. late-night/bank maintenance), some PSU banks/co-op banks return garbled or truncated beneficiary names, some accounts (NRE, certain wallets) fail structurally. Mitigate with retry-later queues, reverse penny drop (user pays ₹1 by UPI — also proves account *operability*), and a second vendor fallback.
4. **Retries cost money** — vendors bill per attempt. The 3-strikes design multiplies per-user cost on bad cohorts; add client-side validation (PAN checksum, IFSC lookup) before burning API calls.
5. **Webhook loss / ordering**: eSign webhook only fires when *all* signers finish — if the Yesly counter-signature is manual, the user-facing "done" state must not wait on it; auto-apply the org DSC server-side.
6. **eSign requests expire** (`expire_in_days`); half-signed expired docs need regeneration logic and the old DID invalidated.
7. **KRA branch**: skipping the KRA fetch and always re-KYCing wastes money and violates the fetch-first expectation; conversely a stale "on hold" KRA status must fall back to fresh KYC.
8. **Selfie liveness in Expo**: Digio's flow runs in its SDK/WebView — camera permissions, low-light captures and older Android devices cause real failure rates; test on low-end devices.
9. **DigiLocker outages** happen (government infra); always keep offline Aadhaar XML as the alternate Aadhaar path.
10. **Sandbox ≠ production** for penny drop and eSign (mocked in sandbox); do a small production pilot before launch.

**Effort estimate** (FastAPI backend + Expo app, Digio single-vendor):
- Vendor onboarding/commercials + sandbox creds: 1–2 weeks (calendar).
- Backend: KYC request orchestration, webhooks + reconciliation poller, 3-strikes state machine, agreement PDF generation, eSign lifecycle, S3 field-encryption, KRA fetch/upload: **~3–4 dev-weeks**.
- App: Digio SDK/WebView embedding in Expo (needs a dev-client / native module — not Expo Go), status screens, failure/grievance UX: **~2 dev-weeks**.
- Compliance artefacts (agreement template per SEBI Annexure, DPDP notices, retention policy): 1 week with counsel.
- Total: **~6–8 dev-weeks** plus vendor/counsel calendar time.

---

## 14. Sources

- Digio product/docs: https://www.digio.in/ · https://www.digio.in/digi-kyc/ · https://www.digio.in/digi-sign/ · https://documentation.digio.in/digikyc/ · https://documentation.digio.in/digisign/types_of_sign/aadhaar_based/ · https://documentation.digio.in/digisign/api_integration/webhooks/ · https://documentation.digio.in/digikyc/bank_account_verification/api_integration/penny_drop/ · https://documentation.digio.in/digikyc/kra/api_integration/fetch_details/ · https://documentation.digio.in/digikyc/kra/api_integration/upload_details/ · https://github.com/digio-tech/gateway_kyc
- Digio via integrators/analyses: https://developers.getknit.dev/docs/getting-started-with-digio-esign-apis · https://productgrowth.in/tools/kyc-identity/digio/ · https://digiqt.com/api-details/india/fintech/aadhar-e-sign-api-digio/
- IDfy: https://www.idfy.com/identity-verification-solutions-by-idfy/api-solutions-overview/ · https://www.idfy.com/bank-account-verification-api/ · https://eve-api-docs.idfy.com/ · https://api-docs.idfy.com/v3/ · https://eve-docs.idfy.com/
- HyperVerge: https://hyperverge.co/ · https://github.com/hyperverge/kyc-india-rest-api · https://hyperverge.co/blog/sebi-kyc-guidelines/
- DigiLocker: https://www.digilocker.gov.in/web/partners/requesters · https://meripehchaan.gov.in/assets/img/chose/Digital%20Locker%20Authorized%20Partner%20API%20Specification%20v2.0.pdf · https://medium.com/@abhaygzb15/digilocker-integration-architecture-a-secure-oauth-based-system-2b844ba63ccc
- SEBI: IA (Amendment) Regulations 2020 — https://www.sebi.gov.in/legal/regulations/jul-2020/sebi-investment-advisers-amendment-regulations-2020_47007.html · Guidelines for Investment Advisers (23 Sep 2020) — https://www.sebi.gov.in/legal/circulars/sep-2020/guidelines-for-investment-advisers_47640.html · Master Circular on KYC norms (12 Oct 2023) — https://www.sebi.gov.in/legal/master-circulars/oct-2023/master-circular-on-know-your-client-kyc-norms-for-the-securities-market_77945.html · KRA→CKYCRR upload circular (via NSDL) — https://nsdl.co.in/downloadables/pdf/2024-0078-_Policy-SEBI_Circular_on_Uploading_of_KYC_information_by_KYC_Registration_Agencies_(KRAs)_to_Central_KYC_Records_Registry_(CKYCRR).pdf
- Legal validity: https://www.leegality.com/blog/law-around-aadhaar-esign · https://corridalegal.com/e-signatures-india-legal-validity-compliance-use-cases/ · https://helpx.adobe.com/legal/esignatures/regulations/india.html · SEBI informal guidance (Paytm Money, e-consent not allowed) — https://cskruti.com/electronic-consent-not-allowed-for-agreements-sebis-clarification-for-rias/ · SEBI informal guidance (Purnartha, s.5 IT Act) — https://www.finseclaw.com/article/sebis-informal-guidance-use-electronic-signatures
- KRA: https://www.cvlindia.com/KRA/KRA · https://www.cvlindia.com/CVLINDIA_DOC/pdf/CVL-KRA%20Operating%20Instructions-New.pdf
- Stamp duty: https://vinodkothari.com/2020/01/stamp-duty-implications-on-e-agreements/ · https://www.leegality.com/blog/esign-estamp-faqs · https://www.esign.ai/blog/pay-stamp-duty-electronically-signed-document-india-states
- Pricing ballparks: https://signyu.com/guides/aadhaar-esign · https://signyu.com/compare/pricing · https://productgrowth.in/tools/kyc-identity/digio/
- DPDP: https://www.pib.gov.in/PressReleasePage.aspx?PRID=2190655 · https://www.ey.com/en_in/insights/cybersecurity/transforming-data-privacy-digital-personal-data-protection-rules-2025 · https://www.befisc.com/fintechsherlock/dpdp-act-kyc-compliance-fintech/
