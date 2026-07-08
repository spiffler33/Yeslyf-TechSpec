# 16. Zerodha Kite Connect — Direct Equity Trading API

**Docs:** https://kite.trade/docs/connect/v3/ · **Console:** https://developers.kite.trade
**Used at:** Direct equity (stocks/ETF) order flow in Yesly for users who hold a Zerodha account — alongside smallcase Gateway (basket/portfolio transactions) and BSE StAR MF (mutual funds).

---

## 1. What it is & where it fits

Kite Connect is Zerodha's REST-like HTTP API set for building trading and investment platforms on top of Zerodha's brokerage: session/login, order placement (equity, F&O, commodity, MF), portfolio (holdings/positions), market quotes, historical candles, WebSocket streaming, and order postbacks (webhooks).

Fit for Yesly (SEBI-registered RIA):

- **Every end user must have a Zerodha trading account.** Kite Connect does not open accounts or execute for non-Zerodha users. The app connects the user's own Zerodha account via an OAuth-like login; orders go into their own demat — no pooling, fully broker-native.
- **Two product tiers:**
  - **Kite Publisher** — free JS/deep-link widget: your app pre-fills a basket, the user reviews and confirms on Zerodha's own screen. No token exchange, no read APIs. Because the *user* places the order manually, Publisher does not fall under SEBI's algo framework.
  - **Kite Connect (full)** — paid API app: your backend places/modifies/cancels orders and reads holdings/positions/quotes on the user's behalf after the daily login.
- **RIA/order-on-behalf caution:** Zerodha's terms make the platform builder responsible for exchange/SEBI compliance. Fully automated order generation (orders fired without a per-order user action) is treated as algo trading and requires exchange approval / strategy registration (Zerodha assists with this). The safe RIA pattern is *advice in Yesly → user-confirmed execution* (Publisher, or Connect orders triggered by an explicit user tap), not unattended rebalancing.
- Zerodha also runs a **Connect Startup/platform partnership program** for fintechs building on the API (revenue/fee arrangements negotiated directly) [verify].

## 2. Integration flow (login → trade)

1. Create an app on developers.kite.trade → get `api_key` + `api_secret`, register the **redirect URL** (backend endpoint) and optional **postback URL** (HTTPS).
2. App opens `https://kite.zerodha.com/connect/login?v=3&api_key=xxx` (in-app browser). User logs in with Zerodha credentials + TOTP 2FA (mandatory on the account).
3. Zerodha redirects to your registered redirect URL with `request_token` (short-lived, single-use).
4. Backend POSTs `/session/token` with `api_key`, `request_token`, and `checksum = SHA-256(api_key + request_token + api_secret)` → receives `access_token` + user profile/exchange permissions. **api_secret never leaves the backend** — mobile apps must proxy through FastAPI.
5. Store `access_token` server-side (encrypted; DPDP: it's a credential to the user's brokerage). All calls use header `Authorization: token api_key:access_token`.
6. Trade/read via REST; receive async order updates on the postback URL (verify payload checksum `SHA-256(order_id + order_timestamp + api_secret)`).
7. Token **expires at ~6:00 AM IST next day** (regulatory flush) — user must redo the login flow every trading day. Design the UX around this.
8. Logout: `DELETE /session/token?api_key=...&access_token=...`.

## 3. Key endpoints

Base: `https://api.kite.trade`

| Call | Purpose | Key inputs → outputs |
|---|---|---|
| GET `kite.zerodha.com/connect/login?v=3&api_key=` | Start login | → redirect with `request_token` |
| POST `/session/token` | Exchange token | api_key, request_token, checksum → `access_token`, user profile |
| DELETE `/session/token` | Logout / invalidate | api_key, access_token → success |
| POST `/orders/:variety` | Place order (`regular`, `amo`, `co`, `iceberg`, `auction`) | tradingsymbol, exchange, transaction_type, order_type (MARKET/LIMIT/SL/SL-M), quantity, product (CNC/MIS/NRML/MTF), price, trigger_price, validity, tag → `order_id` |
| PUT `/orders/:variety/:order_id` | Modify pending order | changed fields → order_id |
| DELETE `/orders/:variety/:order_id` | Cancel pending order | → order_id |
| GET `/orders`, `/orders/:order_id` | Orderbook / order history + status transitions | → order records |
| GET `/trades`, `/orders/:order_id/trades` | Executed trades | → trade records |
| GET `/portfolio/holdings` | Demat holdings (T+1 settled) | → symbol, qty, avg price, P&L |
| GET `/portfolio/positions` | Intraday/derivative positions | → net/day positions |
| GET `/quote`, `/quote/ohlc`, `/quote/ltp` | Market quotes (up to ~500/1000 instruments per call [verify]) | instrument keys → depth/OHLC/LTP |
| GET `/instruments/historical/:token/:interval` | Historical candles | token, interval, from, to → OHLCV |
| GET `/instruments` | Full instrument dump (CSV, refresh daily) | → tradingsymbol ↔ instrument_token map |
| WebSocket `wss://ws.kite.trade` | Streaming ticks (binary) | api_key + access_token → live ticks |
| Postback → your URL | Order status webhook (COMPLETE/CANCEL/REJECTED/UPDATE) | JSON POST + checksum to verify |

## 4. Auth model

- `api_key` (public) + `api_secret` (backend-only).
- Session bootstrap via login redirect → `request_token` → `access_token` with **SHA-256 checksum** (`api_key + request_token + api_secret`).
- All requests: `Authorization: token api_key:access_token`; header `X-Kite-Version: 3`.
- **Daily expiry ~6 AM IST** (regulatory). No refresh token for standard apps — re-login daily. TokenException (HTTP 403) means the session is dead; prompt re-login.
- Postback authenticity via SHA-256 checksum of `order_id + order_timestamp + api_secret`.
- User account must have TOTP 2FA enabled.

## 5. Pricing (INR)

- **Kite Connect app: ₹500/month per app** (reduced from ₹2,000 in 2025). Covers live market data (quotes/WebSocket) **and historical data** — the old ₹2,000/mo historical add-on was folded in free from Feb 2025.
- **Order placement + account/portfolio APIs: free** since ~March 2025 — a ₹0 "personal" tier exists for trading-only APIs [verify whether ₹0 tier applies to third-party multi-user apps or only personal use].
- **Kite Publisher: free.**
- Normal Zerodha brokerage applies to users' trades (₹0 equity delivery; ₹20/0.03% intraday).
- Platform/startup partnerships priced separately by Zerodha's team [verify].

Sources: kite.trade forum announcements ("Revising Kite Connect fees from ₹2000 to ₹500", "Historical data is now free"), support.zerodha.com Kite API charges article.

## 6. India compliance notes (RIA lens)

- **RIA model fits naturally:** advice generated in Yesly, execution in the client's own Zerodha account and demat — no client-fund pooling, satisfying SEBI's segregation expectations for advisers.
- **Automated/unattended orders = algo trading:** SEBI's retail algo framework (2025) requires exchange registration of strategies and imposes limits (e.g., static IP for API access, order rate caps) for orders generated without per-order human action [verify current implementation status]. Zerodha will not host client algos; exchange approval is needed for full automation.
- **Publisher path is exempt** because the user manually confirms each basket on Zerodha's screen — lowest-friction compliant option for an RIA.
- **Consent & DPDP:** the Zerodha linkage token is sensitive personal/financial data — store encrypted in AWS Mumbai, purpose-limit to order execution, honor deletion. Log user consent for each order instruction.
- Zerodha's T&Cs put regulatory responsibility on the platform: document the "user-initiated order" flow for SEBI audit.

## 7. Common pitfalls

- **Daily token death at ~6 AM IST** — every user must re-login each day; batch/overnight jobs cannot use yesterday's token.
- **No sandbox/test environment** — Zerodha provides no paper-trading API; test with a real funded account and tiny quantities (community mocks exist but are unofficial).
- **Rate limits:** quote 1 req/s, historical 3 req/s, orders 10 req/s and 200/min-class caps; hard limits of 10 orders/s, ~400 orders/min [verify], 5,000 orders/day per user per api_key, 25 modifications per order. HTTP 429 on breach.
- **WebSocket:** limited concurrent connections per api_key (commonly 3) and ~3,000 instrument subscriptions per connection [verify]; binary protocol needs the official client libs.
- **Instrument tokens change** — re-download the instruments dump daily before mapping symbols.
- **Postbacks only cover orders placed via your api_key**; user-placed Kite app orders won't hit your webhook — reconcile with GET /orders.
- **Redirect URL is fixed per app** — plan mobile deep-link handoff (backend redirect page → app scheme).

## 8. vs smallcase Gateway (as used in Yesly)

| | Kite Connect | smallcase Gateway |
|---|---|---|
| Broker coverage | Zerodha only | Multi-broker (Zerodha, Groww, etc.) |
| Order model | Backend-placed single orders, full order lifecycle control | Deep-link, user confirms basket in broker flow |
| Reads | Holdings, positions, quotes, historical | Holdings import (with consent), no market data |
| Cost | ₹500/mo per app (data); order APIs free | Commercial per-transaction/licence pricing [verify] |
| Compliance posture | You own order-placement compliance; algo approval if automated | User-confirmed flow, compliance largely absorbed by gateway |
| Fit | Power path for Zerodha users: single-stock buys/sells, live P&L, quotes inside Yesly | Broker-agnostic basket execution & rebalancing |

**Recommendation:** keep smallcase Gateway as the broker-agnostic basket path; use Kite Connect for Zerodha-linked users to enable in-app single-equity orders (explicit user tap per order), live holdings sync, and quotes. Avoid unattended automated ordering until exchange/algo approvals are in place.

---
*Sources: kite.trade/docs/connect/v3 (user, orders, postbacks, exceptions), kite.trade forum pricing announcements, support.zerodha.com Kite API FAQs, zerodha.com/products/api.*
