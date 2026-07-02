# 08 — Amazon SES (Simple Email Service)

**Doc status:** Researched July 2026 against official AWS documentation. Items that could only be confirmed via secondary sources (AWS pricing page not directly fetchable at research time) are marked **verify before build**.

---

## Overview & role in Yesly

Amazon SES is AWS's transactional + bulk email service. For Yesly it is the **email leg of Container 5 (async notification dispatcher)** — the Celery workers that already fan out push (FCM/APNs) and WhatsApp will call SES via the **boto3 `sesv2` client** for:

- **Transactional email**: OTP/login alerts, KYC/onboarding status, advisory nudge summaries, goal/portfolio alerts, statements and SEBI-mandated disclosures.
- **Marketing/lifecycle email** (separate stream): onboarding drip, feature announcements, re-engagement — only for users with explicit DPDP-compliant consent.

Why SES fits Yesly:

- **Same region as the stack**: SES API v2 is fully available in **ap-south-1 (Mumbai)** — API endpoint `email.ap-south-1.amazonaws.com`, SMTP endpoint `email-smtp.ap-south-1.amazonaws.com` (verified July 2026). Email content stays in-region; no extra egress hop.
- **Cheapest per-email price** of any mainstream ESP (~$0.10 per 1,000), no platform fee.
- **IAM-native auth** — no API keys to rotate; Celery workers use the task role.
- Trade-off: SES is a raw pipe. Template management UI, campaign tooling, and suppression UX are minimal — Yesly owns bounce/complaint handling logic.

---

## How it works

```
                        YESLY BACKEND (ap-south-1)
┌────────────┐   enqueue    ┌─────────────────────┐
│ FastAPI    │ ───────────▶ │ Container 5          │
│ (trigger:  │  (Redis/SQS) │ Celery email worker  │
│ nudge, OTP)│              │ boto3 client("sesv2")│
└────────────┘              └──────────┬──────────┘
                                       │ SendEmail (SigV4, HTTPS)
                                       ▼
                            ┌─────────────────────┐
                            │ Amazon SES v2        │
                            │ email.ap-south-1     │──▶ MessageId returned
                            │ ConfigurationSet:    │
                            │  yesly-transactional │
                            └──────────┬──────────┘
                                       │ SMTP delivery
                                       ▼
                            ┌─────────────────────┐
                            │ Recipient mailbox    │
                            │ (Gmail/Outlook/...)  │
                            └──────────┬──────────┘
                                       │ bounce / complaint / open / click
                                       ▼
        ┌──────────────────────────────────────────────────┐
        │ Event destinations on the configuration set:      │
        │  SNS ──▶ SQS ──▶ Celery "email-events" worker     │
        │  EventBridge ──▶ rules ──▶ Lambda / alerting      │
        │  Kinesis Firehose ──▶ S3 (audit archive)          │
        │  CloudWatch ──▶ bounce/complaint-rate alarms      │
        └──────────────────────┬───────────────────────────┘
                               ▼
                 ┌─────────────────────────────┐
                 │ Suppression handling:        │
                 │  hard bounce / complaint ─▶  │
                 │  mark user email undeliver-  │
                 │  able in Postgres + rely on  │
                 │  SES account suppression list│
                 └─────────────────────────────┘
```

Flow summary: Celery worker calls `SendEmail` → SES returns `MessageId` (accepted, not delivered) → SES attempts delivery → lifecycle events (`SEND`, `DELIVERY`, `BOUNCE`, `COMPLAINT`, `OPEN`, `CLICK`, `REJECT`, `RENDERING_FAILURE`, `DELIVERY_DELAY`, `SUBSCRIPTION`) publish to the configuration set's event destinations → Yesly's event consumer updates delivery status and suppression state keyed by `MessageId`.

---

## Key endpoints

SES v2 REST API (all called through boto3 `sesv2` client; SigV4-signed HTTPS against `email.ap-south-1.amazonaws.com`).

| Method | Path / SDK call | Purpose | Key request fields | Key response fields |
|---|---|---|---|---|
| POST | `/v2/email/outbound-emails` — `sesv2.send_email()` | Send one email (Simple, Raw, or Templated) | `FromEmailAddress`, `Destination.ToAddresses[]` (+`Cc`/`Bcc`), `Content.Simple{Subject,Body{Html,Text}}` **or** `Content.Template{TemplateName,TemplateData}` **or** `Content.Raw{Data}`, `ConfigurationSetName`, `EmailTags[]`, `ReplyToAddresses[]`, `FeedbackForwardingEmailAddress`, `ListManagementOptions` | `MessageId` |
| POST | `/v2/email/outbound-bulk-emails` — `sesv2.send_bulk_email()` | Batch templated send (up to 50 destinations per call) | `FromEmailAddress`, `DefaultContent.Template{TemplateName,TemplateData}`, `BulkEmailEntries[]{Destination, ReplacementEmailContent}`, `ConfigurationSetName` | `BulkEmailEntryResults[]{Status, MessageId}` per recipient |
| POST | `/v2/email/templates` — `sesv2.create_email_template()` | Create a stored template | `TemplateName`, `TemplateContent{Subject, Html, Text}` (Handlebars-style `{{variable}}` placeholders) | — (200 on success) |
| PUT/GET/DELETE | `/v2/email/templates/{TemplateName}` — `update/get/delete_email_template()` | Manage templates | `TemplateName`, `TemplateContent` | `TemplateContent` (GET) |
| POST | `/v2/email/configuration-sets` — `create_configuration_set()` | Create config set (per stream: transactional / marketing) | `ConfigurationSetName`, `TrackingOptions`, `SuppressionOptions{SuppressedReasons}`, `ReputationOptions` | — |
| POST | `/v2/email/configuration-sets/{name}/event-destinations` — `create_configuration_set_event_destination()` | Attach event destination | `EventDestination{MatchingEventTypes[], SnsDestination / EventBridgeDestination / KinesisFirehoseDestination / CloudWatchDestination / PinpointDestination}` | — |
| GET | `/v2/email/suppression/addresses/{email}` — `get_suppressed_destination()` | Check if an address is on account suppression list | `EmailAddress` | `SuppressedDestination{Reason: BOUNCE\|COMPLAINT, LastUpdateTime}` |
| DELETE | `/v2/email/suppression/addresses/{email}` — `delete_suppressed_destination()` | Remove address from suppression list | `EmailAddress` | — |
| PUT | `/v2/email/account/suppression` — `put_account_suppression_attributes()` | Set account-level auto-suppression reasons | `SuppressedReasons: ["BOUNCE","COMPLAINT"]` | — |
| GET | `/v2/email/account` — `get_account()` | Sending quota, sandbox status, send rate | — | `SendQuota{Max24HourSend, MaxSendRate, SentLast24Hours}`, `ProductionAccessEnabled` |

### Example — templated SendEmail request (what the Celery worker sends)

```json
POST /v2/email/outbound-emails
{
  "FromEmailAddress": "Yesly <alerts@mail.yesly.in>",
  "Destination": { "ToAddresses": ["investor@example.com"] },
  "Content": {
    "Template": {
      "TemplateName": "yesly-goal-nudge-v1",
      "TemplateData": "{\"first_name\":\"Priya\",\"goal_name\":\"Retirement\",\"gap_pct\":\"12\"}"
    }
  },
  "ConfigurationSetName": "yesly-transactional",
  "EmailTags": [
    { "Name": "notification_type", "Value": "goal_nudge" },
    { "Name": "user_id_hash", "Value": "u_9f8e7d" }
  ],
  "ReplyToAddresses": ["support@yesly.in"]
}
```

Response:

```json
{ "MessageId": "010f0197a1b2c3d4-5e6f7a8b-9c0d-1e2f-3a4b-5c6d7e8f9a0b-000000" }
```

`MessageId` = accepted by SES, **not** delivered. Persist it against the notification row; delivery/bounce events reference the same `mail.messageId`.

### Example — CreateEmailTemplate (Handlebars personalization)

```json
POST /v2/email/templates
{
  "TemplateName": "yesly-goal-nudge-v1",
  "TemplateContent": {
    "Subject": "{{first_name}}, your {{goal_name}} goal needs attention",
    "Html": "<html><body><p>Hi {{first_name}},</p><p>Your <b>{{goal_name}}</b> goal is {{gap_pct}}% behind plan.</p><p><a href=\"{{cta_url}}\">Review in Yesly</a></p></body></html>",
    "Text": "Hi {{first_name}}, your {{goal_name}} goal is {{gap_pct}}% behind plan. Review: {{cta_url}}"
  }
}
```

Handlebars notes: `{{var}}` substitution, `{{#if}}`/`{{#each}}` blocks supported. If `TemplateData` is missing a referenced attribute, SES **accepts the send (returns a MessageId) but does not deliver** — it emits a `RENDERING_FAILURE` event instead. You must subscribe to that event type or these failures are silent.

### boto3 (Python / FastAPI / Celery)

```python
import boto3

ses = boto3.client("sesv2", region_name="ap-south-1")

resp = ses.send_email(
    FromEmailAddress="Yesly <alerts@mail.yesly.in>",
    Destination={"ToAddresses": [user.email]},
    Content={"Template": {
        "TemplateName": "yesly-goal-nudge-v1",
        "TemplateData": json.dumps(payload),
    }},
    ConfigurationSetName="yesly-transactional",
    EmailTags=[{"Name": "notification_type", "Value": "goal_nudge"}],
)
message_id = resp["MessageId"]
```

Retry guidance: catch `TooManyRequestsException` (HTTP 429, send-rate exceeded) with exponential backoff in the Celery task; treat `MessageRejected`, `AccountSuspendedException`, `SendingPausedException` as non-retryable alerts.

---

## Auth

- **No API keys.** SES v2 API calls are standard AWS **SigV4** requests; boto3 signs automatically using the container's IAM role (ECS task role / EKS IRSA / EC2 instance profile). Never bake long-lived access keys into Container 5.
- (SMTP interface is the exception — it uses derived SMTP credentials — but Yesly should use the API, not SMTP.)
- Least-privilege policy for the dispatcher worker:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "YeslyEmailSend",
      "Effect": "Allow",
      "Action": ["ses:SendEmail", "ses:SendBulkEmail"],
      "Resource": [
        "arn:aws:ses:ap-south-1:ACCOUNT_ID:identity/mail.yesly.in",
        "arn:aws:ses:ap-south-1:ACCOUNT_ID:configuration-set/yesly-transactional",
        "arn:aws:ses:ap-south-1:ACCOUNT_ID:configuration-set/yesly-marketing",
        "arn:aws:ses:ap-south-1:ACCOUNT_ID:template/*"
      ],
      "Condition": {
        "StringEquals": { "ses:FromAddress": "alerts@mail.yesly.in" }
      }
    },
    {
      "Sid": "YeslySuppressionOps",
      "Effect": "Allow",
      "Action": ["ses:GetSuppressedDestination", "ses:PutSuppressedDestination", "ses:DeleteSuppressedDestination"],
      "Resource": "*"
    }
  ]
}
```

- Note: SES v2 `SendEmail` authorizes against the **identity, configuration set, and template ARNs** used in the call — include all three resource types. Template/suppression management (`ses:CreateEmailTemplate`, etc.) belongs in a separate CI/admin role, not the runtime worker role.
- Condition keys worth using: `ses:FromAddress`, `ses:FromDisplayName`, `ses:Recipients`.

---

## Webhooks/events

SES has **no direct HTTP webhook**. Events flow through **event destinations attached to a configuration set** (this is why every send must pass `ConfigurationSetName`):

| Destination | Use for Yesly |
|---|---|
| **Amazon SNS** | Primary: SNS topic → SQS queue → Celery `email-events` consumer (or SNS HTTPS subscription straight to a FastAPI webhook endpoint — must handle SNS `SubscriptionConfirmation` + signature verification) |
| **Amazon EventBridge** | Rules-based routing, e.g. `eventType = Bounce` → Lambda that flags the user record; good for ops alerting |
| **Kinesis Data Firehose** | Stream all events to S3 for SEBI/audit-friendly retention and analytics |
| **CloudWatch** | Metric dimensions from `EmailTags` → bounce-rate / complaint-rate alarms (mandatory before production ramp) |

**Event types** (per SES event publishing): `SEND`, `DELIVERY`, `BOUNCE`, `COMPLAINT`, `OPEN`, `CLICK`, `REJECT`, `RENDERING_FAILURE`, `DELIVERY_DELAY`, `SUBSCRIPTION`.

### Bounce event JSON (condensed real shape, verified against AWS docs)

```json
{
  "eventType": "Bounce",
  "bounce": {
    "bounceType": "Permanent",
    "bounceSubType": "General",
    "bouncedRecipients": [
      {
        "emailAddress": "recipient@example.com",
        "action": "failed",
        "status": "5.1.1",
        "diagnosticCode": "smtp; 550 5.1.1 user unknown"
      }
    ],
    "timestamp": "2026-07-02T00:41:02.669Z",
    "feedbackId": "01000157c44f053b-61b59c11-...-000000",
    "reportingMTA": "dsn; mta.example.com"
  },
  "mail": {
    "timestamp": "2026-07-02T00:40:02.012Z",
    "source": "Yesly <alerts@mail.yesly.in>",
    "sendingAccountId": "123456789012",
    "messageId": "010f0197a1b2c3d4-...-000000",
    "destination": ["recipient@example.com"],
    "tags": {
      "ses:configuration-set": ["yesly-transactional"],
      "notification_type": ["goal_nudge"]
    }
  }
}
```

Handling rules for the event consumer:

- `Bounce` with `bounceType: "Permanent"` → mark email undeliverable in Postgres immediately (SES also auto-adds it to the account suppression list). `Transient` (e.g. mailbox full) → retry later, do not suppress.
- `Complaint` → suppress + opt user out of the stream that triggered it. **Never** send a "sorry you unsubscribed" email.
- `Rendering Failure` → page the on-call; the message silently didn't go out.
- Correlate via `mail.messageId` == the `MessageId` returned by `SendEmail`.

### Suppression list

- **Account-level** (global): SES auto-adds addresses on hard bounce and/or complaint (configurable via `PutAccountSuppressionAttributes`, default both). Sends to suppressed addresses are counted/billed but not delivered — they surface as bounce events with `bounceSubType: "OnSuppressionList"`.
- Query/remove: `GetSuppressedDestination`, `ListSuppressedDestinations`, `DeleteSuppressedDestination` (e.g. after a user fixes a typo'd address).
- **Configuration-set-level override**: `SuppressionOptions.SuppressedReasons` on each config set can override the account default — e.g. marketing set suppresses on `BOUNCE` + `COMPLAINT`, transactional set could suppress on `BOUNCE` only (OTP/security email arguably should still attempt). Decide per stream.
- Yesly should still keep its **own** suppression state in Postgres (source of truth for UI + DPDP consent records); treat SES's list as a safety net.

---

## Sandbox/testing

**Sandbox** (every new SES account, per region — verified July 2026):

- Can send **only to verified addresses/domains** or the mailbox simulator.
- Max **200 messages / 24 h**, max **1 message/second**.
- Suppression-list bulk actions and related API calls are **disabled** in sandbox.
- All features otherwise work (templates, config sets, events) — so full integration testing is possible in sandbox.

**Production access request** (per region): SES console → Account dashboard → "Request production access". You must provide: mail type (Transactional / Marketing), website URL, up to 4 contact emails, and check an acknowledgement that you (a) only email recipients who explicitly requested it and (b) have a bounce/complaint handling process. Also scriptable: `aws sesv2 put-account-details --production-access-enabled ...`. **AWS Support responds within 24 hours**; can take longer if they ask follow-ups. Verifying the sending domain first speeds approval. Initial production quota is commonly ~50,000 emails/day (grows automatically with healthy sending; also raisable via Service Quotas) — **verify your granted quota in the console** via `GetAccount`.

**Mailbox simulator** (free of charge, works in sandbox, doesn't hurt reputation metrics):

| Address | Simulates |
|---|---|
| `success@simulator.amazonses.com` | Successful delivery (DELIVERY event) |
| `bounce@simulator.amazonses.com` | Hard bounce (SMTP 550 5.1.1) |
| `ooto@simulator.amazonses.com` | Out-of-the-office auto-reply |
| `complaint@simulator.amazonses.com` | Spam complaint (COMPLAINT event) |
| `suppressionlist@simulator.amazonses.com` | Recipient on suppression list |

Use these in CI to exercise the full event-consumer path (bounce → suppression write) without real mailboxes. Local unit tests: `moto` mocks `sesv2` reasonably well; `botocore.stub.Stubber` for exact request assertions.

**Dedicated vs shared IPs**: Start on **shared IPs** (default, $0 extra, reputation managed by AWS — fine at Yesly's early volume). Dedicated IPs only make sense at sustained high volume (rule of thumb: several hundred thousand emails/month+, steady daily cadence). *Standard* dedicated IPs ($24.95/mo each — verify) require **manual warm-up** (~45-day automated warm-up plan available, but you own the reputation). *Managed* dedicated IPs (monthly fee + per-email usage) auto-scale the pool and auto-warm. Defer both until volume justifies it.

---

## Pricing

FX assumption: **USD 1 = INR 86** (July 2026 ballpark — restate at contract time).

| Item | USD | INR (approx) | Status |
|---|---|---|---|
| Outbound email (API/SMTP) | **$0.10 / 1,000 emails** | ~₹8.6 / 1,000 | Stable for years; 2026 secondary sources confirm — **verify on aws.amazon.com/ses/pricing before build** |
| Attachments / message data | **$0.12 / GB** of attachments sent | ~₹10.3 / GB | Same — verify |
| Inbound email | $0.10 / 1,000 + per-GB chunk charges | ~₹8.6 / 1,000 | Not needed for Yesly v1 |
| Dedicated IP (standard) | **$24.95 / mo per IP** | ~₹2,150 / mo | Last verified via 2026 secondary sources — verify |
| Dedicated IPs (managed) | ~$15 / mo + tiered usage (~$0.08/1k up to 10M, less above) | ~₹1,290 / mo + usage | Secondary-source figures — **verify before build** |
| Virtual Deliverability Manager | **$0.07 / 1,000 emails** (<10M/mo; tiers down to $0.02 at >100M) + $0.0005/1k dashboard queries after first 5k free | ~₹6 / 1,000 | Verify; optional — skip at launch |
| Free tier | Accounts created **before 15 Jul 2025**: 3,000 message charges/mo free for first 12 months (the old 62k/mo EC2-sourced free tier was retired Aug 2023). Accounts created **on/after 15 Jul 2025**: no SES-specific free tier — instead up to **$200 in general AWS Free Tier credits** ($100 signup + $100 activity-based), usable on SES | — | Verified against AWS Free Tier update announcement; check which regime Yesly's account falls under |

**Yesly ballpark** (transactional-heavy, no dedicated IP, no VDM): 50k users × ~10 emails/mo = 500k emails/mo → **$50/mo ≈ ₹4,300/mo**. Even at 5M emails/mo it's $500 ≈ ₹43,000/mo. Email cost is a rounding error next to WhatsApp per-conversation pricing — SES is effectively free at Yesly's scale. Data transfer for HTML-only email is negligible; only attachment-heavy statements (PDFs) add the $0.12/GB line.

---

## Compliance notes (India)

- **DPDP Act 2023**: marketing email is processing of personal data (email address) for a secondary purpose → needs **explicit, granular, recorded consent**, separate from transactional/service messages. Store consent artefacts (timestamp, source, purpose) in Yesly's DB; honour withdrawal immediately. Transactional/advisory-mandated messages (SEBI RIA disclosure, OTP) stand on the service-necessity footing — keep them on the separate `yesly-transactional` stream so a marketing opt-out never blocks them.
- **Unsubscribe**: every marketing email must carry a working one-click unsubscribe. Options: SES v2 **subscription management** (`ListManagementOptions` + contact lists auto-inserts an unsubscribe link and emits `SUBSCRIPTION` events) or Yesly's own unsubscribe endpoint + `List-Unsubscribe`/`List-Unsubscribe-Post` headers. Gmail/Yahoo bulk-sender rules (enforced since Feb 2024) require one-click `List-Unsubscribe-Post` for bulk senders (~5k+/day to Gmail) plus aligned SPF/DKIM/DMARC — treat as mandatory, not optional.
- **Data residency**: sending from **ap-south-1** keeps message content and event data processing in the Mumbai region — consistent with Yesly's ap-south-1 posture and SEBI/RBI-adjacent data-localisation expectations. Note DPDP does not currently mandate hard localisation for this data class, but in-region is the defensible default for an RIA. (Recipient mailboxes are, of course, wherever the user's provider is.)
- **SEBI RIA context**: advisory communications may need retention — Firehose → S3 (ap-south-1, versioned, lifecycle-locked) archive of SEND/DELIVERY events plus rendered-template versioning gives an audit trail of what was sent to whom, when.
- **Deliverability prerequisites (also compliance-adjacent)**: domain identity with **Easy DKIM (2048-bit** selectable, default — use it), **SPF** via a **custom MAIL FROM domain** (e.g. `mail.yesly.in` with MX → `feedback-smtp.ap-south-1.amazonses.com` and SPF `v=spf1 include:amazonses.com ~all`) so SPF aligns with the From domain, and a **DMARC** record (start `p=none` with `rua=` reporting, ratchet to `p=quarantine/reject`). Without the custom MAIL FROM, SPF authenticates against `amazonses.com` and DMARC SPF-alignment fails (DKIM alignment still passes, but do both).

---

## Gotchas & effort

**Gotchas**

1. **`MessageId` ≠ delivered.** SES can accept and then bounce, suppress, or fail rendering. Status is only knowable from events — build the event consumer in the same sprint as sending, not "later".
2. **Rendering failures are silent** on templated sends: valid `MessageId`, no email. Always subscribe to `RENDERING_FAILURE`; validate `TemplateData` against a schema before enqueueing.
3. **Sandbox is per region** and quota state is per region — production access in ap-south-1 doesn't carry to us-east-1 (relevant if a DR region is added).
4. **Quota ramp-up**: production typically starts around 50k/day and 14 msg/sec (verify granted values via `GetAccount`); it grows automatically with healthy volume, but a big-bang campaign on day one will hit `TooManyRequestsException`. Rate-limit the Celery worker below `MaxSendRate` and request quota increases ahead of launches.
5. **Reputation thresholds are account-wide** (verified July 2026 from SES enforcement FAQ): keep bounce rate **< 2%** (best practice); **≥ 5% → account under review**, **≥ 10% → sending pause risk**. Complaints: keep **< 0.1%**; **≥ 0.1% → review**, **≥ 0.5% → pause risk**. One bad marketing blast can pause OTP email for the whole product — this is the strongest argument for separate config sets **and ideally a separate subdomain** (`mail.yesly.in` transactional vs `news.yesly.in` marketing); consider a separate AWS account for marketing at scale.
6. **Cold IP warm-up** only applies if/when moving to standard dedicated IPs — plan a 2–6 week ramp or use managed IPs. Irrelevant on shared IPs.
7. **Open/click tracking rewrites links** through the regional tracking domain (`r.ap-south-1.awstrack.me`) and injects a tracking pixel — corporate filters sometimes flag the awstrack.me redirect. Configure a **custom tracking (redirect) domain** (CNAME, e.g. `click.yesly.in`) via `TrackingOptions` on the config set; keep tracking OFF for OTP/security emails.
8. **Suppression-list sends still bill**: sends to suppressed addresses count against quota and cost money. Check Yesly's own suppression table before enqueueing.
9. **Bounce-rate math counts only unverified-domain hard bounces**, over a floating "representative volume" — you cannot exactly reproduce AWS's number; set CloudWatch alarms at conservative levels (e.g. bounce > 3%, complaint > 0.05%).
10. **Verify identity + custom MAIL FROM before production request** — approval is faster with a verified domain, and DNS propagation (DKIM CNAMEs, MX, SPF, DMARC) takes time; do it first.

**Estimated integration effort** (given Container 5 dispatcher skeleton already exists for push/WhatsApp):

| Work item | Effort |
|---|---|
| Domain identity, Easy DKIM 2048, custom MAIL FROM, SPF/DMARC DNS | 0.5 day (+ DNS propagation wait) |
| Config sets (transactional + marketing) + SNS→SQS event wiring (IaC) | 0.5–1 day |
| Celery send task: boto3 sesv2, templated send, retries/rate-limit | 1 day |
| Event consumer: bounce/complaint/suppression + status updates | 1.5–2 days |
| Template CI (CreateEmailTemplate from repo, versioning) + 3–5 base templates | 1–1.5 days |
| Unsubscribe flow + DPDP consent gating for marketing stream | 1–2 days |
| CloudWatch reputation alarms + Firehose→S3 audit archive | 0.5–1 day |
| Sandbox testing (simulator addresses) + production-access request | 0.5 day (+ ~24 h AWS review) |
| **Total** | **~6–9 engineer-days** |

---

## Sources

Official AWS (fetched/verified July 2026):

- SES v2 SendEmail API reference — https://docs.aws.amazon.com/ses/latest/APIReference-V2/API_SendEmail.html
- SES v2 API reference (SendBulkEmail, CreateEmailTemplate, suppression ops) — https://docs.aws.amazon.com/ses/latest/APIReference-V2/Welcome.html
- Request production access (sandbox limits: verified recipients, 200/day, 1 msg/sec; 24 h review) — https://docs.aws.amazon.com/ses/latest/dg/request-production-access.html
- Sending review process FAQ (bounce 5%/10%, complaint 0.1%/0.5% thresholds) — https://docs.aws.amazon.com/ses/latest/dg/faqs-enforcement.html
- Event publishing examples (bounce/complaint/open/click JSON) — https://docs.aws.amazon.com/ses/latest/dg/event-publishing-retrieving-sns-examples.html
- Monitoring using event publishing (destinations, event types) — https://docs.aws.amazon.com/ses/latest/dg/monitor-using-event-publishing.html
- Account-level suppression list — https://docs.aws.amazon.com/ses/latest/dg/sending-email-suppression-list.html
- Mailbox simulator — https://docs.aws.amazon.com/ses/latest/dg/send-an-email-from-console.html#send-email-simulator
- Dedicated IPs (standard vs managed) — https://docs.aws.amazon.com/ses/latest/dg/dedicated-ip.html
- Easy DKIM / custom MAIL FROM / BYODKIM — https://docs.aws.amazon.com/ses/latest/dg/send-email-authentication-dkim-easy.html , https://docs.aws.amazon.com/ses/latest/dg/mail-from.html
- Virtual Deliverability Manager — https://docs.aws.amazon.com/ses/latest/dg/vdm.html
- SES endpoints & quotas (ap-south-1 confirmed: `email.ap-south-1.amazonaws.com`, SMTP, tracking domain) — https://docs.aws.amazon.com/general/latest/gr/ses.html
- SES pricing (page not directly fetchable at research time — **verify all $ figures here before build**) — https://aws.amazon.com/ses/pricing/
- AWS Free Tier update (post-15-Jul-2025 accounts: up to $200 credits) — https://aws.amazon.com/blogs/aws/aws-free-tier-update-new-customers-can-get-started-and-explore-aws-with-up-to-200-in-credits/

Secondary (pricing cross-checks only; 2026): smtpedia.com/amazon-aws-ses-pricing/ , costgoat.com/pricing/amazon-ses , blog.campaignhq.co/amazon-ses-pricing-2026/
