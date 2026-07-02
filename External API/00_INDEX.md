# External API Research — Index

Deep-dive research (July 2026, from official docs + verified secondary sources) on every external integration in the Yesly flow — cross-checked against the `api:` field of all 31 steps in `Yesly_Flow.html`, its "External integrations" table, plus auth, Tier-4 scheduling, the Video CDN chips, and the Supabase identity bridge. One file per integration; each covers: overview & role in the flow, how it works, endpoint I/O tables with JSON, auth, webhooks, sandbox, pricing, India compliance (SEBI/RBI/DPDP), gotchas and effort. Figures that could not be verified against a live official page are flagged **[verify before build]** inside the files.

## The 15 integrations at a glance

| # | File | Role in flow | Verdict / front-runner | Cost ballpark | Biggest risk |
|---|---|---|---|---|---|
| 01 | [Account Aggregator](01_Account_Aggregator.md) | Step 22 (Tier 2+) — consent-based pull of bank/MF/equity data; Chart 05 | **Setu** — best docs, self-serve sandbox w/ mock FIPs, handles ReBIT JWS + ECDH decryption (3–4× less effort than raw ReBIT) | ~₹10–25 per successful fetch (unofficial; bilateral pricing — get a quote) | FIP flakiness + bank-registered-mobile coupling; Setu webhooks have no retries/signatures → polling backstop. ~6–9 eng-weeks |
| 02 | [KYC + eSign](02_KYC_eSign.md) | Step 21 — PAN → Aadhaar/DigiLocker → penny-drop → selfie → sign RIA agreement; Chart 11 | **Digio** primary (only vendor with KYC + eSign + eStamp + KRA in one contract); IDfy failover | ~₹30–70 all-in per completed Tier-2 user; eSign ₹8–20/signature | Aadhaar-mobile linkage failures have **no in-app fix** — grievance path must absorb them; fuzzy name-matching across PAN/Aadhaar/bank |
| 03 | [Razorpay](03_Payments_Razorpay.md) | Paywall (₹2,999/₹4,999 one-time) + Tier-3 subscription via UPI Autopay; Chart 08 | **Razorpay**, wrapped behind an internal gateway interface so **Cashfree** (1.6–1.95%, ~T+1) stays a negotiation lever | ~2% + 18% GST (~₹118 on ₹4,999); UPI Autopay recurring pricing quote-only | UPI e-mandate lifecycle friction (approval drop-off, post-notice execution failures, ₹15k AFA cap — **no MF carve-out for advisory fees**); Apple IAP ambiguity → keep web/Payment-Links fallback |
| 04 | [smallcase Gateway](04_Broker_smallcase_Gateway.md) | Step 30 — stock-basket execution in user's own broker a/c (v1 deep-link, v1.5 in-app); Chart 07 | Strong fit for v1.5; keeps Yesly execution-only (never broker of record) | Not public — negotiated B2B | **No self-serve sandbox** — commercial onboarding gates all testing; per-broker feature gaps (coverage matrix non-public) |
| 05 | [BSE StAR MF](05_BSE_StAR_MF.md) | Step 30 — mutual-fund orders; ICCL nodal settlement (Yesly never touches client funds) | The (effectively only) compliant MF rail — register HoA (INA000018364) as StAR MF RIA member; SOAP order-entry + JSON UCC/2FA/payment APIs | ~Free to investors; member reg ₹0–15k; per-txn charges land on AMCs; + BASL fees (~₹1L corporate initial) — confirm current schedule | Operational, not technical: membership + product-demo approval gate go-live (weeks); SOAP session expiry, polled feedback (no webhooks), 2–3-week physical mandate activation |
| 06 | [WhatsApp BSP](06_WhatsApp_BSP.md) | Notification fan-out (nudges) via Container 5 | **Go direct with Meta Cloud API** (zero markup, first-party webhooks); Gupshup fallback; Wati ruled out | Utility/auth ~₹0.13–0.145/msg, marketing ~₹0.86–1.09/msg **[verify]** (~₹12.7k/mo at 50k utility + 5k marketing) | Meta reclassifying nudge templates as "marketing" (7–8× cost); India per-user marketing caps silently drop delivery (error 131049) — template wording is load-bearing |
| 07 | [Expo Push](07_Expo_Push.md) | Push channel of the 3-channel fan-out | Expo push service + access token; graduate to OneSignal/FCM only if segmentation needed | Free | Tickets ≠ delivery — must poll receipts within 24h + prune `DeviceNotRegistered`; Indian OEM battery optimizers suppress even "ok" receipts |
| 08 | [Amazon SES](08_Amazon_SES.md) | Email channel (ap-south-1) | SES v2, shared IPs, **two configuration sets** (transactional vs marketing on separate subdomains) | ~$0.10/1k emails (~₹8.6/1k) **[verify]** | Account-wide reputation: one bad marketing blast over bounce/complaint thresholds pauses OTP email for the whole product. ~6–9 days |
| 09 | [Mixpanel](09_Mixpanel.md) | Analytics on nearly every screen; A/B tests on the 3 opening questions | Mixpanel **with India residency** (GCP Mumbai, `api-in.mixpanel.com` — exists since Oct 2024; irreversible per project) | Free to 1M events/mo (covers MVP); ~$0.28/1k events beyond | Permanent identity-merge links (`reset()` on logout!) and PII leaking into properties — never send financial data; enforce a Lexicon tracking plan |
| 10 | [Sentry](10_Sentry.md) | Error + performance monitoring (RN app ↔ FastAPI distributed tracing) | SaaS Sentry, **EU region** (no India option; region immutable at org creation); `@sentry/react-native` (sentry-expo is deprecated) | Team $26/mo base → ~$50/mo realistic | Manual source-map upload after `eas update` (broken stacks otherwise); quota burn from noisy errors; strict PII scrubbing required |
| 11 | [Market data](11_Market_Data.md) | NAVs, XIRR inputs, benchmarks (F03/F06) + Nifty PE/PB & VIX for the rebalancing overlay | **v1: free AMFI NAVAll.txt** (nightly ingest) + in-house XIRR + **one scheme-master license (Accord ACE MF preferred**, CMOTS comparison quote); Value Research only if third-party ratings become a product need | AMFI/NSE reports/RBI free; vendor feeds quote-based (no public prices) | Displaying NSE-originated index/VIX data commercially without an NSE Data & Analytics license — even EOD counts as redistribution. Technical: Nifty P/E structural break Apr-2021 → lead the regime overlay with P/B percentile |
| 12 | [AWS Cognito](12_Auth_Cognito.md) | Login (Step 3), Google OAuth, email OTP, Supabase→Cognito identity bridge | **Essentials tier**, ap-south-1; managed passwordless **email OTP is GA** (no custom Lambda triggers); bridge via `AdminCreateUser` (SUPPRESS) | Free to 10K MAU; then $0.015/MAU | Federated-vs-native duplicate accounts — PreSignUp/`AdminLinkProviderForUser` linking must ship **before** Google login; identity data stays in Mumbai (good DPDP posture) |
| 13 | [Scheduling](13_Scheduling_Calendly_Calcom.md) | Step 31 / G07 — Tier-4 advisor booking | MVP: manual or Calendly Standard; target **Cal.com Platform** for native in-app booking (Calendly has no booking-creation API — webview forever) | Cal.com Platform: free/25 bookings → $299/mo/500 **[verify]** | Cal.com went closed-source Apr-2026; MIT fork **Cal.diy** is the only self-hosted/Mumbai-residency route (feature-drift risk) |
| 14 | [Video CDN](14_Video_CDN.md) | Founder videos at ~5 steps (welcome, SEBI note, mandatory pre-pitch video, free→paid transition) — vendor undecided | **S3 + CloudFront progressive MP4 for v1** (~1 dev-day, no ABR needed); switch to **Mux** if engagement/QoE analytics or catalog churn matter; YouTube/Vimeo rejected (ToS, branding, skip control). "Can't-skip" + watched-vs-skipped is client-side player events → Mixpanel regardless of CDN | CloudFront ~$0 at Yesly's volume (1 TB/mo free tier); Mux ~$0–20/mo; Cloudflare Stream ~$25/mo flat | Gating the flow on a mandatory video with no load-failure fallback — a CDN blip blocks 100% of conversions at that step; hand-rolled HLS hits Android ExoPlayer quirks |
| 15 | [Supabase community bridge](15_Supabase_Community_Bridge.md) | Warm-funnel identity bridge — YesLifers (Supabase) `COMMUNITY_PROFILE` → Cognito `USER`/`AUTH_SESSION` (email, name, avatar, `pledge_taken`); warm users skip P1–P3 | **Server-side REST pull** with a secret key scoped via a dedicated `bridge_profile_v` view + an `identity_map` UUID table from day one (email only for first contact); database webhooks for `pledge_taken` freshness in v1.1 (idempotent upserts + nightly reconcile) | Bridge ~$0; community project needs Supabase **Pro $25/mo** (Free tier pauses after 1 week idle — would break the bridge); Mumbai region confirmed | service_role key blast radius (leak = full community DB, RLS bypassed) and email-as-join-key desync; DPDP: cross-product sharing is a purpose change needing fresh consent notice; deletion does **not** propagate either way — must be disclosed |

## Verdicts on the doc's open decisions

| Open decision (from `Yesly_Flow.html`) | Research verdict |
|---|---|
| **Decision 1 — AA vendor** (Setu/Anumati/OneMoney/CAMSFinServ/Finvu) | **Setu.** Only vendor with self-serve sandbox + mock FIPs; TSP layer absorbs ReBIT signing/crypto; multi-AA routing possible. Pricing must be quoted — treat ₹10–25/fetch as placeholder. Prereq regardless of vendor: Sahamati FIU certification + Central Registry listing in House of Alpha's name. |
| **Decision 2 — KYC vendor** (Digio/IDfy/Hyperverge) | **Digio**, with IDfy as penny-drop/PAN failover; Hyperverge only if selfie-spoofing emerges. Digio uniquely bundles eSign + eStamp + KRA fetch/upload — one contract for Step 21 end-to-end. |
| **Decision 3 — WhatsApp BSP** (AiSensy/Gupshup/Wati) | **None of the three: go direct with Meta Cloud API** from Container 5 (Yesly needs an API pipe, not campaign tooling). Gupshup if a BSP is forced; AiSensy only if marketing tooling becomes a priority; Wati ruled out (markup + weakest API). |
| **Decision 9 — Tier-4 scheduling** (Calendly/Cal.com/manual) | **Manual or Calendly for MVP → Cal.com Platform when booking goes native in-app.** Calendly can never be fully in-app (no booking-creation API). Self-hosting for data residency now means the Cal.diy fork, not Cal.com. |

## Compliance headlines (cross-cutting)

1. **The RIA agreement cannot be "signed" by OTP-acceptance.** SEBI guidance (Paytm Money, 2021) accepts wet signature, **Aadhaar eSign, or DSC** only. Pattern: client signs via Aadhaar eSign (₹8–20), HoA counter-signs via DSC. Email/click-wrap OTP remains fine for non-agreement consents. → resolves the open item from the 30-Jun meetings (file 02).
2. **KYC/agreement trigger is "personalised advice for a fee", not tier.** If Tier 1 outputs any risk-profiled recommendation, KYC + signed agreement may be required at Tier 1 too — flag to counsel (file 02; updates doc Decision 5).
3. **Yesly qualifies as an AA FIU** via HoA's SEBI registration; storage of fetched data is allowed under `consentMode: STORE` bounded by DataLife + purpose code 101; purge on revoke/expiry (file 01).
4. **RBI e-mandate rules bite Tier-3 billing:** ₹15,000 AFA cap applies to advisory subscriptions (the ₹1-lakh carve-out is only for MF/insurance/credit-card payments); 24-hr pre-debit notification adds failure surface (file 03).
5. **App stores:** Google Play affirmatively exempts investment-advisory apps from Play Billing; **Apple is the open gamble** — keep a web/Payment-Links fallback for iOS (file 03).
6. **Data residency:** Cognito + RDS + SES all live in ap-south-1; Mixpanel has an India region; Sentry does not (EU best available); Calendly is US-only — note in the DPDP record of processing (files 08–10, 12, 13).

## Deliberately not separate files (and why)

- **PDF generation → S3, CloudFront, RDS** — internal engine output + AWS infrastructure, not third-party API partners.
- **App Stores (Apple/Google)** — distribution, not an API; the billing-policy analysis lives in file 03.
- **Google OAuth** — covered inside file 12 (Cognito federation); **DigiLocker** — covered inside file 02.
- **Biometric login** (returning users) — device-local, no external call.
- **GitHub Actions / Vercel** — CI and hosting, not runtime integrations.

## Mentioned in meetings but not yet in the flow doc (track, don't build)

- **NSDL CAS / MF Central** — Harish (30-Jun, 3 PM) named CAS statements as an alternative portfolio-import route alongside AA. Not in the doc's integration table; revisit if AA FIP coverage disappoints.
- **CRM / martech stack** (SFMC vs NetCore, omni-channel lead manager, nudge orchestration) — explicitly opened as a workstream on the dev-agency call; premature to research until the martech workshop picks a direction. The doc's notification files (06–08) cover only the delivery rails it would drive.

## Sequencing / effort signal (longest poles first)

1. **BSE StAR MF** — membership + BASL + product-demo approval are calendar-gated (weeks) before any code matters; start paperwork first.
2. **smallcase Gateway** — commercial onboarding gates the sandbox; lead time may dominate v1.5.
3. **Account Aggregator** — ~6–9 eng-weeks; FIP data normalisation is the long pole.
4. **KYC/eSign** — vendor contract + grievance-path design.
5. Razorpay, SES, Expo, Cognito, Mixpanel, Sentry — days each, standard integrations.
6. Market data — v1 is a free-feed ingestion job + one license negotiation (Accord).

---
*Generated 02-Jul-2026 from 7 research agents; integration list verified against the `api:` field of all 31 flow steps. Every file lists its sources; anything marked [verify before build] must be re-checked against the vendor's current page before contracts or code.*
