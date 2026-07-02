# 15 — Supabase Community Bridge (YesLifers → Yesly identity handoff)

## Overview & role in Yesly

The YesLifers community app (a separate product from the same company) runs on **Supabase** (Postgres + Supabase Auth). Yesly runs on **AWS Cognito** (Mumbai). The flow doc specifies a **one-way cross-system identity bridge**:

- A "warm-funnel" user arrives in Yesly from the community (deep link / campaign).
- Yesly's FastAPI backend reads that user's `COMMUNITY_PROFILE` row from Supabase.
- It propagates **email, display name, avatar, and `pledge_taken`** into Yesly's `USER` table and creates the Cognito identity via `AdminCreateUser` (+ `AUTH_SESSION`).
- Warm users with `pledge_taken = true` **skip the pledge screens (P1–P3)**.

The bridge is **read-mostly** (Yesly pulls from Supabase; nothing writes back in v1), **server-side only** (the RN app never talks to Supabase), and **low-volume** (one lookup per warm signup, plus optional change-sync).

## How it works (text flow diagram)

```
[YesLifers app / community site]
        |  deep link: yesly://warm?src=community&e=<email or token>
        v
[Yesly RN app] ---> POST /auth/warm-handoff {email | supabase_jwt}
        |
        v
[Yesly FastAPI, AWS Mumbai]
        |  (a) if JWT present: verify against Supabase JWKS  -> trusted email
        |  (b) GET https://<ref>.supabase.co/rest/v1/community_profile
        |        ?email=eq.<email>&select=id,email,display_name,avatar_url,pledge_taken
        |        headers: apikey + Authorization (SERVICE ROLE / secret key — server only)
        v
[Supabase project (choose ap-south-1 Mumbai)] --> returns 0 or 1 profile rows
        |
        v
[FastAPI] -- found? --> cognito.admin_create_user(email, name, custom:pledge_taken,
        |                 MessageAction="SUPPRESS") -> USER row + identity_map row
        |                 (identity_map: yesly_user_id <-> supabase_profile_id UUID)
        |               -> copy avatar file S3 <- Supabase Storage URL
        |               -> flow flags: skip P1–P3 if pledge_taken
        |
        +-- not found --> normal cold-funnel signup (pledge screens shown)

[Optional sync path]
[Supabase DB webhook on community_profile UPDATE] --pg_net POST-->
        [FastAPI /webhooks/supabase/profile-changed] --> idempotent upsert
        (keeps pledge_taken / display_name fresh after the initial pull)
```

## 1. Supabase platform overview — what matters for a read-mostly bridge

| Piece | What it is | Relevant to the bridge? |
|---|---|---|
| **Postgres** | The actual database (one per project) | Yes — source of truth for `community_profile` |
| **PostgREST REST API** | Auto-generated REST over every table/view at `/rest/v1/` | **Yes — primary integration surface** |
| **Auth (GoTrue)** | JWT-issuing auth server at `/auth/v1/` | Only if the community app deep-links with a user JWT (option d) |
| **Row Level Security (RLS)** | Postgres policies gating row access per role/JWT | Yes — understand what the service key bypasses |
| **Realtime** | WebSocket change feeds | No — overkill for one-shot lookups; webhooks cover sync |
| **Edge Functions** | Deno functions at `/functions/v1/` | Optional — could host a purpose-built `GET /bridge/profile` instead of raw table access |
| **Storage** | S3-like object store at `/storage/v1/` (public or signed URLs) | Yes — where the avatar file lives |
| **Database Webhooks (pg_net)** | Triggers that POST row changes to a URL | Yes — option (b) for pledge_taken sync |

## 2. REST API I/O

### Query the profile

```
GET https://<project-ref>.supabase.co/rest/v1/community_profile
      ?email=eq.vatsal%40example.com
      &select=id,email,display_name,avatar_url,pledge_taken,updated_at
apikey: <service key>
Authorization: Bearer <service key>          # legacy JWT keys; see key note below
Accept: application/json
```

Response `200`:

```json
[
  {
    "id": "7f7f0e0a-9c3b-4f4e-a2f1-2c9d1a1b2c3d",
    "email": "vatsal@example.com",
    "display_name": "Vatsal P",
    "avatar_url": "https://<ref>.supabase.co/storage/v1/object/public/avatars/7f7f....jpg",
    "pledge_taken": true,
    "updated_at": "2026-06-28T09:12:33.000Z"
  }
]
```

Notes on the surface:

- **Always an array** (even for one row). Add `Accept: application/vnd.pgrst.object+json` to get a single object and a `406` when 0 or >1 rows — a nice guard against duplicate emails.
- **Filters**: `col=eq.value`, `ilike.*pattern*`, `in.(a,b)`, `is.null`; combine with `&`. For emails use `ilike` or a normalized (lowercased) column — see Gotchas.
- **Column selection** via `select=` — request *only* the four fields the bridge needs (data minimization, see DPDP).
- **Pagination**: `limit`/`offset` params or `Range: 0-9` header; `Prefer: count=exact` returns total in `Content-Range`. Irrelevant for single-row lookups, useful for a backfill job.
- **Writes** (not needed v1): `POST /rest/v1/table` with `Prefer: resolution=merge-duplicates` for upserts.

### API keys: publishable/secret vs legacy anon/service_role

| Key | Format | RLS | Where it may live |
|---|---|---|---|
| Publishable (new) | `sb_publishable_...` | Subject to RLS | Client apps (not needed by Yesly) |
| **Secret (new)** | `sb_secret_...` | **Bypasses RLS** (`BYPASSRLS`) | **FastAPI / Secrets Manager only** |
| anon (legacy) | JWT | Subject to RLS | Client apps |
| service_role (legacy) | JWT | **Bypasses RLS** | Server only |

- The bridge uses the **secret/service_role key, server-side ONLY — never in the RN app, never in Expo config, never in a repo**. It reads any row in the entire community database.
- Header quirk: legacy JWT keys are sent as both `apikey` and `Authorization: Bearer <key>`. The **new `sb_secret_...` keys go in `apikey`** and cannot be used as a Bearer token unless the two headers are identical — check which key format the YesLifers project has enabled and prefer the new secret keys (revocable individually, no JWT expiry baggage).
- **RLS implication**: because the key bypasses RLS, the *only* access control is your FastAPI code. Constrain blast radius: create a **Postgres view** (e.g. `bridge_profile_v` exposing exactly the 5 columns) or a dedicated Postgres role with `SELECT` on that view only, and query the view — then even a bug in FastAPI can't read DMs or private community content. Alternatively, wrap access in a Supabase **Edge Function** with its own shared-secret auth so the raw service key never leaves Supabase.

## 3. Auth interop options for the bridge

**(a) Backend-to-backend REST pull at warm-handoff — recommended.** FastAPI hits PostgREST with the secret key exactly when a warm user signs up. Simplest possible: no infra on the Supabase side, no queues, ~30 lines of Python. Add a 2–5 s timeout + graceful fallback ("treat as cold funnel, show pledge screens") so a Supabase outage never blocks Yesly signups.

**(b) Supabase Database Webhooks → FastAPI — additive, for freshness.** A webhook on `community_profile` UPDATE/INSERT posts changes to `POST /webhooks/supabase/profile-changed`. Keeps `pledge_taken`/display name in sync *after* the initial pull (e.g., user takes the pledge in the community *after* joining Yesly). See §4.

**(c) Direct Postgres connection (Supavisor pooler) — only if justified.** Connect FastAPI's SQLAlchemy straight to the community DB via the pooler (`aws-0-ap-south-1.pooler.supabase.com`, port `6543` transaction mode / `5432` session mode; direct `db.<ref>.supabase.co:5432` is IPv6-first — [verify before build] IPv4 add-on status). Justified only for bulk backfills/analytics joins across thousands of rows where REST round-trips hurt. For one-row lookups it adds credential management, connection pool tuning, and schema-coupling risk for zero benefit. Skip.

**(d) Verifying a Supabase JWT in FastAPI — if the deep link carries a token.** If the community app deep-links `yesly://warm?token=<supabase_access_token>`, Yesly can verify the user *proved* ownership of the community account (stronger than trusting an email in a URL):

- **New asymmetric keys (recommended)**: verify RS256/ES256 against the project JWKS at `GET https://<ref>.supabase.co/auth/v1/.well-known/jwks.json` (cache ≤10 min; use `PyJWT` + `PyJWKClient` or `jose`). Claims: `sub` (Supabase user UUID — use this for the mapping table), `email`, `role`, `exp`, `iss`.
- **Legacy HS256**: verify with the project's shared JWT secret — works, but sharing that secret with Yesly's backend is a liability; Supabase itself recommends migrating to asymmetric keys, or simply calling `GET /auth/v1/user` with the token (200 = valid) if you'd rather not hold any secret.
- Without a token, email-in-deep-link is spoofable — mitigate by only *pre-filling* from the community profile and still requiring Cognito email OTP verification before honoring `pledge_taken`.

## 4. Webhooks (Database Webhooks / pg_net)

Configured in Dashboard → Database → Webhooks (or SQL `CREATE TRIGGER ... supabase_functions.http_request(...)`). They are literally AFTER triggers that enqueue an async HTTP call via the `pg_net` extension.

Payload POSTed to FastAPI:

```json
{
  "type": "UPDATE",
  "table": "community_profile",
  "schema": "public",
  "record": {
    "id": "7f7f0e0a-...", "email": "vatsal@example.com",
    "display_name": "Vatsal P", "pledge_taken": true, "updated_at": "..."
  },
  "old_record": {
    "id": "7f7f0e0a-...", "email": "vatsal@example.com",
    "display_name": "Vatsal P", "pledge_taken": false, "updated_at": "..."
  }
}
```

- `type` ∈ `INSERT` / `UPDATE` / `DELETE`; `record` is null for DELETE, `old_record` null for INSERT.
- **Delivery guarantees: none.** pg_net is fire-and-forget — **at-most-once**, no retries on failure/timeout (timeout is a few seconds, configurable — [verify before build] current default). Design accordingly:
  - Handler = **idempotent upsert** keyed on the Supabase profile UUID (`record.id`), last-write-wins by `updated_at`.
  - Treat webhooks as a **cache-freshness hint, not the source of truth**: keep the pull path (a) able to re-read the profile at sensitive moments (e.g., right before deciding to skip P1–P3), or run a nightly reconciliation job for the identity_map rows.
- **Authenticate the endpoint**: configure a static header in the webhook (e.g. `X-Bridge-Secret: <random>`) and reject anything else; the payload itself is unsigned.
- Filter noise: create the trigger `AFTER UPDATE OF pledge_taken, display_name, avatar_url, email ON community_profile` so you're not hit on every unrelated column change.

## 5. Pricing & data residency

- **Free**: $0 — 500 MB database, 50k MAU, 5 GB egress, 1 GB storage, 500k edge function invocations; **projects pause after 1 week of inactivity** (unacceptable for a production bridge — the community project must be on Pro).
- **Pro**: **$25/month per project** — 8 GB disk (then $0.125/GB), 100k MAU (then $0.00325/MAU), 250 GB egress (then $0.09/GB), 100 GB storage, $10/mo compute credits, never paused.
- **Team**: $599/month (SOC 2 report access, org controls). Enterprise: custom.
- The *bridge itself* adds ~zero cost — a few thousand single-row REST reads/month is noise. The pricing question belongs to the YesLifers product, not Yesly.
- **Data residency**: Supabase lets you pick the project region at creation, and **Mumbai (`ap-south-1`) is on the current region list** (alongside Singapore, Tokyo, US, EU, etc.). If YesLifers is provisioned in `ap-south-1`, both systems keep personal data in India and bridge latency is single-digit ms AWS-internal. **Region cannot be changed in place later** (requires migration via backup/restore) — [verify before build] that the existing YesLifers project is actually in ap-south-1.

## 6. Compliance / DPDP

Two products of the **same company** sharing user data is still a *purpose change* under DPDP 2023 — consent given for "community participation" does not automatically cover "creating your investment-advisory account".

- **Consent notice wording**: at the warm-handoff screen in Yesly, before the pull: *"We'll use your YesLifers community profile (email, name, photo, pledge status) to set up your Yesly account so you don't repeat these steps."* One tap = recorded consent (store timestamp + notice version in `USER`). The community app's privacy notice should reciprocally disclose that profile data may be shared with Yesly for account creation.
- **Minimize propagated fields**: exactly `email, display_name, avatar_url, pledge_taken` + the stable profile UUID. Do not sync community activity, posts, or engagement scores into a SEBI-regulated advisory system "because we can".
- **Deletion propagation**:
  - *Community account deleted* → Yesly account **survives** (it's a standalone regulated relationship with its own consent), but: the DELETE webhook should null the `identity_map` link and stop any future sync; the copied avatar may be retained (it was propagated with consent) or deleted for goodwill — decide and document.
  - *Yesly account deleted* → no effect on the community account (one-way bridge), but Yesly must erase the propagated fields and the identity_map row per its own DPDP erasure duty. SEBI RIA record-keeping obligations may require retaining KYC/advice records beyond deletion — carve that out explicitly in the privacy notice.
  - Document both directions in the record of processing; "deletion in one system does not delete the other" must be stated to the user at consent time.

## 7. Gotchas & effort

- **service_role/secret key blast radius**: a leak = full read/write on the *entire* community database (RLS bypassed) — every DM, every post. Mitigations: AWS Secrets Manager (never env-in-repo, never client), the restricted **view + dedicated role** pattern from §2, prefer new `sb_secret_` keys (individually revocable), rotate on any suspicion, and alert on anomalous PostgREST volume from the bridge credentials.
- **Email as join key is fragile**: (1) *case sensitivity* — `Vatsal@X.com` vs `vatsal@x.com`; always lowercase+trim on both sides, query with `ilike` or a generated lowercased column; (2) *email changes desync the bridge* — user changes email in the community app and the next lookup misses. **Recommendation: use email only for the first-contact lookup, then persist a mapping table** `identity_map(yesly_user_id UUID, supabase_profile_id UUID, linked_at, last_synced_at)` and key all subsequent syncs/webhooks on the stable Supabase UUID (`record.id` / JWT `sub`), never on email.
- **Avatar: copy, don't hotlink.** Hotlinking the Supabase Storage URL couples Yesly's UI uptime to the community project, leaks Yesly traffic to it, breaks if the file is deleted or the bucket goes private (signed Storage URLs expire), and drains the community project's egress quota. On handoff: download once server-side, virus-scan/resize, store in Yesly's own S3, serve via Yesly's CDN.
- **Rate limits**: Supabase Auth endpoints are rate-limited; PostgREST/data API has no fixed rate limit but is bounded by the project's compute tier — the bridge's volume is trivial, but a runaway retry loop against a Micro instance can degrade the *community app*. Use capped exponential backoff and a circuit breaker (fall back to cold-funnel).
- **Free-tier pausing**: if anyone ever points the bridge at a Free-plan staging project, it will pause after a week of inactivity and the bridge 5xxs. Staging bridge target must also be Pro, or tests must tolerate wake-up.
- **Cognito quirk on the write side**: `AdminCreateUser` with `MessageAction="SUPPRESS"` avoids Cognito's invite email (Yesly owns comms); set `email_verified=true` only if the handoff carried a *verified* Supabase JWT, otherwise force OTP verification before trusting `pledge_taken`.
- **Effort**: option (a) pull + Cognito create + identity_map ≈ **2–3 dev-days** including error paths; webhook sync (b) ≈ +1–2 days incl. idempotency tests; JWT verification (d) ≈ +1 day. Direct Postgres (c): skip.

## Sandbox / testing

- Spin up a free Supabase project (or `supabase start` locally via Docker/CLI — full stack incl. PostgREST and webhooks; use `host.docker.internal` for webhook targets on the host) seeded with fake `community_profile` rows.
- Test matrix: profile found / not found / duplicate emails (force `406` via `vnd.pgrst.object+json`), pledge true/false, webhook UPDATE→pledge flip, webhook lost (reconciliation catches it), Supabase down (cold-funnel fallback), email case mismatch, deleted profile DELETE event.
- Contract-pin the response shape with a recorded fixture; the community team changing/renaming columns is the most likely silent breakage — agree on the `bridge_profile_v` view as the frozen contract.

## Recommendation

**Ship option (a)** — server-side REST pull with a scoped secret key against a dedicated `bridge_profile_v` view in a Mumbai-region project — plus an **`identity_map` UUID table from day one** (email only for first contact). Add **(b) database webhooks** for `pledge_taken` freshness in v1.1, treating them as at-most-once hints over idempotent upserts with a nightly reconcile. Use **(d) JWT verification via JWKS** only if/when the community app can mint deep-link tokens; skip **(c) direct Postgres** entirely. Never let any Supabase key near the RN app.

## Sources

- Supabase REST API (PostgREST): https://supabase.com/docs/guides/api
- API keys (publishable/secret vs anon/service_role): https://supabase.com/docs/guides/api/api-keys
- Row Level Security: https://supabase.com/docs/guides/database/postgres/row-level-security
- Database Webhooks: https://supabase.com/docs/guides/database/webhooks
- pg_net extension: https://supabase.com/docs/guides/database/extensions/pg_net
- JWTs & JWKS verification: https://supabase.com/docs/guides/auth/jwts
- Connecting / Supavisor pooler: https://supabase.com/docs/guides/database/connecting-to-postgres
- Available regions (incl. Mumbai ap-south-1): https://supabase.com/docs/guides/platform/regions
- Pricing: https://supabase.com/pricing
- Storage: https://supabase.com/docs/guides/storage
- Cognito AdminCreateUser: https://docs.aws.amazon.com/cognito-user-identity-pools/latest/APIReference/API_AdminCreateUser.html
