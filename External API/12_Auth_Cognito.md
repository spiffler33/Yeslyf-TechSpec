# 12 — Auth: AWS Cognito (User Pools)

> Researched July 2026 against official AWS docs. Facts marked **[verify]** should be re-checked in the console for ap-south-1.

## Overview & role in Yesly

Cognito user pools are Yesly's identity provider:

- **Google OAuth sign-in** (federated social IdP through the user pool).
- **Email OTP login** — Step 3's "just login, nothing more": user enters email, receives a code, is in. Since Nov 2024 this is a **native, managed Cognito feature** (passwordless email OTP in the Essentials tier) — the old DIY custom-auth-Lambda approach is now optional, not required.
- **Supabase → Cognito identity bridge**: the YesLifers community app (Supabase auth) provisions/updates a matching Cognito user (email, display-name, avatar, `pledge_taken`) via admin APIs so USER + AUTH_SESSION stay coherent across systems.
- Issues **JWTs** that FastAPI verifies statelessly via JWKS.

Region: **ap-south-1 (Mumbai)** — user pools, Essentials tier, managed login, and Lambda triggers are all available there.

## How it works

```
A) Email OTP (managed passwordless, Essentials tier)
[RN app] → InitiateAuth(AuthFlow=USER_AUTH, USERNAME=email,
                        PREFERRED_CHALLENGE=EMAIL_OTP)
        ← ChallengeName=EMAIL_OTP + Session        (Cognito emails code via SES)
[RN app] → RespondToAuthChallenge(EMAIL_OTP_CODE=123456, Session)
        ← ID token + Access token + Refresh token

B) Google sign-in (federated)
[RN app] --signInWithRedirect(provider:Google)--> Managed Login (hosted) 
   → accounts.google.com (OIDC) → callback yesly://auth
   → Cognito maps Google claims via attribute mapping → user in pool
   → tokens returned via authorization-code + PKCE

C) Supabase bridge (server-to-server)
[Supabase edge fn / FastAPI] → AdminCreateUser(email, MessageAction=SUPPRESS,
      attrs: email_verified=true, name, picture, custom:pledge_taken)
   → AdminUpdateUserAttributes on profile change
   → app rows written to USER + AUTH_SESSION keyed by cognito sub

D) API calls
[RN app] --Authorization: Bearer <access JWT>--> [FastAPI]
   FastAPI verifies sig against JWKS
   https://cognito-idp.ap-south-1.amazonaws.com/<poolId>/.well-known/jwks.json
   (cache keys; check iss, aud/client_id, token_use, exp)
```

## Key API calls

| Method | API / SDK call | Purpose | Key request fields | Key response fields |
|---|---|---|---|---|
| POST | `InitiateAuth` | Start sign-in | `AuthFlow`: `USER_AUTH` (choice-based, Essentials+) \| `USER_SRP_AUTH` \| `CUSTOM_AUTH`; `AuthParameters:{USERNAME, PREFERRED_CHALLENGE}` | `ChallengeName` (`EMAIL_OTP`…), `Session`, `AvailableChallenges`, or `AuthenticationResult` |
| POST | `RespondToAuthChallenge` | Answer OTP/SRP/custom challenge | `ChallengeName`, `Session`, `ChallengeResponses:{USERNAME, EMAIL_OTP_CODE}` | `AuthenticationResult:{IdToken, AccessToken, RefreshToken, ExpiresIn}` |
| POST | `InitiateAuth` (`REFRESH_TOKEN_AUTH`) | Refresh tokens | `REFRESH_TOKEN` | new Id/Access tokens |
| POST | `RevokeToken` | Revoke a refresh token (logout-everywhere) | `Token`, `ClientId` | — (derived access tokens become rejected by Cognito APIs; your own API must rely on short expiry) |
| POST | `SignUp` / `ConfirmSignUp` | Self-service registration (if used beyond OTP-auto-create) | `Username`, `UserAttributes` | `UserSub` |
| POST | `AdminCreateUser` | **Supabase bridge**: provision user without invite email | `Username=email`, `MessageAction:"SUPPRESS"`, `UserAttributes:[{email_verified:true},{name},{picture},{custom:pledge_taken}]` | `User{Username(sub), Attributes}` |
| POST | `AdminUpdateUserAttributes` / `AdminGetUser` | Bridge updates / lookups | `UserPoolId`, `Username`, attrs | attributes |
| POST | `AdminLinkProviderForUser` | Link Google identity onto an existing native user (avoid duplicate accounts) | `DestinationUser` (native), `SourceUser` (Google `Cognito_Subject`) | — |
| GET | `https://cognito-idp.ap-south-1.amazonaws.com/{poolId}/.well-known/jwks.json` | JWKS for FastAPI verification | — | `keys[] {kid, n, e}` |
| GET/POST | Managed Login endpoints: `/oauth2/authorize`, `/oauth2/token`, `/oauth2/userInfo`, `/logout` | OIDC flow for Google federation | `client_id`, `redirect_uri`, `code_challenge` (PKCE), `identity_provider=Google` | `code` → `id_token, access_token, refresh_token` |

### Example — email OTP round trip

```json
// InitiateAuth request
{ "AuthFlow": "USER_AUTH", "ClientId": "abc123",
  "AuthParameters": { "USERNAME": "user@x.com", "PREFERRED_CHALLENGE": "EMAIL_OTP" } }
// → response
{ "ChallengeName": "EMAIL_OTP", "Session": "AYABe...",
  "ChallengeParameters": { "CODE_DELIVERY_DESTINATION": "u***@x***" } }
// RespondToAuthChallenge request
{ "ChallengeName": "EMAIL_OTP", "ClientId": "abc123", "Session": "AYABe...",
  "ChallengeResponses": { "USERNAME": "user@x.com", "EMAIL_OTP_CODE": "123456" } }
// → response
{ "AuthenticationResult": { "IdToken": "eyJ...", "AccessToken": "eyJ...",
    "RefreshToken": "eyJ...", "ExpiresIn": 3600, "TokenType": "Bearer" } }
```

### Email OTP: managed vs custom-auth-Lambda

- **Managed (recommended, GA since Nov 2024)**: enable **choice-based sign-in** (`ALLOW_USER_AUTH`) + `EMAIL_OTP` in `SignInPolicy.AllowedFirstAuthFactors`. Requires **Essentials or Plus** tier and SES for email. Constraint: EMAIL_OTP as a *first factor* can't coexist with *required* MFA. Zero Lambda code.
- **Custom auth challenge (legacy/DIY)**: `CUSTOM_AUTH` flow with three Lambda triggers — `DefineAuthChallenge` (state machine: issue CUSTOM_CHALLENGE, allow N retries), `CreateAuthChallenge` (generate 6-digit code, send via SES, stash in `privateChallengeParameters`), `VerifyAuthChallengeResponse` (compare). Works on any tier; full control over email template/expiry/attempts; more code to own. Use only if the managed feature's fixed UX (code length/expiry) doesn't fit.

### Amplify / React Native client

- **Amplify v6** (`aws-amplify` + `@aws-amplify/react-native`): `signIn({ username, options: { authFlowType: 'USER_AUTH', preferredChallenge: 'EMAIL_OTP' }})` → `confirmSignIn({ challengeResponse: code })`. Handles SRP, token storage (Keychain/Keystore), refresh.
- **Google**: `signInWithRedirect({ provider: 'Google' })` opens Managed Login → Google via an auth session browser; needs a deep-link scheme (`yesly://`) registered in the app client callback URLs. In Expo, works in dev builds (not Expo Go) **[verify current Expo Go support]**.
- **Hosted UI (Managed Login) vs SDK**: Managed Login (Essentials) is required in the loop for federation; for email OTP you can stay fully native with Amplify APIs — matching the "just login, nothing more" minimal Step-3 UX. Managed Login branding is configurable (new branding designer) but layout is fixed — don't plan pixel-perfect custom hosted pages.

## Auth (how Yesly's servers call Cognito)

- Client calls (`InitiateAuth`, `SignUp`) are **unsigned** public API calls with `ClientId` (use a client **without** a secret for mobile).
- `Admin*` calls (the Supabase bridge) are **SigV4-signed** with IAM credentials — run from FastAPI or a Lambda with a least-privilege role (`cognito-idp:AdminCreateUser`, `AdminUpdateUserAttributes`, `AdminGetUser`, `AdminLinkProviderForUser` on the one pool ARN). Never ship admin credentials near Supabase client code; bridge should be Supabase → (webhook/queue) → your backend → Cognito.

## Webhooks (Lambda triggers)

Cognito's "webhooks" are **Lambda triggers**: `PreSignUp`, `PostConfirmation`, `PreAuthentication`, `PostAuthentication`, `PreTokenGeneration` (add custom claims like `pledge_taken` into the JWT), `UserMigration`, the three custom-auth triggers, `CustomEmailSender`. 
- **`PreSignUp` auto-link gotcha**: when a user who already exists (email) signs in with Google for the first time, Cognito would create a *second* user (`google_1234...`). The standard fix is a PreSignUp trigger that calls `AdminLinkProviderForUser` to link the incoming Google identity to the existing native user. Caveats: link **before** the federated user is created (the trigger flow does this, but errors leave orphaned `google_*` users); linking is effectively permanent; the first federated sign-in after linking can throw an error requiring a retry **[verify current behavior]**; attribute mapping may overwrite `name`/`picture` on each Google login — decide precedence with the Supabase bridge.
- `UserMigration` trigger is the alternative bridge pattern (lazy migration on first login) — for Yesly, proactive `AdminCreateUser` from the Supabase side is simpler and keeps `pledge_taken` fresh.

## Sandbox / testing

- No sandbox mode — create separate **dev/staging/prod user pools** (infrastructure-as-code: Terraform/CDK) since many pool settings are immutable after creation (e.g., username attributes, required attributes).
- SES starts in sandbox (verified recipients only) — request production access early for OTP emails; monitor SES bounce/complaint rates.
- Local testing: `cognito-local` / moto emulators are approximations only — do auth-flow integration tests against the dev pool.
- Google: separate OAuth client IDs per environment; test the PKCE + deep-link flow on both platforms.

## Pricing (July 2026)

Tier structure introduced **Nov 22, 2024** (Lite / Essentials / Plus), per-MAU, per pool:

- **Lite**: legacy feature set (incl. classic hosted UI, custom-auth Lambdas). Free **10K MAU** (accounts created before 2024-11-22 keep the old **50K free** on Lite), then ~$0.0055/MAU declining with volume (to ~$0.0046 and lower at large scale).
- **Essentials** (default; what Yesly needs for managed email OTP + Managed Login): free **10K MAU**, then **$0.015/MAU**.
- **Plus** (threat protection/advanced security): **$0.02/MAU, no free tier**.
- **SAML/OIDC enterprise federation**: $0.015/MAU beyond 50 free — **does not apply to Google social login**, which bills as normal MAU. 
- **M2M**: $0.00225 per token request (relevant if service-to-service auth uses Cognito app clients).
- SES email (~$0.10/1k emails) and any SMS via SNS are extra.
- Ballpark for Yesly: **free up to 10K MAU; 50K MAU ≈ $600/mo on Essentials** (40K × $0.015). MAU counts only users with identity operations that month.

## Compliance & data residency (DPDP)

- User pool data (emails, names, attributes) is stored **in-region**: ap-south-1 (Mumbai) — good DPDP posture; no cross-border storage for the identity store.
- JWTs: don't stuff sensitive personal data into custom claims (they're readable by the client and logged by proxies).
- DPDP consent: signup flow should capture consent notice; Cognito doesn't manage consent records — keep that in your USER table.
- SEBI/RIA angle: auth logs (CloudTrail for admin APIs, Plus-tier threat monitoring if adopted) support audit-trail expectations; retain per your compliance policy.

## Gotchas & effort

1. **Fixed UX limits**: managed email OTP code format/expiry and Managed Login page layout are not fully customizable. If product demands a bespoke OTP email or in-house UI text, fall back to the custom-auth Lambda flow (own the SES template).
2. **Federated-vs-native account linking** is the classic Cognito pain (see PreSignUp above). Decide day one: email is the join key; implement the linking trigger before launching Google sign-in, or you'll ship duplicate accounts that are miserable to merge later.
3. **Immutable pool settings** (sign-in attribute choices, custom attribute schema — attributes can be added but not removed/changed) → IaC + a rehearsal migration plan.
4. **Refresh-token revocation**: `RevokeToken` invalidates the refresh token and its derived tokens for Cognito's own endpoints, but **your FastAPI accepts JWTs until `exp`** — keep access-token expiry short (15–60 min) and, if you need instant kill, maintain a small deny-list keyed by `origin_jti`.
5. **Throttling**: user-pool APIs have category-based account/region quotas (e.g., `UserAuthentication` ~120 RPS, `UserCreation` ~50 RPS by default — adjustable via Service Quotas) **[verify current numbers]**. OTP-heavy login storms (marketing push at 9am) can hit these; add client backoff and request quota raises ahead of campaigns.
6. **Custom attributes** (`custom:pledge_taken`) are strings/numbers only, max lengths apply, and are not searchable via ListUsers filters — mirror anything queryable into your own USER table (which Yesly already plans).
7. **Bridge idempotency**: `AdminCreateUser` on an existing email throws `UsernameExistsException` — bridge must upsert (catch → `AdminUpdateUserAttributes`).

**Effort ballpark**: managed email OTP + Amplify RN ≈ 3–4 dev-days; Google federation + deep links + PreSignUp linking ≈ 3–5 days (linking edge cases dominate); FastAPI JWT middleware ≈ 1 day; Supabase bridge ≈ 2–3 days incl. retries/idempotency.

## Sources

- https://docs.aws.amazon.com/cognito/latest/developerguide/amazon-cognito-user-pools-authentication-flow-methods.html (USER_AUTH, EMAIL_OTP, AllowedFirstAuthFactors)
- https://aws.amazon.com/blogs/aws/improve-your-app-authentication-workflow-with-new-amazon-cognito-features/ (Nov 2024: Essentials/Plus, Managed Login, passwordless GA)
- https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-sign-in-feature-plans.html and .../feature-plans-features-essentials.html
- https://aws.amazon.com/cognito/pricing/ (Lite/Essentials/Plus rates, 10K free, M2M, federation)
- https://docs.aws.amazon.com/cognito-user-identity-pools/latest/APIReference/API_InitiateAuth.html ; API_AdminCreateUser.html ; API_AdminLinkProviderForUser.html ; API_RevokeToken.html
- https://docs.aws.amazon.com/cognito/latest/developerguide/user-pool-lambda-pre-sign-up.html (auto-link pattern) **[re-verify linking edge cases]**
- https://docs.aws.amazon.com/cognito/latest/developerguide/amazon-cognito-user-pools-using-tokens-verifying-a-jwt.html (JWKS verification)
- https://docs.aws.amazon.com/cognito/latest/developerguide/limits.html (quotas/throttling)
- https://docs.amplify.aws/react-native/build-a-backend/auth/ (Amplify v6 RN)
