# 04 — Broker Execution: smallcase Gateway

> Research date: July 2026. Sources: official docs at developers.gateway.smallcase.com and gateway.smallcase.com. Where details are behind smallcase's commercial onboarding (notably pricing and the full broker-feature matrix), that is stated explicitly rather than guessed.

## Overview & role in Yesly

**What it is.** smallcase Gateway is a B2B transaction SDK + API layer from smallcase Technologies that lets a third-party platform (like Yesly) place equity transactions — single stocks, ETFs, REITs/InvITs, and multi-stock baskets — through the **user's own broker account**, across most large Indian brokers, without the platform itself becoming a broker or touching client funds/securities. It also supports importing a user's demat holdings (with consent), mutual fund holdings import, broker account opening, and fetching funds. smallcase claims 10M+ users reached via Gateway partners.

**Role in Yesly (Step 30 of the flow — trade execution).**

- **v1:** Yesly deep-links the user out to their own broker app to execute the recommended trades. No Gateway dependency; zero execution infrastructure; but no order feedback loop.
- **v1.5:** in-app execution with an order ledger + reconciliation. smallcase Gateway is the candidate here: Yesly's FastAPI backend creates a transaction intent, the React Native SDK opens the broker flow in-app, the user authorizes at their broker, and Yesly gets back order status (via SDK response + webhook) to write into its order ledger and reconcile against the recommendation.
- Yesly is **not** the broker of record. Individual stocks execute in the user's own demat/broker account. Yesly (House of Alpha, SEBI RIA INA000018364) stays advice-only; Gateway keeps execution at the user's broker.

## How it works

Text flow (v1.5 in-app execution):

```
Yesly app (React Native)                Yesly backend (FastAPI)                smallcase Gateway                 User's broker
        |                                        |                                    |                              |
        |-- user taps "Execute plan" ----------->|                                    |                              |
        |                                        |-- POST /gateway/<name>/transaction |                              |
        |                                        |   headers: x-gateway-secret,       |                              |
        |                                        |            x-gateway-authtoken(JWT)|                              |
        |                                        |   body: intent=TRANSACTION,        |                              |
        |                                        |         orderConfig (securities)   |                              |
        |                                        |<------- { transactionId, expireAt }|                              |
        |<-- transactionId ----------------------|                                    |                              |
        |                                        |                                    |                              |
        |-- SmallcaseGateway.init(sdkToken) ------------------------------------------>                              |
        |-- triggerTransaction(transactionId) ---------------------------------------->                              |
        |         SDK shows broker chooser -> user picks broker -> broker login ------|----- OAuth/TOTP/PIN -------->|
        |         user reviews & confirms order screen --------------------------------|----- order placed at broker>|
        |<-- txn response (orderBatches, statuses, smallcaseAuthToken) ---------------|<---- fill/reject from RMS ---|
        |                                        |                                    |                              |
        |                                        |<== webhook/postback: order update ==                              |
        |                                        |-- (optional) GET order details ---->                              |
        |                                        |-- write order ledger, reconcile     |                              |
```

Key points:

1. **Server-side intent.** Every user interaction (order, holdings import, broker connect, fetch funds) starts with a backend call to the Create Transaction API, which returns a unique `transactionId`. This must be done server-side so the gateway secrets are never in the app.
2. **Client-side SDK.** The `transactionId` is passed to the SDK's `triggerTransaction`. The SDK renders broker chooser, broker login, order confirmation, and status screens.
3. **Order at broker.** The order is placed in the user's own broker account against their own funds and demat; broker-side RMS can still reject it.
4. **Feedback.** Order status comes back in the SDK promise resolution and (recommended for the ledger) via server-side webhooks/postbacks, plus a fetch-details API.

## Supported intents

| Intent | What it does |
|---|---|
| `TRANSACTION` | Buy/sell securities (single stocks, ETFs, REITs/InvITs, or a basket of mixed BUY/SELL legs in one transaction). For smallcase-branded baskets, order types include INVESTMORE / FIX / REBALANCE / SIP / MANAGE / EXIT with an `scid`. |
| `HOLDINGS_IMPORT` | Import the user's demat equity holdings (stocks, ETFs, REITs/InvITs, smallcases) with one-time in-flow consent via broker login. |
| `CONNECT` | Connect/login a broker account without transacting (docs call this "broker connect"; the task brief's "AUTHORISE" corresponds to this intent). |
| `MF_HOLDINGS_IMPORT` | Import mutual fund holdings (demat and non-demat), via SDK flow or DIY API with PAN/phone/email in `assetConfig`. |
| Fetch funds | Documented flow to fetch the user's available broker funds (server-side, for a connected user). |
| SIP (smallcase orders) | SIP appears as an order type on smallcase-basket transactions; systematic repeat orders on plain securities baskets would be Yesly-side scheduling that creates a fresh `TRANSACTION` each time. |

Note: the exact enum spelling per endpoint should be confirmed against the API reference during onboarding; docs consistently show `TRANSACTION`, `HOLDINGS_IMPORT`, `CONNECT`, and `MF_HOLDINGS_IMPORT`.

## Broker coverage

Broker keys listed in the official Broker ⇔ Feature Map page (as researched): `aliceblue` (Alice Blue), `angelbroking` (Angel One), `axis` (Axis Securities), `dhan` (Dhan), `edelweiss` (Nuvama, formerly Edelweiss), `fisdom`, `fivepaisa` (5paisa), `fundzbazar` (FundzBazar/Prudent), `groww` (Groww), `hdfc` (HDFC Securities), `hdfcsky` (HDFC Sky), `icici` (ICICIdirect), `iifl` (IIFL), `kite` (Zerodha), `kotak` (Kotak Securities), `motilal` (Motilal Oswal), `paytm` (Paytm Money), `sbi` (SBI Securities), `trustline`, `upstox` (Upstox).

**Important caveats:**

- **Feature support varies per broker.** The detailed feature-by-broker matrix lives in a Google Sheet linked from the docs and could not be fully verified here. Not every broker in the key list supports every intent. Example verified from docs: standard demat **holdings import** is listed for 5paisa, Angel One, Dhan, Fisdom, HDFC Sky, IIFL, Motilal Oswal, Trustline, Upstox and Zerodha — i.e., a subset. `groww` exists as a broker key in SDK config, but Groww is **not** listed among holdings-import-supported brokers, and its support for securities transactions via Gateway could not be verified from public docs — confirm directly with smallcase before assuming Groww coverage. Given Groww's large retail share, this is a material coverage question for Yesly.
- The broker list changes over time (brokers get added/removed); re-verify the live Broker ⇔ Feature Map at contract time.

## Key endpoints / SDK calls

Base URLs seen in official docs: production `https://gatewayapi.smallcase.com`, staging `https://gatewayapi.stag.smallcase.com`.

| Name | Purpose | Key request fields | Key response fields |
|---|---|---|---|
| `POST /gateway/<gatewayName>/transaction` (Create Transaction) | Create a transaction intent server-side; one call per user interaction | Headers: `x-gateway-secret` (API secret), `x-gateway-authtoken` (JWT). Body: `intent` (`TRANSACTION` / `HOLDINGS_IMPORT` / `CONNECT` / `MF_HOLDINGS_IMPORT`), `orderConfig` (for TRANSACTION: `type` e.g. SECURITIES, `securities[]` with ticker/`transactionType` BUY-SELL/quantity, optional `notes` up to 256 chars), `assetConfig` (for MF import: PAN, phone, email) | `transactionId`, `expireAt` (transaction validity) |
| Fetch order / transaction details (server API) | Poll order status after SDK flow, for the order ledger | `transactionId` or batch identifier; gateway auth headers | `orderBatches[]`: batch status (`PLACED`, `COMPLETED`, `PARTIALLYFILLED`, `UNFILLED`, `UNPLACED`); per-order: `status` (`COMPLETE`, `PARTIAL`, `REJECTED`, `CANCELLED`), `tradingsymbol`, `transactionType`, `orderType`, `filledQuantity`, `averagePrice` |
| Fetch Holdings (server API) | Retrieve imported holdings for a connected user | Connected-user `x-gateway-authtoken` (JWT containing `smallcaseAuthId`) | `securities[]` (holdings/positions across exchanges, quantities, average price, company details), `smallcases` (public/private, with constituents), `snapshotDate` (freshness) |
| Fetch funds (server API) | User's available broker funds for a connected user | Connected-user auth token | Available funds figure (shape per API reference) |
| SDK `setConfigEnvironment(...)` | Configure SDK before init | `environmentName` (`SmallcaseGateway.ENV.PROD`; staging/dev enums exist), `gatewayName`, `isLeprechaun` (test mode), `isAmoEnabled` (after-market orders), `brokerList` (restrict broker chooser) | — |
| SDK `init(sdkToken)` | Start a gateway session for the user | `sdkToken` = the JWT (guest or connected) from your backend | init response / error in `err.userInfo` |
| SDK `triggerTransaction(transactionId)` | Run the actual flow (broker chooser, login, order confirm) | `transactionId` from backend | Transaction response incl. order result and, on first connect, the connected-user auth data (`smallcaseAuthToken`/`smallcaseAuthId`) |
| SDK `logoutUser()` / `showOrders()` / `triggerLeadGen(userInfo)` | Logout broker session; native order screen; broker account-opening lead flow | — | — |

Example — create a stocks order transaction (shape assembled from official docs; confirm exact field names against the API reference under NDA/onboarding):

```json
POST https://gatewayapi.smallcase.com/gateway/yesly/transaction
Headers:
  x-gateway-secret: <API_SECRET>
  x-gateway-authtoken: <JWT signed HS256 with gateway secret>

{
  "intent": "TRANSACTION",
  "orderConfig": {
    "type": "SECURITIES",
    "securities": [
      { "ticker": "RELIANCE", "type": "BUY",  "quantity": 5 },
      { "ticker": "TCS",      "type": "SELL", "quantity": 2 }
    ]
  },
  "notes": "yesly-plan-8842"
}

Response (documented fields): { "transactionId": "...", "expireAt": "..." }
```

React Native SDK (package `react-native-smallcase-gateway`, aka scgateway; verbatim patterns from official docs):

```javascript
import SmallcaseGateway from "react-native-smallcase-gateway";

await SmallcaseGateway.setConfigEnvironment({
  isLeprechaun: false,
  isAmoEnabled: true,
  gatewayName: "<YOUR_GATEWAY_NAME>",
  environmentName: SmallcaseGateway.ENV.PROD,
  brokerList: [],
});

await SmallcaseGateway.init(sdkToken);           // sdkToken = JWT from Yesly backend

SmallcaseGateway.triggerTransaction(transactionId)
  .then((txnResponse) => { /* write to order ledger, show status */ })
  .catch((err) => { /* err.userInfo has error details */ });
```

SDK platform requirements (from docs, mid-2026): React Native SDK v7.2.3+ targets RN 0.7x lines, iOS deployment target 13+ built with current Xcode, Android minSdk 21 / compileSdk 33+, Gradle 7.6.x. Android needs an intent-filter for the `scgateway://<gatewayName>` redirect scheme.

## Auth

- **Credentials (issued by smallcase business team at onboarding):** a `gatewayName` (unique integration identifier), a **Secret** (used to HS256-sign JWTs), and an **API secret** (sent as `x-gateway-secret` header on server APIs). None of these may reach the client.
- **JWT (`x-gateway-authtoken` / SDK `sdkToken`):** created on Yesly's backend, HS256-signed with the Secret, short expiry, generated on the fly:
  - **Guest user token:** payload `{ "guest": true, "exp": ... }` — for a first-time user who hasn't connected a broker; the SDK will show the broker chooser.
  - **Connected user token:** payload contains `smallcaseAuthId` (the stable identifier smallcase creates for a user+broker-account pairing on first successful flow) + expiry. Using it skips broker selection for returning users. Yesly must persist `smallcaseAuthId` against its user record; two Yesly users may only share a `smallcaseAuthId` if they genuinely share a broker account, and mixing them yields a `user_mismatch` error.
- **Broker session persistence:** the actual broker login session is owned by the broker, not by smallcase — the docs are explicit that the smallcase auth token "cannot be used to maintain/refresh the user's broker login session" and Gateway does not maintain broker sessions server-side. The SDK checks for an active session on the smallcase platform and skips login if one exists; otherwise the user re-authenticates at the broker (for many brokers this is effectively a daily/session-based re-login, per each broker's own session policy).

## Webhooks / callbacks

- **Events:** securities transaction order updates (placed/completed), holdings import completion, MF holdings import completion.
- **Setup:** Yesly exposes a public HTTPS endpoint (FastAPI route on AWS Mumbai) and shares the URL with the smallcase Gateway team for registration — there is no self-serve webhook config.
- **Payloads:** stock order webhooks carry `batchId`, transaction details, and an `orders[]` array with `status`, `tradingsymbol`, `transactionType`, quantities. Holdings-import webhooks (v2) carry the full `securities[]` and `smallcases` structures.
- **Verification:** each payload includes a `checksum` — SHA256 over `timestamp + smallcaseAuthId` keyed with your API_SECRET (for MF holdings import: `timestamp + transactionId`). Recompute and compare server-side before trusting the payload.
- **Retries:** 3 retries on delivery failure — after 15 minutes, after 24 hours, and finally after 48 hours. Design the ledger writer to be idempotent (dedupe on `batchId`/`transactionId`).
- **Client callback:** in parallel, `triggerTransaction`'s promise resolves with the transaction result in-app; treat the webhook as the source of truth for reconciliation (the app can be killed mid-flow).

## Sandbox

- **Staging environment:** `https://gatewayapi.stag.smallcase.com`, with the demo gateway name `gatewaydemo` used throughout the docs' examples (e.g. `POST /gateway/gatewaydemo/transaction`). SDK enums include non-prod environments; docs' leprechaun guidance notes `environmentName` stays PROD when testing with leprechaun tokens on the production environment.
- **Leprechaun mode (test brokers):** set `isLeprechaun: true` and use **virtual tokens ("leprechauns") issued by the smallcase integration team** — broker login is bypassed, each token acts as a separate broker account with virtual funds, market timings are simulated (orders work outside real market hours), and special tokens simulate edge cases (low funds, order failures). Leprechaun mode exists for most brokers **except** (per docs) Dhan, Fisdom, HDFC Sky, Groww and Motilal Oswal.
- Practical note: there is no fully self-serve sandbox — you need smallcase to issue gateway credentials and leprechaun tokens, so a commercial conversation precedes any hands-on POC beyond reading docs.

## Pricing

**Not public.** smallcase Gateway pricing is a negotiated commercial B2B agreement; neither the developer docs nor gateway.smallcase.com publish per-transaction, per-user, or platform fees, and no verifiable public ballpark for the B2B Gateway product was found in this research. Points that *are* verifiable:

- On smallcase's own consumer platform, users pay roughly Rs. 100 + GST per lump-sum smallcase-basket transaction and Rs. 10 + GST per SIP installment (varies by broker) — this is **consumer-side pricing for smallcase baskets**, not what a Gateway partner pays, but it hints that per-transaction fee mechanics exist in the ecosystem.
- Expect the Gateway deal to involve some mix of integration/platform fee and per-transaction or per-connected-user charges; treat any number as unknown until quoted. Budget assumption for Yesly planning: mark as **TBD pending smallcase commercial discussion**, and get the fee model in writing before committing v1.5 architecture to Gateway.

## Compliance notes (India / SEBI)

- **Execution-only via the user's own broker.** Orders are placed in the user's own trading account and settle into the user's own demat. Yesly never holds, pools, or routes client funds or securities — consistent with SEBI (Investment Advisers) Regulations, 2013, which prohibit an RIA from pooling client assets and require segregation of advisory from execution/distribution.
- **Advice vs execution separation.** Yesly (RIA INA000018364) issues the advice; execution is the client's action at their own broker, facilitated by a neutral technology layer. SEBI's IA framework permits advisers to offer execution "through direct schemes/products" without charging for it; the key constraints are no commissions/consideration linked to execution flowing to the RIA in ways that conflict with IA regulations, clear client consent, and no obligation on the client to execute through the offered channel. Have compliance counsel confirm the Gateway arrangement (and any revenue share, if smallcase offers one) against current IA regulations and House of Alpha's compliance policy — a fee *paid to* smallcase is cleaner for an RIA than revenue *received from* execution.
- **Consent trails.** Holdings import happens only after explicit in-flow user consent (broker login + authorization screen); order placement requires the user to review and confirm on the broker's order screen with 2FA per broker/exchange norms. Keep Yesly-side records: recommendation, rationale, transactionId, order outcome — this doubles as the RIA record-keeping trail.
- **Broker of record.** The broker remains responsible for KYC, RMS, margins, contract notes, and settlement. smallcase Technologies operates the Gateway as a technology platform (smallcase itself holds SEBI registrations for its own businesses; the Gateway partner does not inherit any broking status — nor any broking obligations).
- **Data:** holdings data is personal financial data obtained with consent; store it per Yesly's privacy policy and (as applicable) DPDP Act 2023 obligations, in the AWS Mumbai region.

## Gotchas & effort

1. **Broker coverage gaps.** Feature support is per-broker and the authoritative matrix is a non-public spreadsheet. Groww's transaction support via Gateway is unverified publicly (and Groww lacks leprechaun test mode and isn't in the holdings-import list) — if a large share of Yesly's users are Groww-first, v1 deep-linking may remain the only path for them. Verify the live matrix per intent before committing.
2. **Broker session expiry.** Broker login sessions are broker-owned; most brokers require a fresh client-side login per session/day. Every Yesly "execute" action must tolerate a login interstitial; don't design UX that assumes silent order placement.
3. **Holdings freshness.** All brokers except Zerodha need a client-side broker login to refresh holdings; check `snapshotDate` and re-trigger `HOLDINGS_IMPORT` when stale. Holdings are a snapshot, not a live feed — reconcile Yesly's ledger accordingly (and remember users can trade directly in their broker app, invalidating Yesly's picture).
4. **Order rejections and partial fills.** Broker RMS can reject for insufficient funds/margin/blocked scrips; basket orders return per-order statuses (`COMPLETE`/`PARTIAL`/`REJECTED`/`CANCELLED`) inside `orderBatches`. The v1.5 order ledger must model partial execution of a recommendation, retries for `UNPLACED`/`UNFILLED` legs, and user-abandoned flows (transactionId has an `expireAt`).
5. **Reconciliation is on Yesly.** Gateway gives order-level status, not portfolio accounting. Combine webhook events + fetch-details polling + periodic holdings import; make webhook processing idempotent (3-retry delivery means duplicates are possible).
6. **SDK churn.** Native SDKs (iOS/Android under the RN wrapper) rev frequently with Xcode/Gradle/RN version requirements; pin versions, watch the changelog, and budget maintenance time each Expo/RN upgrade. Note the RN SDK requires native modules — Expo **managed** workflow needs a dev client / prebuild (config plugin or bare workflow), not Expo Go.
7. **Commercial and dependency risk.** Credentials, webhook registration, leprechaun tokens, and the broker matrix all flow through smallcase's business/integration team — no self-serve. Yesly's execution layer would depend on smallcase's commercial priorities, broker relationships (brokers have been added and removed over time), and uptime. Keep v1's deep-link-out path alive as the fallback even after v1.5 ships.
8. **Effort estimate.** Backend: JWT minting, create-transaction wrapper, webhook receiver with checksum verification, order ledger + reconciliation jobs — roughly 2–3 engineer-weeks. App: SDK config plugin setup, init/trigger wiring, status UX including login/rejection/partial states — roughly 1–2 engineer-weeks. Plus commercial onboarding lead time with smallcase (credentials + leprechaun tokens) before any end-to-end testing, which can dominate the calendar.

## Sources

- https://developers.gateway.smallcase.com/docs/overview — Gateway overview and modules
- https://developers.gateway.smallcase.com/docs/getting-started — credentials, JWT (guest/connected), transaction flow, leprechaun intro
- https://developers.gateway.smallcase.com/docs/securities-transactions — order config, orderBatches/status model
- https://developers.gateway.smallcase.com/docs/holdings-import — holdings import flow, snapshotDate freshness caveat
- https://developers.gateway.smallcase.com/docs/broker-feature-map — broker keys list; detailed matrix in linked (non-public) sheet
- https://developers.gateway.smallcase.com/reference/connect-transactionid — Create Transaction endpoint URL, headers, CONNECT intent
- https://developers.gateway.smallcase.com/docs/webhooks — webhook events, checksum verification, retry schedule
- https://developers.gateway.smallcase.com/page/user-broker-sessions — smallcaseAuthId semantics, user_mismatch
- https://developers.gateway.smallcase.com/page/broker-logout and getting-started FAQs — broker-owned sessions, no server-side session maintenance
- https://developers.gateway.smallcase.com/page/using-virtual-tokens-leprechaun-for-testing — leprechaun mode, excluded brokers
- https://developers.gateway.smallcase.com/docs/using-react-native-sdk — RN SDK (react-native-smallcase-gateway) code snippets and requirements
- https://developers.gateway.smallcase.com/docs/mutual-fund-holdings-import-via-api — MF_HOLDINGS_IMPORT intent, assetConfig
- https://developers.gateway.smallcase.com/page/fetch-funds — fetch funds flow, gatewayapi URLs (incl. gatewayapi.stag.smallcase.com / gatewaydemo)
- https://gateway.smallcase.com/ — product page, feature set, scale claims
- https://www.smallcase.com/learn/smallcase-fees-and-charges/ — consumer-side smallcase transaction fees (context only; not Gateway B2B pricing)
- https://github.com/smallcase/react-native-smallcase-gateway and https://www.npmjs.com/package/react-native-smallcase-gateway — RN package
