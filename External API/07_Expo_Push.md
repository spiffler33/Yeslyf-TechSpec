# 07 — Expo Push Notification Service

## Overview & role in Yesly

Expo's push notification service is a free proxy layer in front of FCM (Android) and APNs (iOS). The Yesly app (React Native/Expo) obtains a single cross-platform `ExpoPushToken`; the backend sends one uniform JSON payload to `exp.host` and Expo handles the FCM/APNs protocol differences, credential exchange, and delivery handoff.

Role in Yesly:

- **App (Expo/React Native):** requests OS notification permission, creates Android notification channels, calls `Notifications.getExpoPushTokenAsync({ projectId })`, and POSTs the token to the FastAPI backend (`device_tokens` table keyed to user + device).
- **Container 5 (async notification dispatcher, Celery):** the "push" channel of the fan-out. Celery tasks batch queued notifications (up to 100/request), POST to the Expo send API, persist ticket IDs, then a follow-up task polls the receipts API (~15 min later) and prunes dead tokens.
- Suits Yesly's notification classes: nudge reminders, goal/plan updates, market/portfolio alerts (transactional), and — only with recorded consent — promotional content.

Token model (verified from Expo FAQ):

- The `ExpoPushToken` **never expires**, but it **changes if the Android `applicationId` or the project (`experienceId`/projectId) changes**.
- **Android:** token *may change* when the app is uninstalled and reinstalled.
- **iOS:** token stays the same even after uninstall/reinstall.
- Uninstalled devices surface as `DeviceNotRegistered` errors → backend must stop sending and delete the token.
- Under the hood, Expo maps the ExpoPushToken to the current native FCM registration token / APNs device token; the app never has to handle native tokens (unless you later go direct via `getDevicePushTokenAsync`).

Best practice: re-run `getExpoPushTokenAsync` on every app launch and upsert to the backend (idempotent), so rotation on Android reinstall is self-healing.

## How it works

```
┌──────────────────────────┐
│ Yesly app (Expo RN)      │
│ 1. requestPermissions()  │
│ 2. setNotificationChannel│  (Android: channel must exist client-side)
│ 3. getExpoPushTokenAsync │
│    ({ projectId })       │
└───────────┬──────────────┘
            │ POST /api/v1/devices { expo_push_token, platform, user_id }
            ▼
┌──────────────────────────┐
│ FastAPI backend          │
│ upsert device_tokens     │
│ (token, user, platform,  │
│  last_seen, active flag) │
└───────────┬──────────────┘
            │ enqueue notification job
            ▼
┌──────────────────────────┐        ┌─────────────────────────────┐
│ Container 5: Celery      │ batch  │ Expo push service           │
│ dispatcher               ├───────►│ POST exp.host/--/api/v2/    │
│ - chunk ≤100 msgs/req    │ ≤100   │       push/send             │
│ - gzip body              │        │  → maps ExpoPushToken to    │
│ - retry 429/5xx w/       │        │    FCM v1 / APNs token,     │
│   exponential backoff    │        │    hands off to Google/Apple│
└───────────┬──────────────┘        └──────────────┬──────────────┘
            │ store ticket IDs                     │
            ▼                                      ▼
┌──────────────────────────┐            FCM ──► Android device
│ Celery beat: ~15 min     │            APNs ─► iOS device
│ later, POST /push/       │
│ getReceipts (≤1000 ids)  │
│ - ok        → done       │
│ - DeviceNotRegistered    │
│   → deactivate token     │
│ - MessageTooBig etc.     │
│   → log + fix payload    │
└──────────────────────────┘
```

Key point: a **ticket** only means Expo accepted the message; a **receipt** confirms FCM/APNs accepted the handoff. Neither guarantees the notification was displayed on the device (at-least-once delivery to FCM/APNs; duplicates or drops are possible).

## Key endpoints

Base host: `https://exp.host`. Required headers: `host: exp.host`, `accept: application/json`, `accept-encoding: gzip, deflate`, `content-type: application/json`.

| Method | Path | Purpose | Key request fields | Key response fields |
|---|---|---|---|---|
| POST | `/--/api/v2/push/send` | Send 1–100 notifications (JSON object or array; gzip body supported) | `to` (ExpoPushToken or array), `title`, `body`, `data` (custom JSON, ~4 KiB; total payload ≤ 4096 bytes), `sound` (iOS), `badge` (iOS), `subtitle` (iOS), `mutableContent` (iOS), `channelId` (Android), `categoryId`, `priority` (`default`\|`normal`\|`high`), `ttl` (seconds), `expiration`, `collapseId`, `interruptionLevel` (iOS), `icon`/`tag` (Android), `richContent` | `data`: array of tickets — `status` (`ok`\|`error`), `id` (ticket ID, only when ok), `message`, `details.error`; top-level `errors[]` |
| POST | `/--/api/v2/push/getReceipts` | Fetch delivery receipts for up to 1000 ticket IDs (poll ~15 min after send; receipts kept ~24 h) | `ids`: array of ticket IDs | `data`: map of ticketId → `{ status, message?, details?.error }`; missing IDs = not ready yet; top-level `errors[]` |

### Example — send request

```json
POST https://exp.host/--/api/v2/push/send
Content-Type: application/json

[
  {
    "to": "ExponentPushToken[xxxxxxxxxxxxxxxxxxxxxx]",
    "title": "SIP due tomorrow",
    "body": "Your ₹5,000 SIP for Goal 'House' executes on 3 Jul.",
    "data": { "type": "nudge", "goal_id": "g_123", "deep_link": "yesly://goals/g_123" },
    "sound": "default",
    "badge": 1,
    "channelId": "reminders",
    "priority": "high",
    "ttl": 3600
  }
]
```

### Example — send response (tickets)

```json
{
  "data": [
    { "status": "ok", "id": "XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX" },
    {
      "status": "error",
      "message": "\"ExponentPushToken[yyy]\" is not a registered push notification recipient",
      "details": { "error": "DeviceNotRegistered" }
    }
  ]
}
```

### Example — receipts request/response

```json
POST https://exp.host/--/api/v2/push/getReceipts

{ "ids": ["XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX"] }
```

```json
{
  "data": {
    "XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX": { "status": "ok" },
    "YYYYYYYY-YYYY-YYYY-YYYY-YYYYYYYYYYYY": {
      "status": "error",
      "message": "The device cannot receive push notifications anymore...",
      "details": { "error": "DeviceNotRegistered" }
    }
  }
}
```

### Error codes to handle in Container 5

| Error | Where | Action |
|---|---|---|
| `DeviceNotRegistered` | ticket or receipt | Stop sending; mark token inactive/delete. Continuing to send can get the app throttled by Expo. |
| `MessageTooBig` | receipt | Total payload exceeded 4096 bytes — trim `data`, send a reference ID + fetch on open instead. |
| `MessageRateExceeded` | receipt | Too many messages to that device — back off exponentially, retry later. |
| `MismatchSenderId` / `InvalidCredentials` | receipt | FCM/APNs credential problem in EAS — ops alert, nothing to retry until fixed. |
| `TOO_MANY_REQUESTS` (HTTP 429) | request-level | Exceeded ~600 notifications/sec/project — throttle + exponential backoff. |
| `PUSH_TOO_MANY_NOTIFICATIONS` / `PUSH_TOO_MANY_RECEIPTS` / `PUSH_TOO_MANY_EXPERIENCE_IDS` | request-level | >100 messages, >1000 receipt IDs, or mixed projects per request — fix batching logic. |

### Rate limits

- **600 notifications/second per project** (official). No SLA is offered; Expo advises building in retries (exponential backoff on 429/5xx) and limiting concurrency (the official Node SDK caps at 6 concurrent connections and auto-gzips). Smooth large campaign bursts across seconds rather than firing all at once.

## Auth

- **Default:** no authentication required to call the send API — anyone with a valid ExpoPushToken can push to it. Tokens are therefore secrets; never log or expose them.
- **Enhanced push security (recommended for Yesly):** enable "Enhanced Security for Push Notifications" in the project's expo.dev dashboard, generate an access token, then all send/receipt calls must include `Authorization: Bearer <EXPO_ACCESS_TOKEN>`. Requests without it get `UNAUTHORIZED`. Store the token in AWS Secrets Manager; Node SDK supports it via the `accessToken` constructor option (v3.6.0+), Python SDK via session headers.

## Webhooks / events

**None.** The Expo push service has no delivery webhooks, event streams, or callbacks. The model is **poll-based**: send → store ticket IDs → poll `getReceipts` (~15 minutes later; receipts retained ~24 hours, so the Celery reconciliation task must run within that window). If Yesly needs real-time delivery/open analytics, that's a reason to graduate to OneSignal or direct FCM/APNs (see Gotchas).

## Sandbox / testing

- **Expo push tool:** [expo.dev/notifications](https://expo.dev/notifications) — paste an ExpoPushToken, compose title/body/data/channelId, and send test pushes without any backend. First stop for debugging.
- **Expo Go vs development builds:** since **SDK 53, remote push notifications do not work in Expo Go on Android** (deprecated in SDK 52, removed in 53). Expo Go on iOS still supports them, but the reliable path is a **development build** (`eas build --profile development`) with real FCM/APNs credentials — this is what Yesly QA should use. Local (in-app scheduled) notifications still work in Expo Go.
- **cURL smoke test** against `exp.host/--/api/v2/push/send` works from any machine — useful for testing Container 5 payloads before wiring Celery.
- There is no separate sandbox environment or sandbox tokens — you test against the production Expo service with dev-build tokens.

### Credential setup (EAS) — prerequisite for any real testing

- **Android (FCM v1):** create a Firebase project → Project settings → Service accounts → *Generate new private key* (JSON). Upload via `eas credentials` → Android → production → *Google Service Account* → *Manage your Google Service Account Key for Push Notifications (FCM V1)* → upload key; or via expo.dev → Project settings → Credentials → *FCM V1 service account key*. Also place `google-services.json` in the project and reference it via `expo.android.googleServicesFile` in app.json (this file is safe to commit; the service-account JSON is not — gitignore it).
- **iOS (APNs):** requires a paid Apple Developer account. EAS can generate and manage the APNs key for you during `eas credentials` / `eas build` — the usual choice is to let EAS handle it so the key never touches the repo.

## Pricing

- **Free.** Official FAQ: "There is no cost associated with sending notifications through Expo push notification service." No per-notification or MAU charges; INR conversion n/a.
- Context: EAS **Build/Update/Submit** have free and paid plans (Yesly will likely pay for EAS Build for CI anyway), but the push service itself is not metered — verify current EAS plan terms before build in case bundling changes.
- Hidden cost is operational: Celery workers, receipt-polling jobs, token hygiene tables — engineering time, not vendor fees.

## Compliance notes (India)

- **DPDP Act 2023:** push tokens tied to a user are personal data. Yesly needs a lawful basis (consent or legitimate use) to process them; record consent state per user per notification category (transactional vs promotional) in the backend, and honor withdrawal by deactivating tokens/categories.
- **OS-level permission ≠ legal consent.** The iOS/Android notification permission prompt only authorizes the OS to display notifications — it is not DPDP-grade consent for *marketing* messages. Yesly should collect an explicit in-app opt-in for promotional pushes, separate from the OS prompt, with timestamped audit trail.
- **SEBI RIA context:** advisory nudges/alerts should be distinguishable from promotional content; keep the classification in the notification payload/records so communications audits are answerable. (Exact SEBI communication-record requirements — verify with compliance before build.)
- **Data residency:** Expo's push infrastructure is not India-hosted; notification titles/bodies transit Expo, Google (FCM), and Apple (APNs) servers abroad. Keep payloads minimal — send "You have a new plan update" + deep link, not portfolio values or PII, so sensitive data stays in AWS Mumbai and is fetched over the authenticated API on open. TRAI/DLT rules apply to SMS/voice, not push — no DLT registration needed for this channel.

## Gotchas & effort

**Gotchas**

1. **Tickets ≠ delivery.** A `status: "ok"` ticket only means Expo queued it. You must persist ticket IDs and poll `getReceipts` (≤1000 IDs/call, within 24 h) or you will silently accumulate dead tokens and blame "flaky push".
2. **Prune `DeviceNotRegistered` immediately** (from tickets *and* receipts). Repeatedly pushing to dead tokens risks throttling by Expo/FCM/APNs.
3. **Token rotation:** Android tokens can change on reinstall; app must re-register the token on every launch and the backend upsert must deduplicate (same device, new token → replace, don't append).
4. **iOS permission UX:** the permission prompt can only be shown once; if the user declines, you can't re-prompt (only deep-link to Settings). Use a pre-permission explainer screen. iOS **provisional authorization** (`ios: { allowProvisional: true }` in `requestPermissionsAsync`) delivers quietly to Notification Center without a prompt — good for trialing nudges; verify current expo-notifications API surface before build.
5. **Android channels are client-side:** `channelId` in the payload must match a channel already created in the app via `setNotificationChannelAsync`. Channel **importance is fixed at creation** — you can't raise it later from the server (only the user can, in Settings). Create Yesly's channels (`reminders`, `alerts`, `promotions`) on first launch before token registration.
6. **Indian OEM battery optimizers:** Xiaomi/MIUI, Oppo/ColorOS, Vivo, OnePlus aggressively kill background processes and can suppress FCM delivery — a large share of Yesly's Android user base. Receipts will say "ok" (FCM accepted) yet nothing shows. Mitigations: `priority: "high"` for time-critical nudges, in-app prompts to whitelist Yesly from battery optimization, and never relying on push as the sole channel for critical actions (pair with in-app inbox). This is an ecosystem problem, not fixable in code — set expectations.
7. **Credential mismatches** (`MismatchSenderId`, `InvalidCredentials`) after Firebase project changes or Apple key revocation are the most common "push suddenly stopped" cause — monitor receipt errors and alert.
8. **No server-side scheduling, no segmentation, no analytics dashboard.** Expo sends immediately, to explicit token lists, with no open/CTR tracking. Yesly's Celery beat handles scheduling and the backend owns segmentation. Graduate to **OneSignal** (segments, journeys, analytics, quiet hours) or **direct FCM v1/APNs** (via `getDevicePushTokenAsync`) if/when marketing needs outgrow this — the migration cost is a token-type change plus sender rewrite.
9. **Server SDKs:** Node — `expo-server-sdk` (official, maintained by Expo: chunking to 100, receipt chunking to 1000, gzip, 6-connection throttle, `accessToken`). Python — `exponent-server-sdk` (PyPI) from `expo-community/expo-server-sdk-python`: **community-maintained**, v2.x, irregular release cadence driven by contributions; it works (PushClient.publish, PushMessage, typed exceptions incl. DeviceNotRegisteredError) but pin the version, review the source (~1 file), and be prepared to vendor it or call the HTTP API directly with `httpx` from Celery — verify maintenance status before build.

**Estimated integration effort**

- App side (permissions UX, channels, token registration, notification handlers/deep links): **2–3 dev-days**.
- Backend/Container 5 (token store + upsert API, Celery send task with chunking/backoff, ticket persistence, receipts reconciliation job, token pruning, consent gating): **3–5 dev-days**.
- EAS credentials (FCM v1 + APNs) and dev-build test loop: **0.5–1 day**.
- Total: roughly **1–1.5 sprint-weeks** for one engineer, production-grade with receipt reconciliation.

## Sources

- https://docs.expo.dev/push-notifications/sending-notifications/ — send API, payload fields, limits, tickets/receipts, errors, security
- https://docs.expo.dev/push-notifications/push-notifications-setup/ — token retrieval, permissions, channels, credentials
- https://docs.expo.dev/push-notifications/faq/ — token lifecycle, free pricing, 600/sec limit, DeviceNotRegistered, at-least-once delivery
- https://docs.expo.dev/push-notifications/fcm-credentials/ — FCM v1 service-account setup via EAS
- https://docs.expo.dev/push-notifications/overview/ — service overview
- https://docs.expo.dev/versions/latest/sdk/notifications/ — expo-notifications API reference
- https://expo.dev/changelog/sdk-53 — Expo Go Android push removal (SDK 53)
- https://expo.dev/notifications — push testing tool
- https://github.com/expo/expo-server-sdk-node — official Node sender SDK
- https://github.com/expo-community/expo-server-sdk-python / https://pypi.org/project/exponent-server-sdk/ — community Python SDK
