# 13 — Scheduling: Calendly vs Cal.com (Decision 9 — Step 31 Tier-4 advisor booking)

> Researched July 2026. **Landscape change:** Cal.com took its production codebase **closed-source in April 2026**; the open-source community edition was relaunched as **Cal.diy** (MIT). Facts marked **[verify]**.

## Overview & role in Yesly

Step 31: a Tier-4 user books a session with a SEBI-registered advisor. Requirements implied by the flow:

- Multiple advisors → availability, **round-robin or routed assignment**.
- Booking initiated **inside the RN app**, confirmation reflected in-app (and ideally in USER/journey state via webhook → FastAPI).
- Finance context → prefer **data-residency control** and minimal PII leaving India.
- Low volume initially (Tier-4 is deep in the funnel) → cost sensitivity, option to start manual.

Contenders: **Calendly** (polished SaaS, webview embed), **Cal.com** (API-first platform, self-hostable via Cal.diy), **manual** (ops-managed Google Calendar + a form).

## How it works

```
Option A — Calendly
[RN app] → GET yesly-api/advisors/slots?  (optional: proxy Calendly availability)
[RN app] → WebView(https://calendly.com/d/<single-use-link>?prefill…)
   user books in webview (Calendly UI, Calendly servers, US data)
Calendly ──webhook invitee.created──► [FastAPI] → write BOOKING row,
   mark journey step done, push in-app confirmation
   (cancel/reschedule → invitee.canceled + new invitee.created)

Option B — Cal.com (managed platform or self-hosted Cal.diy)
[RN app] → GET api.cal.com/v2/slots?eventTypeId=…&start=…&end=…   (native UI!)
[RN app] → POST api.cal.com/v2/bookings {start, attendee:{name,email,tz}}
   ← booking JSON (uid, meet link) — rendered natively in-app
Cal.com ──webhook BOOKING_CREATED / RESCHEDULED / CANCELLED──► [FastAPI]
   Round-robin: team event type distributes across advisor calendars

Option C — Manual
[RN app] → "Request a call" form → FastAPI → ops Slack/email
   ops books on advisor's Google Calendar; status pushed back manually
```

## Key endpoints

### Calendly API v2 (v1 was discontinued May 2025)

Base: `https://api.calendly.com` — Bearer auth.

| Method | Path | Purpose | Key request fields | Key response fields |
|---|---|---|---|---|
| GET | `/users/me` | Resolve current user/org URIs | — | `resource.uri`, `current_organization` |
| GET | `/event_types?organization={uri}` | List bookable event types (per advisor) | — | `collection[]{uri, scheduling_url, duration}` |
| POST | `/scheduling_links` | **Single-use** booking link (expires after 1 booking / 90 days) | `owner` (event_type uri), `max_event_count:1` | `booking_url` |
| GET | `/scheduled_events?organization=&invitee_email=` | List bookings | filters | `collection[]{uri, start_time, status}` |
| GET | `/scheduled_events/{uuid}/invitees` | Invitee details (email, Q&A, cancel/reschedule URLs) | — | `cancel_url`, `reschedule_url`, `questions_and_answers` |
| POST | `/webhook_subscriptions` | Register webhook | `url`, `events:["invitee.created","invitee.canceled","invitee_no_show.created"]`, `organization`, `scope`, `signing_key` | subscription uri |

Notes: **no public "create booking" API** — booking happens in Calendly's UI (webview). Availability API exists (`/event_type_available_times`) for display. A newer "Scheduling API" for building custom booking UIs was announced on the developer portal **[verify scope/eligibility — likely Enterprise]**. Rate limits ~60 req/min (120 Enterprise). Round-robin/routing requires the **Teams** plan.

### Cal.com API v2

Base: `https://api.cal.com/v2` — header `cal-api-version` pins behavior.

| Method | Path | Purpose | Key request fields | Key response fields |
|---|---|---|---|---|
| GET | `/event-types` / `/teams/{id}/event-types` | Event types incl. team round-robin types | — | `id, slug, lengthInMinutes, schedulingType` |
| GET | `/slots?eventTypeId=&start=&end=&timeZone=Asia/Kolkata` | **Available slots — powers a fully native picker** | query params | `slots{date:[times]}` |
| POST | `/bookings` | **Create booking via API** | `eventTypeId`, `start` (UTC ISO), `attendee:{name, email, timeZone, language}`, `metadata` | `uid, status, start/end, meetingUrl, attendees` |
| POST | `/bookings/{uid}/cancel` · `/bookings/{uid}/reschedule` | Cancel / reschedule programmatically (in-app flows) | `reason` / `start` | updated booking |
| GET | `/bookings?attendeeEmail=&status=` | List/reconcile bookings | filters | bookings[] |
| POST | `/webhooks` (or per event-type/team) | Register webhook | `subscriberUrl`, `triggers:["BOOKING_CREATED","BOOKING_RESCHEDULED","BOOKING_CANCELLED","BOOKING_NO_SHOW_UPDATED","MEETING_ENDED"]`, `secret` | webhook id |

### Example — Cal.com create booking

```json
POST /v2/bookings   (cal-api-version: 2024-08-13)
{
  "eventTypeId": 812345,
  "start": "2026-07-15T09:30:00Z",
  "attendee": { "name": "Asha P", "email": "asha@x.com",
                "timeZone": "Asia/Kolkata", "language": "en" },
  "metadata": { "yesly_user_id": "c3f1a2b4", "tier": "4" }
}
→ 201
{ "status": "success", "data": { "uid": "bk_9XyZ", "status": "accepted",
    "start": "2026-07-15T09:30:00Z", "end": "2026-07-15T10:00:00Z",
    "hosts": [{ "name": "Advisor R" }], "meetingUrl": "https://meet.google.com/…" } }
```

### Example — Calendly `invitee.created` webhook payload (trimmed)

```json
{ "event": "invitee.created",
  "payload": {
    "email": "asha@x.com", "name": "Asha P",
    "scheduled_event": { "uri": "https://api.calendly.com/scheduled_events/AAA",
      "start_time": "2026-07-15T09:30:00Z",
      "event_memberships": [{ "user_email": "advisor@yesly.in" }] },
    "cancel_url": "https://calendly.com/cancellations/…",
    "reschedule_url": "https://calendly.com/reschedulings/…",
    "tracking": { "utm_content": "yesly_user_c3f1a2b4" } } }
```

## Auth

- **Calendly**: Personal Access Token (per admin user; doesn't expire; fine for a single-org backend) or **OAuth 2.1** (only needed for multi-tenant public apps). Webhook payloads signed (`Calendly-Webhook-Signature`, HMAC with your `signing_key`) — verify in FastAPI.
- **Cal.com managed**: API keys (`Bearer cal_live_…`) for org-level use; **Platform plans** use OAuth client credentials (`x-cal-client-id`/`x-cal-secret-key`) + "managed users" (create a shadow Cal user per advisor — Yesly likely only needs advisors as team members, not managed end-users). Webhooks HMAC-signed with your secret (`X-Cal-Signature-256`).
- **Cal.diy self-hosted**: your instance, your API keys; same v2-style API surface **[verify Cal.diy API parity post-fork — commercial features were stripped]**.

## Webhooks

| | Calendly | Cal.com |
|---|---|---|
| Booking created | `invitee.created` | `BOOKING_CREATED` |
| Canceled | `invitee.canceled` (reschedule = cancel + new create, linked via `old_invitee`/`new_invitee`) | `BOOKING_CANCELLED`, distinct `BOOKING_RESCHEDULED` |
| No-show | `invitee_no_show.created` / `.deleted` (manual marking) | `BOOKING_NO_SHOW_UPDATED` |
| Other | `routing_form_submission.created` | `MEETING_STARTED`, `MEETING_ENDED`, form/recording triggers |
| Plan gate | **Standard+ (webhooks not on Free)** | Available broadly incl. self-hosted |

FastAPI handler: verify signature → idempotent upsert on booking UID → update journey state → trigger in-app/push confirmation.

## Sandbox / testing

- **Calendly**: no sandbox; use a free/trial org with test advisors. Webhook testing via a dev subscription pointed at a tunnel (ngrok). Careful: real emails go out.
- **Cal.com**: managed — separate test team/org; Platform plans include dev credentials **[verify]**. Self-hosted Cal.diy — run a full staging instance in Docker (Postgres + app) in Mumbai; free.
- Both: test IST timezone edges (half-hour offset!), DST-source invitees, reschedule and cancel loops end-to-end.

## Pricing (July 2026)

**Calendly** (per seat = per advisor):
- Free (1 event type, no webhooks) → **Standard ~$10/seat/mo annual ($12 monthly)** → **Teams ~$16/seat/mo annual ($20 monthly)** — round-robin & routing need Teams → Enterprise custom (~$15k+/yr).
- 5 advisors on Teams ≈ **$80–100/mo**. API itself is free; it rides on seats.

**Cal.com**:
- Managed app: Free individual; **Teams ~$12/seat/mo (annual)**; Organizations ~$30/seat/mo.
- **Platform/API plans** (for embedding into Yesly): **Starter — free tier, 25 bookings/mo included, $0.99/extra booking**; **Essentials — $299/mo, 500 bookings, $0.60 overage**; **Scale — $2,499/mo, 5,000 bookings, SOC2/HIPAA support**. **[verify — pricing page fetch failed; figures from platform pricing page via secondary sources]**
- **Self-hosted Cal.diy: $0 license (MIT)** + your infra/ops. Post-April-2026 caveat: it's a community edition; commercial/enterprise features removed, future feature velocity uncertain.

**Manual**: ~$0 tooling; ops time per booking; no webhooks, no scale.

## Compliance & data residency (DPDP)

- **Calendly**: US-hosted SaaS, no India/EU residency for the core product; invitee name/email/answers live on Calendly servers. DPDP's blacklist regime (Rules notified Nov 2025, full effect ~May 2027) doesn't forbid US transfer today, but for an RIA the booking record ("this user sought advice") is sensitive-adjacent — DPA + minimal prefill (avoid phone, avoid tier/financial context in questions) required.
- **Cal.com managed**: also US-hosted by default **[verify EU hosting option on Organizations plan]**.
- **Self-hosted Cal.diy in AWS Mumbai**: full residency control — the only option that keeps booking PII entirely in India. This is the compliance trump card.
- All options: advisor calendars (Google Workspace) already sit outside India; residency purism has limits — document flows in the DPDP notice either way.

## Comparison & recommendation

| Criterion | Calendly | Cal.com managed/platform | Cal.diy self-hosted | Manual |
|---|---|---|---|---|
| Native in-app booking UX | ✗ (webview only) | ✓ (slots+bookings API) | ✓ | ✓ (custom) |
| Round-robin / routing | Teams plan | ✓ (team event types, API) | ✓ | ✗ |
| Webhooks → journey state | ✓ (Standard+) | ✓ | ✓ | ✗ |
| Data residency (India) | ✗ | ✗ (US) | ✓ | ✓ |
| Ops burden | lowest | low | **high** (you run it) | high (human) |
| Cost at 5 advisors, <200 bookings/mo | ~$100/mo | $0–299/mo (Starter may suffice) | infra ~$50–100/mo + ops | ops time |
| Long-term risk | vendor lock, webview UX | closed-source vendor since 4/2026 | community-edition drift | doesn't scale |

**Recommendation:** 
1. **MVP (low Tier-4 volume): start manual or Calendly Standard** with a single advisor link in a webview — days of effort, validates demand.
2. **Target state: Cal.com Platform (Starter → Essentials)** — the slots/bookings API gives a fully **native** in-app booking + confirmation flow (no webview), round-robin across advisors, and clean webhooks into FastAPI. Keep **self-hosted Cal.diy** as the exit if DPDP localization mandates or costs bite — the API-shaped integration code mostly carries over.
3. Avoid building deep on Calendly: no booking-creation API (webview forever) and no residency path.

## Gotchas & effort

1. **Webview UX (Calendly)**: cookie/consent banners, keyboard jank, no control over loading states; deep-link back to app after booking is fragile — rely on the webhook, not the webview callback, as source of truth.
2. **Timezones**: send/store UTC; render Asia/Kolkata explicitly (UTC+5:30 trips half-hour bugs); invitee-side DST (NRI users) — always pass `timeZone` on Cal.com bookings.
3. **Reschedules**: Calendly models them as cancel+create (join via `old_invitee`) — handle idempotently or journey state flaps; Cal.com's distinct `BOOKING_RESCHEDULED` is cleaner but keep the booking `uid` chain.
4. **No-shows**: both require the advisor to mark no-show manually; build an ops nudge (post-`MEETING_ENDED` check) if no-show handling matters for Tier-4 follow-up.
5. **Cal.com fork risk**: pin to API v2 with an explicit `cal-api-version`; watch Cal.diy repo health before betting self-hosted.
6. **Advisor calendar hygiene**: round-robin is only as good as connected calendars; stale Google tokens silently shrink availability — monitor via webhook gaps.

**Effort ballpark**: Calendly webview + webhook ≈ 2–3 dev-days. Cal.com native picker + booking + webhooks ≈ 5–8 dev-days. Self-hosting Cal.diy adds ~1 week setup + ongoing ops. Manual ≈ 1 day of form + ops process.

## Sources

- https://developer.calendly.com/getting-started ; https://developer.calendly.com/api-docs (API v2; "Now Available: Scheduling API")
- https://developer.calendly.com/receive-data-from-scheduled-events-in-real-time-with-webhook-subscriptions (webhook events)
- https://calendly.com/help/webhooks-overview (plan gating) ; https://calendly.com/pricing
- https://cal.com/docs/api-reference/v2/introduction (API v2, cal-api-version)
- https://cal.com/platform/pricing (Platform Starter/Essentials/Scale) ; https://cal.com/pricing
- https://cal.com/blog/cal-com-goes-closed-source-why and https://cal.com/blog/cal-diy-open-source-to-closed-source (April 2026 closed-source move; Cal.diy MIT)
- https://thenewstack.io/cal-com-codebase-security-ai/ (third-party coverage of the fork)
- DPDP: https://www.dpdpa.com/dpdparules/rule15.html (cross-border blacklist approach)
