# 05 - BSE StAR MF (Mutual Fund Order Routing & Settlement)

> Research date: July 2026. Primary sources: bsestarmf.in official "BSE StAR MF - Webservice Structure v3.5 (Aug 2023)" PDF, BSE India membership pages, SEBI circulars.
>
> **Important caveat:** BSE distributes its full API kit (latest web-service structure docs, v2/REST specs, UAT credentials) only to registered members on request via email with a live Member ID. Everything below marked "verified" comes from the publicly downloadable official PDF (`bsestarmf.in/APIFileStructure.pdf`) and BSE/SEBI pages; items marked "unverified / member-only" could not be confirmed from public documentation and must be validated after membership.

---

## Overview & role in Yesly

**What it is.** BSE StAR MF is BSE's web-based mutual fund order-routing and transaction platform (launched 4 Dec 2009). It is India's largest MF order platform by a wide margin — media reports citing BSE put it at 85%+ share of exchange-based MF distribution, ~66.3 crore transactions in FY25 and a record 7.13 crore transactions in October 2025. It connects Mutual Fund Intermediaries (MFIs — BSE brokers with ARN), Mutual Fund Distributors (MFDs — limited-purpose members), and SEBI-Registered Investment Advisers (RIAs) to all major AMCs/RTAs (CAMS, KFintech) for purchase, redemption, switch, SIP/XSIP/ISIP, STP and SWP.

**Money never pools with the platform or the intermediary.** Settlement runs through the exchange's clearing corporation (ICCL) nodal mechanism: investor's bank account → ICCL → AMC scheme account, and units are allotted directly into the investor's own folio (or demat account). Per BSE's RIA framework page, an RIA's operations are "restricted to investor onboarding and order placement and at no point of time will they be in custody of funds or units of the investors." This aligns with SEBI's 2021-22 circulars discontinuing pooling of funds/units by platforms and intermediaries (effective 1 July 2022).

**Role in Yesly (Step 30 of the Yesly flow).** Mutual-fund orders in Yesly execute through BSE StAR MF. House of Alpha is a SEBI-registered RIA (**INA000018364**), so Yesly would:

- Obtain BSE StAR MF registration as an RIA (RIA/member code) — RIAs are explicitly eligible per BSE's "StAR Mutual Fund Platform — Registered Investment Advisors" page.
- Register each advisory client as a UCC (Unique Client Code) on StAR MF with full KYC/FATCA/bank/nominee data.
- Place **direct-plan** orders only (advisory model, zero commission) under the RIA code from the FastAPI backend.
- Client pays from their own bank account (net banking / UPI / NACH mandate for SIPs) into the ICCL nodal account; units credit to the client's own folio. Yesly never touches client money.

## How it works

```
Yesly RN app                Yesly FastAPI backend                BSE StAR MF / ICCL                AMC / RTA
     |                              |                                   |                              |
     |-- KYC-verified client ------>|                                   |                              |
     |                              |-- UCC Registration (JSON API) --->|                              |
     |                              |-- FATCA upload / nominee API ---->|                              |
     |                              |                                   |                              |
     |-- "Invest 10,000 in X" ----->|                                   |                              |
     |                              |-- getPassword (SOAP) ------------>|  (encrypted session pwd)     |
     |                              |-- orderEntryParam PURCHASE ------>|-- order registered --------->|
     |                              |<- order no. + pipe-delim resp ---|                              |
     |                              |-- Single Payment API ------------>|  (netbanking/UPI/mandate)    |
     |<-- payment page / UPI intent-|                                   |                              |
     |-- client pays from OWN bank ------------------------------------>| ICCL nodal account           |
     |                              |                                   |-- funds to AMC before cutoff>|
     |                              |-- 2FA Subscription API ---------->|  (BSE OTP page in app)       |
     |<-- OTP page (webview) -------|                                   |                              |
     |-- client enters OTP -------------------------------------------->|  order confirmed             |
     |                              |<- server-to-server POST ----------|  (verified orders)           |
     |                              |-- OrderStatus / Allotment API --->|<-- allotment (T+1) ----------|
     |<-- "Units allotted" ---------|                                   |  units in CLIENT folio/demat |

Redemption: orderEntryParam (R) -> BSE 2FA redemption OTP by client -> RTA -> proceeds paid
directly to client's registered bank account (T+2/T+3 per scheme). Platform never holds funds.
SIP: mandate registration (e-NACH/UPI/physical) -> XSIP/ISIP registration -> BSE auto-triggers
each instalment on schedule -> track child orders via ChildOrderDetails API.
```

## Key endpoints / SDK calls

There is no official SDK; integration is against SOAP 1.2 (order entry) and mixed SOAP/JSON services. Verified endpoints from the official v3.5 document (demo hostnames shown; live equivalents are shared with members — live order entry WSDL is at `bsestarmf.in`):

| Name (service / method) | Purpose | Key request fields | Key response fields |
|---|---|---|---|
| `MFOrderEntry/MFOrder.svc` → `getPassword` (SOAP) | Create session for order APIs | UserId (5), Password (20), PassKey (10, random entropy) | `ResponseCode|EncryptedPassword` — 100 success / 101 invalid; session valid **1 hour** |
| `MFOrder.svc` → `orderEntryParam` (SOAP) | Lumpsum purchase / redemption (NEW only; MOD/CXL discontinued for lumpsum due to real-time RTA integration) | TransCode, TransNo (`YYYYMMDD<memberid>NNNNNN`), UserID, MemberId, ClientCode (UCC), SchemeCd, BuySell (P/R), BuySellType (FRESH/ADDITIONAL), DPTxn (C/N/P), OrderVal or Qty, AllRedeem, FolioNo, KYCStatus, EUIN/EUINVal, DPC, Password (encrypted), PassKey, Param2 (PG ref, purchase), Param3 (bank a/c, redemption), MobileNo, EmailID, MandateID (OTM purchase) | Pipe-delimited: TransCode \| TransNo \| BSE OrderId \| UserId \| MemberId \| ClientCode \| BSE remarks \| Success flag (0 = ok) |
| `MFOrder.svc` → `sipOrderEntryParam` (SOAP) | SIP registration/cancel | SchemeCd, ClientCode, start date, FREQUENCY TYPE (MONTHLY/QUARTERLY/WEEKLY), INSTALLMENT AMOUNT, NO OF INSTALLMENTS, FIRSTORDERFLAG, TRANSMODE (D/P), REGID (for CXL), Param2 = end date (daily SIP) | SIP RegId, status remarks |
| `MFOrder.svc` → `xsipOrderEntryParam` (SOAP) | XSIP / ISIP registration (mandate-backed exchange SIP) | As SIP + Mandate ID | XSIP RegId, remarks |
| `MFOrder.svc` → `spreadOrderEntryParam`, `switchOrderEntryParam` (SOAP) | Overnight spread orders; switch between schemes | From/To scheme codes, amount/units | Order no., remarks |
| `StarMFWebService/StarMFWebService.svc` → `getPassword`, `MFAPI` (SOAP; **5-minute** session) | "Additional services" multiplexer, selected by `Flag` param: FATCA upload, **mandate registration** (physical/X-SIP/e-NACH), STP/SWP registration & cancellation, client order payment status, change password | Flag, UserId, EncryptedPassword, pipe-delimited param string | Flag-specific pipe-delimited response (e.g., Mandate ID) |
| `StarMFWebService.svc` → `OrderStatus`, `AllotmentStatement`, `RedemptionStatement`, `MandateDetails`, `ChildOrderDetails`, `EMandateAuthURL`, `AOFPanSearch`, `GetAccessToken` | Status polling and reports (verified via public `/help` page of the service) | Member/user creds + date range / order no. / mandate no. | Order status, allotment (units, NAV, folio), redemption proceeds, mandate status, SIP child orders |
| `StarMFCommonAPI/ClientMaster/Registration` (JSON) | **Enhanced UCC registration** — create/modify client (the current method; older MFAPI flag-based UCC creation is deprecated) | ~180 fields: client code, holding type, tax status, PAN/PEKRN, KYC type, bank details (up to 5), demat, FATCA, nominee opt-in/out, contact | Status code, remarks (field-level rejections) |
| `BSEMFWEBAPI/api/2FAAuthController/_2FAAuthunetication/...` (JSON) | **Redemption 2FA**: get BSE OTP page URL to embed in app; OTP goes to client's mobile/email per AMC records | LoginID, Password, MemberCode, ClientCode, LoopbackReturnUrl, InternalRefNo | StatusCode (100/101), ReturnUrl (OTP page), PendingAuthOrder count |
| `BSEMFWEBAPI/api/_2FA_AuthenticationController/_2FA_Authentication/...` (JSON) | **Subscription (purchase) 2FA**: fetch BSE 2FA page for pending purchase orders; on completion BSE POSTs verified orders server-to-server | loginId, password, membercode, clientcode, primaryholder (1/2/3), internalrefno | responsestring (HTML page), statuscode; POST-back: Orders[] {orderno, Type, status e.g. "111" = all holders authenticated} |
| Single Integrated Payment API (JSON/SOAP) | One payment API for all modes — net banking, UPI, OTM/mandate debit; replaces deprecated Direct Payment Gateway API | Client code, order refs, payment mode, bank/UPI code, loopback URL | Payment URL / status; poll with Client Order Payment Status |
| Nomination API + nominee image upload; AOF image upload; scan-mandate image upload; cheque image upload (NRI/minor); SIP/XSIP pause; SIP-to-XSIP shift (JSON) | Supporting compliance/ops flows | Base64 or byte image payloads, naming conventions enforced | Status code, remarks |

**Example — authentication + purchase (condensed from official doc):**

```xml
POST https://bsestarmfdemo.bseindia.com/MFOrderEntry/MFOrder.svc/Secure   (SOAP 1.2 + WS-Addressing)
wsa:Action = http://bsestarmf.in/MFOrderEntry/getPassword
<bses:getPassword>
  <bses:UserId>123401</bses:UserId><bses:Password>mf@abc</bses:Password><bses:PassKey>abcdef1234</bses:PassKey>
</bses:getPassword>
-- response --
<getPasswordResult>100|UWQ/yLAS7KmiHyExtsKKq3DuHTWuyGOzy0P4s38PSL8nA==</getPasswordResult>

wsa:Action = http://bsestarmf.in/MFOrderEntry/orderEntryParam
<bses:orderEntryParam>
  <bses:TransCode>NEW</bses:TransCode><bses:TransNo>202202181234000002</bses:TransNo>
  <bses:UserID>123401</bses:UserID><bses:MemberId>1234</bses:MemberId>
  <bses:ClientCode>clientcode01</bses:ClientCode><bses:SchemeCd>02-DP</bses:SchemeCd>
  <bses:BuySell>P</bses:BuySell><bses:BuySellType>FRESH</bses:BuySellType><bses:DPTxn>P</bses:DPTxn>
  <bses:OrderVal>500</bses:OrderVal><bses:KYCStatus>Y</bses:KYCStatus><bses:EUINVal>N</bses:EUINVal>
  <bses:DPC>Y</bses:DPC><bses:Password>UWQ/...==</bses:Password><bses:PassKey>abc123</bses:PassKey>
</bses:orderEntryParam>
-- response --
NEW|202202181234000002|7531973|123401|1234|clientcode01|ORD CONF: Your Request for FRESH PURCHASE
500.000 in SCHEME: 02-DP ... ORDER NO: 7531973|0
```

**Example — subscription 2FA (JSON, from official doc):**

```json
POST .../BSEMFWEBAPI/api/_2FA_AuthenticationController/_2FA_Authentication/w
{ "loginId": "XXXXX", "password": "XXXXX", "membercode": "XXXX",
  "clientcode": "456fefe", "primaryholder": "1", "internalrefno": "123456987" }
→ { "responsestring": "<HTML OTP page>", "statuscode": "100", "internalrefno": "123456987" }
BSE later POSTs to member's registered URL:
{ "statuscode": "100", "internalrefno": "123456987",
  "Orders": [ { "orderno": "8739152", "Type": "Purchase", "status": "111" } ] }
```

**BSE StAR MF v2 / newer REST APIs (partially verified).** BSE has been rolling out a newer "StARMF v2" JSON/REST API stack; third-party evidence exists (a public Postman workspace "BSE StARMF v2 API" by Remiges Tech, and a circulating "bse-starmfv2-api V0.9.x" spec), and several fintechs reference it. The official v2 spec is **member-distributed, not public** — endpoint names, auth model, and coverage must be confirmed with BSE after membership. "UDP APIs" could not be verified in any public BSE documentation as of July 2026; treat as unverified. Plan integration against the documented SOAP/JSON v3.5 stack and ask BSE for the v2 kit during onboarding.

## Auth

- **Two credential layers:** (1) BSE-issued Member ID + User ID (web-service login) + password; (2) per-session **encrypted password** obtained via `getPassword(UserId, Password, PassKey)` where PassKey is any random alphanumeric string you supply (entropy). Response `100|<encrypted password>`; `101` on failure.
- **Session validity:** 1 hour for `MFOrder.svc`; **5 minutes for all other services** — regenerate liberally; the encrypted password is unique per login and must accompany every order.
- **Password rotation:** the underlying login password expires periodically (documented error: `PASSWORD EXPIRED`); rotate via the Change Password flag of the Additional Services `MFAPI`. Five wrong attempts locks the user (`MAXIMUM LOGIN ATTEMPTS`). Build rotation + lockout alerting into the backend.
- Always use the `/Secure` (HTTPS) endpoints from the `?singleWsdl` definitions; SOAP 1.2 with WS-Addressing headers (`wsa:Action`, `wsa:To`) is mandatory.
- Some newer JSON services use their own login-per-call model (2FA APIs take login/password in the body); UCC/image APIs have separate auth per section of the spec.

## Webhooks / callbacks

- **There are no push webhooks for order lifecycle events** in the documented stack. Order/allotment/redemption status is **pull-based**: poll `OrderStatus`, `AllotmentStatement`, `RedemptionStatement`, `MandateDetails`, `ChildOrderDetails` APIs, and/or download the exchange's end-of-cycle feedback files (order/allotment/redemption feedback per the WEB file structures document). Allotment feedback typically lands T+1 after funds reach the AMC; redemption confirmations follow the RTA cycle.
- **Two documented exceptions (callback-ish):** (1) the Subscription 2FA flow performs a **server-to-server POST** of client-verified orders to the member's registered post URL when the client completes the OTP journey; (2) payment flows use loopback/return URLs after net-banking/UPI journeys (payment status itself should still be confirmed via the Client Order Payment Status API).
- Practical design for Yesly: a polling worker (e.g., Celery/EventBridge schedule) reconciling order → payment → allotment states, plus idempotent handlers for the 2FA POST-back.

## Sandbox

- **Yes — UAT/demo environment exists:** `https://bsestarmfdemo.bseindia.com` (order entry demo WSDL: `.../MFOrderEntry/MFOrder.svc?singleWsdl`; UCC demo: `.../StarMFCommonAPI/ClientMaster/Registration`). A demo scheme master is exposed at `bsestarmfdemo.bseindia.com/RptSchemeMaster.aspx`.
- **Access requires membership.** Official go-live path (v3.5 doc): (1) request API docs + Test Market credentials by email quoting your live BSE StAR MF Member ID (contacts in the doc are @bsetech.in addresses); (2) build against Test Market; (3) request a demo of your product to the Exchange; (4) on approval receive Live API links; (5) go live. BSE recommends SoapUI for testing (Postman works but you hand-craft XML).
- No self-serve public sandbox, no public API keys. Budget the membership + demo-approval sequence into the project timeline.

## Pricing

To be re-verified with BSE at onboarding — BSE revises this via notices and much of the current member fee schedule is not on public pages:

- **End investor: zero.** BSE StAR MF levies nothing on investors; RIA clients pay only Yesly's advisory fee and the scheme's direct-plan TER.
- **Membership/registration:** MFD limited-purpose membership was launched with a one-time fee of Rs 15,000 (2013); BSE later ran waivers (2018-19: registration free, ~Rs 300 admin/processing charge). Current MFD/RIA registration fee schedule: confirm with BSE membership desk (unverified publicly as of July 2026).
- **Per-transaction charges:** since Dec 2016 BSE has charged **AMCs** (not distributors/RIAs) roughly Rs 6-30 per transaction on volume slabs; BSE has publicly maintained no transaction fee on distributors. BSE's StAR MF revenue (₹231 crore FY25 per press reports) is AMC-funded. Historically proposals to charge members ~Rs 5-10/order were rolled back; verify the current member-side schedule in BSE notices before modeling costs.
- **Adjacent unavoidable costs for an RIA:** BASL (BSE Administration & Supervision Ltd) membership is mandatory for all SEBI RIAs since 2021 — Rs 1 lakh initial / ~Rs 99,000 renewal for corporates/LLPs (Rs 2,000/1,800 for individuals). Payment-rail costs (UPI/net-banking PG, e-NACH mandate registration fees) may apply per bank arrangements.

## Compliance notes (India / SEBI)

- **2FA is SEBI-mandated, and BSE provides the rails.** SEBI circular of 31 Mar 2022 mandated 2FA for **redemptions** (effective, after extension, 1 Oct 2022); circular SEBI/HO/IMD/IMD-I DOF1/P/CIR/2022/132 of 30 Sep 2022 extended it to **subscriptions effective 1 Apr 2023**. One factor must be an OTP to the unit-holder's email/phone registered with the AMC/RTA. For systematic transactions (SIP/STP/SWP), 2FA applies **only at registration**, not each instalment. These requirements are consolidated in SEBI's Master Circular for Mutual Funds (June 2024) and remain in force as of mid-2026 (no public circular found relaxing them; re-verify at build time). Implementation on StAR MF = the Redemption 2FA and Subscription 2FA APIs above: Yesly must embed BSE's OTP page in the app; orders are not processed until the client authenticates.
- **NAV by funds realisation:** SEBI circulars of 17 Sep 2020 and 31 Dec 2020, effective **1 Feb 2021** — purchase NAV is the day funds are **credited to the scheme account before the scheme cutoff** (3:00 pm most schemes; 1:30 pm liquid/overnight), regardless of order time. On StAR MF, the exchange applies earlier internal cutoffs so ICCL can pass funds to the AMC in time; late payment = next-day (or later) NAV. Set client expectations in the app.
- **No pooling:** SEBI's 2021-22 circulars discontinued pooling of client funds/units by platforms/distributors/IAs (effective 1 Jul 2022). StAR MF's ICCL nodal-account model is the compliant architecture — Yesly must never intermediate funds.
- **RIA model:** direct plans only, no commission/EUIN-based earnings (EUIN declaration goes as N/blank for RIA orders); advisory fee charged separately under the IA Regulations. BSE's RIA framework restricts the RIA to onboarding + order placement. Serving **non-advisory** users with direct-plan execution would trigger SEBI's Execution Only Platform (EOP) framework (2023) — for advisory clients of the RIA, EOP registration is not required.
- **UCC hygiene is regulatory:** PAN-Aadhaar-linked, **KYC-validated** (KRA) status, FATCA/CRS declaration, nominee opt-in/opt-out, and clean bank details are prerequisites — orders reject otherwise. NRI clients need correct tax status (NRE/NRO), FATCA, and (for some cases) cheque-image upload per the NRI/minor API.
- Data residency: BSE/ICCL are Indian entities; no cross-border data issue. Keep client PII handling aligned with the DPDP Act.

## Gotchas & effort

- **SOAP from Python:** order entry is SOAP-1.2-only with WS-Addressing. `zeep` works (community libraries confirm) but needs explicit `wsa` plugin/headers, the `/Secure` port from `?singleWsdl`, and careful service/port selection (`MFOrder` / `WSHttpBinding_MFOrderEntry1`). Responses are pipe-delimited strings inside XML — write a robust parser and treat `Success flag != 0` + remark text as the source of truth.
- **Session churn:** 1-hour (orders) / 5-minute (everything else) encrypted-password validity; plus periodic login-password expiry (`PASSWORD EXPIRED`) and 5-attempt lockout. Automate `getPassword` refresh and scheduled password rotation.
- **Cutoff choreography:** same-day NAV needs order + successful payment early enough for ICCL→AMC transfer before scheme cutoff. UPI/net-banking is same-day if early; mandate (OTM) debits and cheque modes add days. Surface "expected NAV date" logic in UX.
- **Mandate lead times kill naive SIP UX:** physical mandates take ~2-3 weeks to register; e-NACH via `EMandateAuthURL` is days but bank-dependent; UPI autopay mandates are fastest but amount-capped (~Rs 1 lakh, bank/NPCI dependent). First SIP debit date must respect mandate-activation buffers.
- **XSIP auto-triggers instalments on the exchange side** once registered — you don't place instalment orders. Track them via `ChildOrderDetails`; reconcile failures (mandate bounce) yourself.
- **No push notifications:** all order/allotment/redemption state is polled or file-based, with T+1-style delays on allotment feedback; build a reconciliation worker and idempotent state machine, not request/response assumptions.
- **UCC rejections are the #1 operational drag:** ~180-field client master with strict enums (tax status x account type matrix, occupation codes, state/country codes); KYC not "validated" at KRA, bank IFSC/name mismatches, or missing nominee choice all bounce orders. Validate aggressively before submission.
- **Modification/cancellation of lumpsum orders is gone** (real-time RTA integration) — only NEW; build client-side confirmation (plus mandatory 2FA) before submission.
- **v2 REST docs are member-only**; do not architect around unverified endpoints.
- **Effort estimate:** membership + BASL paperwork and BSE product demo approval: ~4-8 weeks elapsed. Engineering: UCC onboarding + order entry + payments + 2FA embed + SIP/mandates + reconciliation ≈ 6-10 engineer-weeks on the FastAPI backend, plus app-side webview flows for 2FA/payments. Consider an aggregator layer (e.g., MFCentral-adjacent vendors or BSE-integrated ASPs) only if membership timelines block launch — at the cost of margin and control.

## Sources

- https://bsestarmf.in/ (official platform site)
- https://www.bsestarmf.in/APIFileStructure.pdf (official "MF - Web Services Structure v3.5, Aug 2023" — endpoints, auth, order/SIP/2FA/UCC/payment structures; primary source for API details above)
- https://bsestarmf.in/WEBFileStructure.pdf (official file structures for web/file-based feedback)
- https://www.bsestarmf.in/StarMFWebService/StarMFWebService.svc/help (live list of additional-service operations)
- https://www.bseindia.com/Static/Markets/MutualFunds/registered_invest_advisors.aspx (BSE — RIA framework on StAR MF)
- https://www.bseindia.com/Static/Markets/MutualFunds/MFI.aspx (BSE — MFI segment page)
- https://www.bseindia.com/static/members/Mutual_Fund_Distributor_Registration.aspx and https://www.bseindia.com/downloads1/MFD%20Registration%20FAQs.pdf (MFD membership)
- https://bsestarmfdemo.bseindia.com/RptSchemeMaster.aspx (demo/UAT environment)
- https://www.sebi.gov.in/legal/circulars/sep-2022/two-factor-authentication-for-transactions-in-units-of-mutual-funds_63557.html (SEBI 2FA circular, 30 Sep 2022)
- https://www.sebi.gov.in/legal/master-circulars/jun-2024/master-circular-for-mutual-funds_84441.html (SEBI Master Circular for Mutual Funds, June 2024)
- https://www.amfiindia.com/investor/knowledge-center-info?zoneName=CutOffTimingsAndNewRuleOnApplicableNAV (NAV realisation rule, effective 1 Feb 2021)
- https://cafemutual.com/news/industry/7016-bse-starts-levying-transaction-fee-on-its-bse-star-mf-platform and https://cafemutual.com/news/industry/7282-no-transaction-fee-from-distributors-on-bse-star-mf-platform (transaction fee history)
- https://cafemutual.com/news/industry/2788-bse-opens-membership-for-mutual-fund-distributors (Rs 15,000 lifetime MFD fee, 2013)
- https://cafemutual.com/news/industry/13047-bse-star-mf-and-nse-nmf-ii-to-waive-off-registration-fee-to-attract-distributors (fee waivers)
- https://www.bseindia.com/downloads1/FAQs_BASL_Membership.pdf and https://cafemutual.com/news/industry/22235-rias-have-to-become-member-of-bses-arm-before-august-31-2021-sebi (BASL membership for RIAs)
- https://www.business-standard.com/markets/news/bse-star-mf-rides-digital-investment-growth-amid-equity-market-rally-125070801497_1.html (scale: 7.13 crore txns Oct 2025, FY25 volumes/revenue)
- https://github.com/utkarshohm/mf-platform-bse (community Python/zeep integration; SIP auto-trigger behavior)
- https://www.postman.com/remiges-tech/bse-starmf-v2-api/overview (third-party evidence of StARMF v2 API; official v2 spec is member-only — unverified)
