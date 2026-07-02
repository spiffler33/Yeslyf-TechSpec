# 10 — Sentry (Error Monitoring & Performance)

> Researched July 2026 against official Sentry docs. Facts marked **[verify]** should be re-checked before contract.

## Overview & role in Yesly

Sentry gives Yesly one pane of glass for:

- **React Native / Expo app**: JS errors, **native crashes** (iOS/Android), ANRs, release health (crash-free sessions per release), performance (app start, slow screens, HTTP spans).
- **FastAPI backend**: unhandled exceptions, slow endpoints, DB spans.
- **Distributed tracing**: one trace from a tap in the RN app through the FastAPI call it triggered — invaluable when debugging "OTP login hangs at Step 3"-type issues.

For a finance app, the differentiator is disciplined **data scrubbing** — Sentry must never store request bodies containing financial or identity data.

## How it works

```
[RN app (Expo)]                          [FastAPI (AWS Mumbai)]
 @sentry/react-native                     sentry-sdk (python)
 ├─ JS error handler                      ├─ Starlette/FastAPI middleware
 ├─ Native crash handlers (iOS/Android)   ├─ traces_sample_rate sampling
 ├─ Breadcrumbs (nav, taps, http)         └─ before_send scrubbing
 ├─ Session tracking → release health
 └─ Tracing: outgoing fetch gets
    sentry-trace + baggage headers ────────► continues same trace
        │                                        │
        ▼                                        ▼
        [Sentry ingest: o<org>.ingest.de.sentry.io  (EU region org)]
                              │
             server-side scrubbing (PII filters)
                              │
        [Issues (fingerprinted) · Transactions/Spans · Releases · Alerts]
                              │
              Slack / email / PagerDuty / Jira
```

**Event model**: an *event* (error or transaction) is grouped into an **issue** by **fingerprinting** (stack-trace based, customizable). Events carry **breadcrumbs** (trail of prior actions), **release** (ties to a deploy; enables regressions + suspect commits), **environment** (dev/staging/prod), tags, and contexts. **Transactions** contain **spans** (http, db, UI render); traces link transactions across services via `trace_id`.

## Key SDK calls / endpoints

| Method | Call / path | Purpose | Key fields (request) | Key fields (response/result) |
|---|---|---|---|---|
| SDK (RN) | `Sentry.init({ dsn, environment, release, dist, tracesSampleRate, sendDefaultPii:false, enableAutoSessionTracking:true })` | Init; wraps root component via `Sentry.wrap(App)` | DSN routes to region ingest | — |
| SDK (RN) | `Sentry.captureException(e)` / `captureMessage()` | Manual capture | error, extras, tags | event_id |
| SDK (RN) | `Sentry.addBreadcrumb({category, message, level})` | Custom breadcrumbs (e.g., "otp_requested") | — | — |
| SDK (RN) | `Sentry.setUser({ id: cognitoSub })` | Correlate errors to user — **id only, no email** | — | — |
| Config | `app.json → plugins: ["@sentry/react-native/expo", {project, organization, url}]` + `metro.config.js → getSentryExpoConfig` | Expo config plugin: native init + source-map/debug-ID wiring for EAS builds | `SENTRY_AUTH_TOKEN` as EAS secret | — |
| CLI | `npx sentry-expo-upload-sourcemaps dist` | Upload source maps after **`eas update`** (OTA) — not automatic | release, dist must match | — |
| SDK (Py) | `sentry_sdk.init(dsn, traces_sample_rate=0.2, profiles_sample_rate, environment, release, send_default_pii=False, event_scrubber=EventScrubber(denylist=[...]))` | FastAPI/Starlette integrations auto-enable on import | — | — |
| SDK (Py) | `sentry_sdk.set_user({"id": sub})`, `sentry_sdk.start_span(...)` | Context + custom spans | — | — |
| HTTP | `POST https://o<org>.ingest.de.sentry.io/api/<project>/envelope/` | Raw envelope ingestion (SDKs handle this) | envelope: event JSON | 200 + event id |
| API | `https://de.sentry.io/api/0/...` (org-region API) | Releases, issues automation (e.g., `POST /organizations/{org}/releases/`) | version, refs, projects | release object |

### Example error event (simplified JSON as stored)

```json
{
  "event_id": "9abf12...",
  "level": "error",
  "release": "com.yesly.app@1.4.0+42",
  "environment": "production",
  "user": { "id": "c3f1a2b4-cognito-sub" },
  "exception": { "values": [{ "type": "TypeError",
      "value": "undefined is not a function",
      "stacktrace": { "frames": ["..."] } }] },
  "breadcrumbs": [
    { "category": "navigation", "message": "OpeningQuestions -> OtpLogin" },
    { "category": "http", "data": { "url": "/auth/otp", "status_code": 500 } }
  ],
  "contexts": { "trace": { "trace_id": "4bf92f35...", "span_id": "00f067aa..." } },
  "tags": { "os": "android 15", "device": "Pixel 8" }
}
```

## Auth

- **DSN** per project (public key, safe to embed in the app) — routes events to your org's region ingest domain.
- **Auth tokens** (org tokens with scopes, e.g., `project:releases`) for CI: source-map upload, release creation. Store as EAS secret (`SENTRY_AUTH_TOKEN`) and in GitHub Actions secrets for backend releases.
- SaaS SSO for team access; API uses Bearer tokens.

## Webhooks / alerting & integrations

- **Issue alerts** (new issue, regression, spike) and **metric alerts** (error rate, p95 latency, crash-free % drop) route to **Slack, email, PagerDuty/Opsgenie, MS Teams**; two-way **Jira/Linear/GitHub** issue linking.
- Generic **webhooks + Sentry Integration Platform** for custom receivers (e.g., a FastAPI endpoint that posts to an internal ops channel). Payload: issue/event JSON, signed with a shared secret (`sentry-hook-signature`).
- Release health alerts: crash-free sessions below threshold after a release → good gate for staged rollouts of OTA updates.

## Sandbox / testing

- No sandbox needed: use separate **projects** (`yesly-app`, `yesly-api`) and **environments** (dev/staging/prod); filter alerts to prod only.
- Test wiring with `Sentry.nativeCrash()` (RN) and a `/debug-sentry` route raising an exception in FastAPI.
- Verify source maps: force a JS error in a release build and check the stack is symbolicated; `npx sentry-cli sourcemaps explain <event_id>` diagnoses failures.

## Pricing (July 2026)

- **Developer (free)**: 1 user, ~5K errors/mo, limited spans/replays, 30-day retention. OK for a solo spike, not a team.
- **Team**: **$26/mo base** — includes **50K errors, 5M spans** (cut from 10M in Aug 2025), 50 replays, 1 cron monitor, 5GB logs, 1GB attachments; unlimited users. Overage via pay-as-you-go budget (errors ≈ $0.00029/ea at Team rates; spans ≈ $8 per extra ~5M-ish block — usage-dependent). Reserved volume ~20% cheaper than PAYG.
- **Business**: **$80/mo base**, same base quotas, adds 90-day retention, advanced quota controls, dashboards, metric alerts breadth.
- Ballpark for Yesly at moderate scale (100K errors, 10M spans, 500 replays): **≈ $50/mo on Team**.
- **Self-hosted**: free (FSL-licensed, Docker Compose, ~16GB RAM min) — you pay infra + ops; no SLA. Realistic only if data residency becomes a hard requirement.

## Compliance & data residency (DPDP)

- SaaS regions: **US or EU (Germany, `de.sentry.io`) only — no India region**. Region is chosen at org creation and **cannot be changed** (new org required). Choose **EU** for the stronger adequacy posture.
- DPDP (Rules notified Nov 2025; blacklist-based cross-border regime, full effect ~May 2027): exporting *error telemetry* to EU is currently permissible; risk is PII leaking into events. Mitigate:
  - `send_default_pii=False` (default) in both SDKs; never attach request bodies on auth/financial endpoints (`max_request_body_size="never"` on sensitive routes or globally).
  - Python `EventScrubber` denylist + Sentry **server-side data scrubbing** (Settings → Security & Privacy: scrub defaults on, add custom fields: `pan`, `phone`, `otp`, `account`, `amount`).
  - `setUser({id})` only; no email/phone/name.
  - `before_send` / `beforeSend` hooks as the last line of defense (drop or redact).
- Self-hosted Sentry in AWS Mumbai is the fallback if regulators/clients demand full localization — budget ~1 engineer-week setup + ongoing ops.
- Sign Sentry's DPA; list as processor in privacy notice.

## Gotchas & effort

1. **`sentry-expo` is dead** — deprecated since Expo SDK 50. Use `@sentry/react-native` + its Expo config plugin + `getSentryExpoConfig` in Metro. Old tutorials referencing `sentry-expo` will mislead.
2. **Source maps & EAS**: native EAS **builds** upload source maps automatically (with the auth token secret), but **`eas update` (OTA) does not** — you must run `sentry-expo-upload-sourcemaps` after every update or JS stacks are minified garbage. Keep `release`/`dist` consistent between the update and the upload; this is the #1 support-ticket generator.
3. **Quota burn from noisy errors**: one buggy render loop can eat 50K errors in an hour. Set **spike protection** (on by default), per-key rate limits, `ignoreErrors` for known noise (network blips, `AbortError`), and sample transactions (`tracesSampleRate` 0.1–0.2 in prod, 1.0 in dev).
4. **Fingerprint drift**: OTA updates without source maps change stack shapes → duplicate issues; another reason to nail gotcha #2.
5. **Distributed tracing needs CORS/header allow-listing**: FastAPI must accept `sentry-trace` and `baggage` headers; propagate `tracePropagationTargets` in RN to only your API domain (avoid leaking trace headers to third parties).
6. **Session replay on RN** exists but records screens — treat like Mixpanel replay: masking on, and consider disabling on financial-detail screens entirely.

**Effort ballpark**: RN + Expo plugin + source-map CI ≈ 2–3 dev-days; FastAPI ≈ 0.5 day; scrubbing rules + alert routing ≈ 1 day; ongoing triage discipline.

## Sources

- https://docs.sentry.io/platforms/react-native/manual-setup/expo/ (Expo plugin; sentry-expo deprecation)
- https://docs.expo.dev/guides/using-sentry/
- https://docs.sentry.io/platforms/react-native/sourcemaps/uploading/expo-advanced/ (EAS update manual upload)
- https://docs.sentry.io/platforms/python/integrations/fastapi/ **[FastAPI integration]**
- https://docs.sentry.io/organization/data-storage-location/ (US/EU only; immutable)
- https://docs.sentry.io/pricing/ (quotas: 50K errors / 5M spans on Team; PAYG model)
- https://sentry.io/pricing/ (Team $26, Business $80, Developer free)
- https://sentry.zendesk.com/hc/en-us/articles/40116900282011 (Aug 2025 plan changes — spans 10M→5M)
- https://docs.sentry.io/security-legal-pii/scrubbing/ **[data scrubbing]**
- https://develop.sentry.dev/self-hosted/ **[self-hosted]**
