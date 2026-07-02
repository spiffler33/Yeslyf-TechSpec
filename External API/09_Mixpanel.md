# 09 — Mixpanel (Product Analytics)

> Researched July 2026 against official Mixpanel docs. Facts marked **[verify]** should be re-checked at contract time.

## Overview & role in Yesly

Mixpanel is the event-analytics backbone for Yesly. Nearly every screen fires events (screen viewed, video watched-vs-skipped, question answered, pledge taken), which feed:

- **Funnels** — e.g., Splash → 3 opening questions → OTP login → first module completion, with drop-off at each step.
- **A/B tests on the 3 opening questions** — variant assignment sent as an event property / user property, compared in funnels.
- **Cohorts & retention** — e.g., "users who watched ≥3 videos in week 1" vs churn.
- **Session replay (mobile)** — optional visual debugging of drop-offs.

**Key 2026 update relevant to Yesly:** Mixpanel now offers **India data residency** (GCP `asia-south1`, Mumbai) alongside US and EU. The earlier assumption that India residency is unavailable is **outdated** — it was launched in October 2024. This materially improves the DPDP posture (see Compliance below).

## How it works

```
[RN app (Expo)]
   mixpanel-react-native SDK
   ├─ track("Video Watched", {module_id, pct_watched, variant})
   ├─ identify($user_id = cognito_sub)   ← after OTP/Google login
   └─ people.set({$name, plan_tier})     ← profile props (non-financial only)
        │  batched HTTPS POST
        ▼
[Mixpanel ingestion: api-in.mixpanel.com  (India residency project)]
        │
        ▼
[Identity resolution — Simplified ID Merge: $device_id ↔ $user_id]
        │
        ▼
[Mixpanel project (Mumbai DC)] ──► Funnels / Cohorts / Experiments / Replay UI
        ▲
[FastAPI backend] ── POST /import (service account) for server-side events
                     (e.g., "Pledge Recorded", "Advisor Booking Confirmed")
```

Two ingestion paths:
1. **Client-side** (`/track` via RN SDK) — UI events, automatic batching/retry, offline queue.
2. **Server-side** (`/import` from FastAPI) — trusted events, backfills, events derived from webhooks (e.g., Calendly booking). `/import` is strict-validation, supports gzip, and is the recommended server path.

## Key endpoints / SDK calls

| Method | Path / SDK call | Purpose | Key request fields | Key response |
|---|---|---|---|---|
| POST | `https://api-in.mixpanel.com/track` (US: `api.mixpanel.com`) | Client event ingestion (lenient, drops bad rows silently by default; `?verbose=1` for errors) | JSON array of `{event, properties:{token, distinct_id, time, $insert_id, ...}}` | `1`/`0` (or verbose JSON) |
| POST | `https://api-in.mixpanel.com/import?strict=1` | Server-side / historical ingestion; strict validation; gzip; ≤2,000 events/batch, ≤2MB | Same event JSON; `time` required; `$insert_id` strongly recommended | `{"code":200,"num_records_imported":N}`; 400 with per-row errors |
| POST | `https://api-in.mixpanel.com/engage#profile-set` | User profile upsert (`$set`) | `{$token, $distinct_id, $set:{...}}` | `1`/`0` |
| POST | `https://api-in.mixpanel.com/engage#profile-set-once` | Set-if-absent (`$set_once`) e.g. first_seen, acquisition_source | `{$token, $distinct_id, $set_once:{...}}` | `1`/`0` |
| SDK | `new Mixpanel(token, trackAutomaticEvents)` + `mixpanel.init(optOutTrackingDefault, superProperties, serverURL)` | RN init; **`serverURL: "https://api-in.mixpanel.com"` is mandatory for the India project** | — | — |
| SDK | `mixpanel.identify(userId)` | Attach `$user_id` after login (use Cognito `sub`, never email/phone) | — | — |
| SDK | `mixpanel.track(name, props)`, `mixpanel.registerSuperProperties({...})`, `mixpanel.getPeople().set({...})` | Events, super props (variant, app_version), profiles | — | — |
| SDK | `mixpanel.reset()` | On logout — generates a new `$device_id` | — | — |
| GET | `https://data-in.mixpanel.com/api/2.0/export` | Raw event export (service account) | `from_date`, `to_date` | JSONL events |

### Example event JSON (`/import` body)

```json
[
  {
    "event": "Video Watched",
    "properties": {
      "time": 1751450000,
      "distinct_id": "c3f1a2b4-cognito-sub",
      "$insert_id": "9f0e2d-uuid-v4-per-event",
      "$user_id": "c3f1a2b4-cognito-sub",
      "$device_id": "expo-install-uuid",
      "module_id": "m12_tier2_tax",
      "watched_pct": 82,
      "outcome": "watched",
      "opening_q_variant": "B",
      "app_version": "1.4.0"
    }
  }
]
```

Limits: ≤255 properties/event, names/string values truncated at 255 chars, soft cap of 5,000 distinct event names, ≤2,000 events per batch.

## Auth

- **`/track`**: project **token** embedded in each event (public, not a secret — fine for client SDK).
- **`/import` / export APIs**: **Service Account** (recommended) via HTTP Basic auth (`username:secret`) + `project_id` query param; or legacy project API secret. Store the service-account secret in AWS Secrets Manager, used only by FastAPI.
- No OAuth; no request signing.

## Webhooks

Mixpanel is analytics-in, not webhooks-out for raw events. It offers **cohort sync / webhook destinations** (send cohort membership changes to a URL or partners like Braze) on paid plans **[verify plan gating]**. Yesly likely doesn't need this at MVP; funnels/cohorts are consumed in the Mixpanel UI.

## Sandbox / testing

- No formal sandbox — create a **separate dev project** (also India residency) and a separate token; keep dev/prod tokens in Expo env config.
- `/import?strict=1&verbose=1` returns per-row validation errors — use in CI smoke tests.
- Lexicon lets you define/govern an event schema; do this early (a tracking-plan spreadsheet → Lexicon) given "events on nearly every screen".

## Pricing (July 2026)

- **Free**: up to **1M monthly events**, unlimited seats, 5 saved reports, ~10K session replays/mo. Likely sufficient for Yesly's MVP and well beyond (1M events ≈ 33k DAU × 30 events/day-month equivalent).
- **Growth**: usage-based, **~$0.28 per 1,000 events above 1M/mo** (as of May 2026); unlimited reports, ~20K replays. Ballpark: 5M events ≈ $1.1k/mo, 10M ≈ $2.5k/mo. Annual prepay discounts ~10–15%.
- **Enterprise**: custom (governance, SLAs).
- Session Replay (mobile incl. React Native) is GA on **all plans/regions** with monthly free allotments; higher volumes are a paid add-on.
- **[verify]** whether India residency requires a paid plan — docs say to contact Mixpanel to enable it; confirm it's available on Free/Growth before committing.

## Compliance & data residency (DPDP)

- **India residency exists**: project data stored/processed in GCP `asia-south1` (Mumbai); use `in.mixpanel.com` console and `api-in.mixpanel.com` ingestion. **Residency is fixed at project creation — cannot migrate later.** Decide before the first event.
- Under DPDP Act 2023 + DPDP Rules (notified 13 Nov 2025, full compliance by ~May 2027): cross-border transfer is blacklist-based (permitted unless a country is restricted), so US residency wouldn't be illegal today — but India residency removes the question entirely and is the safer default for a SEBI-registered RIA. Note Mixpanel Inc. remains a US company (support/admin access is cross-border); cover via DPA.
- **PII hygiene (critical)**: never send financial data (portfolio values, PAN, bank info, risk scores as raw numbers), phone, or email as event/profile properties. Use Cognito `sub` as `distinct_id`. Bucket sensitive dimensions (e.g., `aum_band: "5-10L"`) only if product truly needs them. Configure RN session replay masking (text/image masking is on by default — keep it on for all input fields).
- Sign Mixpanel's DPA; document Mixpanel as a processor in the DPDP notice/consent flow.

## Gotchas & effort

1. **Residency is irreversible** — creating the project on US servers by accident means re-creating and losing history.
2. **`serverURL` everywhere** — RN SDK, server `/import`, and any proxy must all point at `api-in.mixpanel.com`; events sent to the US endpoint for an India project are silently lost.
3. **`$insert_id` dedup** — dedup is best-effort within (project, event name, distinct_id, ~same day). Generate a UUID per logical event at the producer; on FastAPI retries reuse the same `$insert_id` or you'll double-count funnel steps.
4. **Simplified ID Merge pitfalls**: org must be on Simplified ID Merge (default for new orgs). Merges ($device_id↔$user_id) are permanent — never reuse a device_id for another user; call `reset()` on logout (shared-device scenario) or post-logout events pollute the previous user. Fire `identify()` immediately after Cognito login, before other events, or pre-login events land in a separate anonymous profile until merged.
5. **Ad-blockers / tracking blockers** — minor on native mobile, but if any web surface exists, route through a first-party proxy (CloudFront → `api-in.mixpanel.com`) ; the RN SDK's `serverURL` supports pointing at your proxy.
6. **A/B on opening questions**: Mixpanel is the measurement layer, not the assignment layer — assign variants in your backend or a flag tool and stamp `opening_q_variant` as a super property + user property. Mixpanel's own "Experiments" report reads such properties.
7. **Event-name sprawl** — with events on every screen, enforce a naming convention (Object-Action, e.g., `Video Watched`) via Lexicon before launch; renaming later breaks saved funnels.

**Effort ballpark**: RN SDK + identity plumbing + tracking plan ≈ 3–5 dev-days; server `/import` pipeline ≈ 1–2 days; dashboards/funnels ≈ ongoing analyst time.

## Sources

- https://docs.mixpanel.com/docs/privacy/in-residency (India residency, endpoints)
- https://mixpanel.com/blog/india-data-residency/ and https://docs.mixpanel.com/changelogs/2024-10-11-india-data-residency
- https://docs.mixpanel.com/docs/privacy/eu-residency
- https://docs.mixpanel.com/docs/data-structure/events-and-properties (event JSON, limits)
- https://developer.mixpanel.com/reference/import-events / https://developer.mixpanel.com/reference/track-event **[endpoint reference]**
- https://docs.mixpanel.com/docs/tracking-methods/sdks/react-native and .../react-native/react-native-replay (RN SDK + session replay)
- https://github.com/mixpanel/mixpanel-react-native ; https://github.com/mixpanel/mixpanel-react-native-session-replay
- https://mixpanel.com/pricing/ (Free 1M events; Growth ~$0.28/1k over 1M — May 2026)
- https://docs.mixpanel.com/docs/session-replay (plan availability)
- DPDP Rules status: https://www.dpdpa.com/dpdparules/rule15.html ; https://itif.org/publications/2025/06/09/india-cross-border-data-transfer-regulation/
