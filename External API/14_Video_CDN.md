# 14 — Video CDN (Founder Videos)

## Overview & role in Yesly

Yesly plays short founder-recorded videos at roughly five points in the onboarding/upgrade flow:

| Flow step | Video | Skippable? |
|---|---|---|
| Pre-login splash | Welcome video from founder (plays before any auth) | Yes |
| SEBI registration note | Short "we are a SEBI-registered RIA" explainer | Yes |
| Before essential-plan pitch | Mandatory founder video — **cannot skip** | No |
| Free → paid transition | Upgrade rationale video | Yes |
| (Reserve) misc. flow steps | Future founder notes | Yes |

Characteristics that drive the decision:

- **Modest, fixed catalog**: tens of videos, uploaded by the team, not UGC. No live streaming.
- **Playback surface**: React Native / Expo app (`expo-video` or `react-native-video`), iOS + Android.
- **Analytics**: watched-vs-skipped (and watch-percentage) events must land in **Mixpanel** — the product decision ("did warm users watch the mandatory video?") lives there, not in a video-vendor dashboard.
- **Latency**: users are in India; startup latency (time-to-first-frame) matters because these videos gate flow progression.
- **Stack**: already on AWS Mumbai (ap-south-1), FastAPI backend, S3 in use for other assets.
- **No DRM needed** — these are marketing/onboarding videos, not paid content. Signed URLs to prevent hotlinking are sufficient; Widevine/FairPlay would be pure cost and complexity. Explicitly: **do not buy DRM for this.**

## How it works (generic flow, any vendor)

```
[Team uploads founder MP4]
        |
        v
[Encoding step] --- S3+MediaConvert / Mux asset / Stream video
        |            (produces HLS renditions: 360p/720p/1080p)
        v
[Webhook / job-complete] --> FastAPI stores {video_key -> playback_url/playback_id}
                                 in a `videos` table (RDS)
        |
        v
[App requests flow step] --> FastAPI returns playback URL
        |                     (+ short-lived signed token if private)
        v
[expo-video / react-native-video plays HLS]
        |
        +--> onLoad        -> mixpanel.track("video_started", {video_key, step})
        +--> onProgress    -> track 25/50/75% quartiles
        +--> onEnd         -> mixpanel.track("video_completed", ...)
        +--> user skips    -> mixpanel.track("video_skipped", {at_seconds, pct})
        |
        v
["cannot skip" video: UI simply renders no skip/seek controls
 and gates the Continue button on the onEnd event]
```

Key insight: **"cannot skip" and watched-vs-skipped are 100% client-side concerns** — they are implemented with player events and UI state in the RN app, identically for every CDN below. The CDN choice only affects encoding, delivery, latency, and whether you get QoE analytics for free.

---

## Option 1 — AWS-native: S3 + CloudFront (+ optional MediaConvert)

The "already on AWS Mumbai" default. Two sub-variants:

**1a. Progressive MP4 (simplest):** upload a well-encoded MP4 (H.264, ~2–4 Mbps, `moov` atom at front / "fast start") to S3, serve via CloudFront. Both `expo-video` and `react-native-video` play progressive MP4 fine. No adaptive bitrate — one rendition for everyone; a user on weak 4G either buffers or you ship a single conservative ~1.5–2.5 Mbps 720p file.

**1b. HLS ABR via MediaConvert:** run each MP4 through AWS Elemental MediaConvert (job template producing an HLS ladder, e.g. 360p/540p/720p/1080p), output to S3, serve the `.m3u8` + segments via CloudFront. This is what Mux/Stream do for you automatically; on AWS you own the job templates, the completion EventBridge events, and the packaging details.

### Endpoints / operations (AWS)

| Operation | Call | Input | Output |
|---|---|---|---|
| Upload source | `s3.put_object` / presigned PUT | Bucket, key, MP4 bytes | 200; object at `s3://yesly-videos-src/welcome.mp4` |
| (1b) Transcode | `mediaconvert.create_job` | Job template ref, input S3 URI, output group (Apple HLS), ladder settings | Job ID; async |
| (1b) Job done | EventBridge rule `MediaConvert Job State Change` → Lambda/FastAPI | `{"detail": {"status": "COMPLETE", "jobId": ...}}` | Store `https://cdn.yesly.in/videos/welcome/index.m3u8` |
| Serve | CloudFront distribution, origin = output bucket (OAC) | GET `.m3u8`/`.ts`/`.mp4` | Cached segments from Indian edge |
| Restrict access | CloudFront **signed URLs** (single file / MP4) or **signed cookies** (HLS — many segment URLs) | CloudFront key pair, policy `{Resource, DateLessThan}` | `?Expires=&Signature=&Key-Pair-Id=` or `CloudFront-Policy/-Signature/-Key-Pair-Id` cookies |

Notes:
- For HLS behind auth you need **signed cookies** (a signed URL only covers one URL; HLS fetches dozens of segment files) — RN cookie handling with the native players is fiddly (see Gotchas). For founder marketing videos, a public-read CloudFront path with an unguessable key prefix is a defensible v1 simplification.
- No built-in video analytics — you rely entirely on client player events → Mixpanel (which you need anyway).

### Effort assessment

- Progressive MP4: **~1 dev-day** (bucket, distribution, OAC, upload script, table of URLs).
- HLS ABR + MediaConvert pipeline + signed cookies: **~1–2 dev-weeks** including job templates, event handling, player testing on Android — this is the hidden cost of "just use AWS".

### Pricing (CloudFront, India)

- Data transfer out to internet, India price class: **~$0.109/GB for the first 10 TB/month** (India is one of CloudFront's most expensive regions; US is ~$0.085/GB). [verify before build — confirm on the CloudFront pricing page calculator for ap-south-1]
- HTTPS requests in India: ~$0.012 per 10,000. [verify before build]
- **CloudFront always-free tier: 1 TB data transfer out + 10M requests per month, every month.** At Yesly's scale (see worked example below) this likely makes CloudFront delivery **effectively $0** for a long time.
- MediaConvert: per-minute of output per rendition (AVC, ≤30fps SD/HD tiers); for tens of short videos this is single-digit dollars total, one-time. [verify before build — MediaConvert pricing page for ap-south-1]
- S3 storage: negligible (a few GB).

---

## Option 2 — Mux Video (top challenger)

API-first video platform: you upload, Mux encodes to HLS ABR ("just-in-time" renditions), gives you a `playback_id`, serves via multi-CDN, and **Mux Data QoE analytics is included free** (startup time, rebuffering, watch time, exit-before-start — per view).

### Endpoint I/O table (Mux)

Base URL `https://api.mux.com`, auth = HTTP Basic with Access Token ID/Secret (server-side only).

| Purpose | Endpoint | Input (key fields) | Output (key fields) |
|---|---|---|---|
| Create direct upload | `POST /video/v1/uploads` | `{"cors_origin":"*", "new_asset_settings": {"playback_policy":["public"], "video_quality":"basic"}}` | `{"data":{"id":"upl_...","url":"https://storage.googleapis.com/...","status":"waiting"}}` |
| Upload the file | `PUT <upload.url>` | Raw MP4 bytes (resumable) | 200 |
| Get asset | `GET /video/v1/assets/{ASSET_ID}` | — | `{"data":{"id":"asset_...","status":"ready","duration":94.3,"playback_ids":[{"id":"PLAYBACK_ID","policy":"public"}]}}` |
| Create asset from URL (alt.) | `POST /video/v1/assets` | `{"inputs":[{"url":"https://s3.../welcome.mp4"}],"playback_policy":["public"]}` | asset object as above |
| Playback | `GET https://stream.mux.com/{PLAYBACK_ID}.m3u8` | (+ `?token=JWT` if signed policy) | HLS manifest |
| Thumbnail/poster | `GET https://image.mux.com/{PLAYBACK_ID}/thumbnail.jpg?time=0` | — | JPEG (use as `poster` to mask cold start) |
| Create signing key | `POST /system/v1/signing-keys` | — | `{"data":{"id":"KEY_ID","private_key":"<base64 RSA PEM>"}}` |
| Signed playback token | (build in FastAPI, no API call) | JWT RS256: `{"sub":"PLAYBACK_ID","aud":"v","exp":<unix>,"kid":"KEY_ID"}` | `?token=<jwt>` on the `.m3u8` URL |

Example `video.asset.ready` webhook (POST to your FastAPI endpoint, `mux-signature` header, retried with backoff for 24h on non-2xx — deduplicate by event `id`):

```json
{
  "type": "video.asset.ready",
  "id": "evt_...",
  "object": { "type": "asset", "id": "asset_abc123" },
  "environment": { "name": "production", "id": "..." },
  "created_at": "2026-07-02T10:00:00.000000Z",
  "data": {
    "id": "asset_abc123",
    "status": "ready",
    "duration": 94.3,
    "max_stored_resolution": "HD",
    "playback_ids": [{ "id": "pb_xyz", "policy": "public" }],
    "passthrough": "video_key=founder_mandatory_v2"
  }
}
```

Use `passthrough` to carry your internal `video_key` so the webhook handler can upsert `videos(video_key, playback_id, duration, status)` idempotently.

### React Native support

- Playback: `stream.mux.com/....m3u8` in `expo-video` or `react-native-video` — plain HLS, works with AVPlayer/ExoPlayer.
- Mux Data SDK: official wrapper for **react-native-video** (`@mux/mux-data-react-native-video`) gives QoE beacons automatically. There is **no official Mux Data SDK for `expo-video`** — if you standardize on `expo-video`, you still get Mux Data's CDN-side request logs but not full player QoE, and your Mixpanel events remain the product source of truth. [verify before build — check current Mux Data SDK list for an expo-video integration]

### Pricing (Mux, current public rates)

Plans: Free / Pay-as-you-go ($20 monthly usage credit) / Pre-pay (Launch $20/mo → $100 credit; Scale $500/mo → $1,000 credit).

- **Encoding**: "Basic" quality = **free** (all resolutions). "Plus" quality ~$0.025–0.031/min (720p–1080p, first tier); "Premium" higher. Basic is fine for talking-head founder videos.
- **Storage**: Basic/Plus ~**$0.003/min/month at 1080p** ($0.0024 at ≤720p). Free plan: up to 10 videos stored.
- **Delivery**: **first 100,000 minutes delivered each month are free**, then ~$0.001/min at 1080p ($0.0008 ≤720p).
- **Mux Data**: included at no additional charge for Mux Video customers.

[verify before build — mux.com/pricing; the model changed to quality-tiers (Basic/Plus/Premium) and rates are tiered by resolution and volume]

At Yesly's scale (say 10,000 views/month × 2-min videos = 20,000 delivered minutes; ~60 stored minutes): delivery is inside the free 100k minutes, storage ≈ $0.20/mo, Basic encoding free → **effectively $0–$20/mo** (Pay-as-you-go credit likely absorbs everything).

---

## Option 3 — Cloudflare Stream

Flat, duration-based pricing; encoding free; serves HLS/DASH from Cloudflare's edge (extensive India PoPs, though Indian ISP peering can occasionally route via Singapore — test with Jio/Airtel).

### Endpoint I/O table (Cloudflare Stream)

Base `https://api.cloudflare.com/client/v4/accounts/{ACCOUNT_ID}`, header `Authorization: Bearer <API_TOKEN>`.

| Purpose | Endpoint | Input | Output |
|---|---|---|---|
| Upload ≤200MB (simple) | `POST /stream` (multipart) | `file=@welcome.mp4` | `{"result":{"uid":"abc123...","readyToStream":false,"playback":{"hls":"https://customer-<CODE>.cloudflarestream.com/<UID>/manifest/video.m3u8"}}}` |
| Upload large (tus) | `POST /stream` (tus protocol) | tus headers + chunks | video `uid` |
| Copy from URL | `POST /stream/copy` | `{"url":"https://s3.../welcome.mp4","meta":{"name":"welcome"}}` | video object |
| Get video / poll ready | `GET /stream/{UID}` | — | `{"result":{"uid":"...","readyToStream":true,"duration":94.3,"thumbnail":"..."}}` |
| Require signed URLs | `POST /stream/{UID}` | `{"requireSignedURLs":true}` | updated video |
| Short-lived token (low volume) | `POST /stream/{UID}/token` | optional `{"exp":...,"downloadable":false,"accessRules":[...]}` | `{"result":{"token":"<JWT>"}}` (default 1h; endpoint intended for <1,000 tokens/day — self-sign with a key from `POST /stream/keys` beyond that) |
| Playback | `GET https://customer-<CODE>.cloudflarestream.com/<UID or TOKEN>/manifest/video.m3u8` | token replaces UID in the path when signed | HLS manifest |
| Webhook config | `PUT /stream/webhook` | `{"notificationUrl":"https://api.yesly.in/webhooks/stream"}` | secret for `Webhook-Signature` verification |

Webhook fires when a video finishes processing (`readyToStream: true`) or errors; payload is the video object. [verify before build — exact payload shape and signature scheme in the Stream webhooks doc]

### React Native caveats

- **No official RN SDK.** The `<Stream>` React component and iframe player are web-only. In RN you play the raw HLS manifest URL in `expo-video`/`react-native-video` — that works, but you get none of Cloudflare's player analytics, and Stream's built-in analytics is delivery-minutes oriented, not watched-vs-skipped. Your Mixpanel instrumentation does all the analytics work.
- Signed playback: token goes **in the URL path**, which is simpler than CloudFront cookies and equivalent to Mux's query token.

### Pricing (Cloudflare Stream)

- **Storage: $5 per 1,000 minutes stored (prepaid blocks; duration-based, file size irrelevant).**
- **Delivery: $1 per 1,000 minutes delivered** (includes bandwidth; no egress fees; encoding and ingress free).
- Yesly worked example: 60 stored minutes → $5 minimum block; 20,000 delivered min/month → $20/mo. **≈ $25/mo.** No free tier for Stream itself (it's a paid add-on; some Pro/Business bundles include small allowances — [verify before build]).

---

## Option 4 — Unlisted YouTube / Vimeo embeds (rejected)

Tempting because it's free and the founder already has a YouTube channel. Wrong for a fintech onboarding flow:

- **Branding & chrome**: YouTube iframe brings logo, "Watch on YouTube", related-video end screens — inside a SEBI-registered advisory app's mandatory compliance video, this is unacceptable UX and looks unprofessional.
- **Ads & recommendations risk**: you don't control what YouTube overlays or recommends; "unlisted" is discoverable by link and can be scraped/re-shared.
- **No skip control**: you cannot reliably suppress seeking inside the YouTube iframe player, so the "cannot skip" requirement is unimplementable; player event granularity (for watched-vs-skipped) is coarse and web-view based in RN (no native player).
- **Tracking/DPDP**: embedding YouTube injects Google tracking into a regulated fintech flow with no DPA in your favor — a needless third-party data disclosure to explain in your DPDP notice.
- **ToS**: YouTube's ToS restricts embedding in apps in ways that interfere with player functionality (e.g., hiding controls, forcing completion) — exactly what Yesly needs to do. Vimeo Pro allows cleaner embeds but at that point you're paying anyway; pay Mux/Cloudflare for a real API instead.

**Verdict: do not use, even for v1.**

---

## "Cannot skip" + watched-vs-skipped (client-side, CDN-agnostic)

```tsx
// expo-video sketch — identical logic for any CDN
const player = useVideoPlayer(playbackUrl, p => { p.play(); });

// 1. No controls on the mandatory video: nativeControls={false},
//    render no seek bar, no skip button. Continue button disabled.
<VideoView player={player} nativeControls={false} />

// 2. Events -> Mixpanel
useEffect(() => {
  mixpanel.track("video_started", { video_key, flow_step, mandatory });
  const sub = player.addListener("statusChange", ...);
  // quartile tracking via timeUpdate events:
  //   fire video_progress at 25/50/75% once each
  // completion:
  //   playToEnd -> mixpanel.track("video_completed", { watch_pct: 100 })
  //   -> setCanContinue(true)        // gates the flow
}, []);

// 3. Skippable videos: Skip button ->
//    mixpanel.track("video_skipped", { at_seconds: player.currentTime,
//      watch_pct: player.currentTime / player.duration })
// 4. App background/kill during mandatory video: persist "not completed";
//    on return, restart or resume — completion flag only set on playToEnd.
```

Rules of thumb: derive `watch_pct` from player time, not wall clock; debounce progress events (quartiles only — don't spam Mixpanel per second); treat `video_completed` as the only unlock signal for the mandatory step and mirror it to the backend (`POST /flow/video-completed`) so a reinstall doesn't re-gate a paying user unfairly (or does — product decision).

## Auth

- **CloudFront**: signed URLs (MP4) / signed cookies (HLS) with a CloudFront key pair; or public bucket path for marketing videos.
- **Mux**: `playback_policy: "signed"` → FastAPI mints RS256 JWT (`sub`=playback_id, `aud`="v", `exp`, `kid`) with a key from `POST /system/v1/signing-keys`; token appended as `?token=`. Public policy is fine for these videos.
- **Cloudflare Stream**: `requireSignedURLs: true` → token from `/token` endpoint or self-signed JWT (claims: `exp` ≤24h, `nbf`, `accessRules` e.g. `ip.geoip.country`), token replaces the UID in the playback URL.
- All vendor API keys/secrets live **server-side in FastAPI only** (Secrets Manager); the app only ever receives playback URLs/tokens.

## Webhooks

| Vendor | Event | Handler action |
|---|---|---|
| Mux | `video.asset.ready` / `video.asset.errored` / `video.upload.asset_created` | Upsert `videos` row by `passthrough` key; verify `mux-signature`; idempotent (Mux retries 24h, duplicates possible) |
| Cloudflare | video ready notification to `notificationUrl` | Same upsert; verify `Webhook-Signature` |
| AWS | EventBridge `MediaConvert Job State Change` → Lambda → FastAPI | Same upsert; no signature needed (in-account) |

For a catalog of tens of fixed videos, webhooks are a convenience, not a necessity — a manual "poll until ready then paste playback ID into admin" works for v1.

## Sandbox / testing

- **Mux**: free plan (100k delivery min/mo, 10 stored videos) + separate sandbox/production environments per account — ideal for dev/staging keys.
- **Cloudflare Stream**: no free tier; a $5 storage block on a dev account is the minimum test cost.
- **AWS**: normal dev account; CloudFront free tier covers testing.
- Test matrix that matters: Android low-end device on throttled 3G/4G (network conditioner), iOS on Wi-Fi, app-background mid-video, seek-attempt on mandatory video, airplane-mode failure UX (mandatory video must have a retry state, not a dead end).

## Pricing summary (Yesly worked example: ~60 stored min, 20k delivered min/mo, India)

| Option | Setup cost | Monthly | Notes |
|---|---|---|---|
| S3+CloudFront (MP4) | ~1 day dev | **~$0** (inside 1 TB free tier) | No ABR, no vendor analytics |
| S3+CloudFront+MediaConvert (HLS) | ~1–2 weeks dev | ~$0 + one-time encode $ | You own the pipeline |
| Mux (Basic quality) | ~2–3 days dev | **~$0–20** (free delivery ≤100k min; PAYG $20 credit) | Mux Data QoE free |
| Cloudflare Stream | ~2–3 days dev | **~$25** ($5 storage block + $20 delivery) | Flat, predictable |
| YouTube unlisted | 0 | $0 | Rejected (ToS, branding, no skip control) |

All rates: [verify before build] against current official pricing pages before contracting.

## Compliance / DPDP notes

- Viewer analytics (watch events tied to a user ID) are **personal data** under DPDP 2023 — cover "product analytics including video engagement" in the consent notice you already need for Mixpanel; no new category if events go only to Mixpanel.
- If Mux Data is enabled, it collects viewer session data (IP, device) on Mux's US infrastructure — add Mux as a processor in the privacy notice, or disable Mux Data beacons and rely on Mixpanel only. CloudFront keeps logs in your AWS account (simplest story).
- No sensitive personal data is in the videos themselves; the risk surface is small. Keep video URLs free of PII (no email/user_id in query strings that end up in CDN logs).
- Data residency: video files on CloudFront ap-south-1 origin stay in India at rest; Mux/Cloudflare store masters on their infrastructure (US/global) — fine for marketing content, just say so in the record of processing.

## Recommendation

- **v1 (few, fixed videos): S3 + CloudFront with well-encoded progressive MP4s.** Zero new vendors, ~$0/month inside the free tier, one dev-day, data stays on AWS Mumbai. Encode conservatively (720p, ~2 Mbps, faststart), ship a poster image, and lean on Mixpanel events for all analytics. Skip MediaConvert/HLS until buffering complaints are real.
- **If the founder wants engagement/QoE analytics without building anything, or the catalog starts churning weekly: Mux.** Free-to-cheap at this scale, Basic encoding free, 100k free delivery minutes, real API + webhooks, Mux Data included. It removes all video-ops.
- Cloudflare Stream is a fine middle option ($25/mo flat) but offers no RN SDK and weaker per-view analytics than Mux — it wins only if Yesly already adopts Cloudflare elsewhere.
- YouTube/Vimeo embeds: rejected outright (see Option 4).

## Gotchas & effort

- **Android ExoPlayer HLS quirks**: ExoPlayer (via `expo-video`/`react-native-video`) is stricter than AVPlayer — mismatched segment durations, non-keyframe-aligned renditions, or missing `EXT-X-VERSION` cause stalls/black frames that iOS silently tolerates. If you hand-roll MediaConvert HLS, test Android first. Mux/Stream output is already player-safe.
- **Cold-start latency**: first play of a rarely-watched video hits the origin (CloudFront miss → S3 Mumbai is fast; Mux/Stream first-hit may be slower from Indian PoPs). Mitigate: show poster instantly, preload the *next* flow video (`player.preload`/prefetch the manifest) while the user reads the previous screen — this matters more than CDN choice for perceived latency.
- **CloudFront signed cookies + RN**: native players don't automatically attach cookies set via JS fetch; you must pass headers/cookies through the player's `source.headers` — test on both platforms, it's the classic failure point of the AWS route. Public-path v1 avoids it entirely.
- **Captions**: founder videos need burned-in or sidecar captions (many users watch muted; also an accessibility win). HLS sidecar (`WebVTT`) is supported by Mux (`text_tracks` on asset) and MediaConvert; for the MP4-only v1, burned-in captions are the pragmatic choice.
- **Mux Data + expo-video gap**: no official Mux Data SDK for expo-video (only react-native-video) — decide the player library before the vendor. [verify before build]
- **Don't gate the app on video load**: mandatory video must have a timeout/fallback (e.g., after 2 failed loads show text version + continue) or a CDN blip blocks 100% of conversions at that step.
- Effort: CloudFront MP4 ~1 day; Mux integration (upload script + webhook + signed tokens) ~2–3 days; MediaConvert HLS pipeline ~1–2 weeks.

## Sources

- CloudFront pricing: https://aws.amazon.com/cloudfront/pricing/ (India ~$0.109/GB first 10 TB; 1 TB/mo always-free tier)
- CloudFront private content (signed URLs/cookies): https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/PrivateContent.html
- AWS MediaConvert: https://aws.amazon.com/mediaconvert/pricing/
- Mux pricing: https://www.mux.com/pricing and https://www.mux.com/docs/pricing.txt
- Mux direct uploads: https://www.mux.com/docs/guides/upload-files-directly
- Mux signed playback: https://www.mux.com/docs/guides/secure-video-playback
- Mux webhooks: https://www.mux.com/docs/core/listen-for-webhooks
- Cloudflare Stream pricing: https://developers.cloudflare.com/stream/pricing/
- Cloudflare Stream signed URLs: https://developers.cloudflare.com/stream/viewing-videos/securing-your-stream/
- Cloudflare Stream API: https://developers.cloudflare.com/api/resources/stream/
- expo-video: https://docs.expo.dev/versions/latest/sdk/video/
- react-native-video: https://docs.thewidlarzgroup.com/react-native-video/
