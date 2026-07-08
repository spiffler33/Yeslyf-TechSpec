# 20 — Microsoft Sign-In: Entra ID (OIDC) federated through Cognito

> Researched July 2026 against Microsoft Learn + AWS docs. Facts marked **[verify]** should be re-checked before launch.

## Overview & role in Yesly

Adds a "Continue with Microsoft" button next to Google/Apple on the Step 2 login screen. Microsoft Entra ID (formerly Azure AD, part of the Microsoft Identity Platform) acts as an **OIDC identity provider inside the existing Cognito user pool** — no backend change beyond attribute mapping and (recommended) the pre-signup account-linking Lambda. FastAPI keeps verifying the same Cognito JWTs; it never talks to Microsoft.

Unlike Google/Apple, Cognito has **no first-class "Microsoft" social provider** — you add Entra ID as a *generic OIDC provider* on the user pool. Functionally identical for the app.

## How it works

```
[Expo app] --signInWithRedirect(provider:{custom:"MicrosoftEntraID"})-->
  Cognito Managed Login: https://<domain>.auth.ap-south-1.amazoncognito.com/oauth2/authorize
    ?identity_provider=MicrosoftEntraID&redirect_uri=yesly://auth&code_challenge=...(PKCE)
  → 302 to https://login.microsoftonline.com/{tenant}/oauth2/v2.0/authorize
        (scopes: openid profile email)
  → user signs in with Microsoft account / MFA
  → Microsoft redirects code to
        https://<domain>.auth.ap-south-1.amazoncognito.com/oauth2/idpresponse
  → Cognito exchanges code (client_id + client_secret) at
        https://login.microsoftonline.com/{tenant}/oauth2/v2.0/token
  → Cognito validates id_token iss against configured issuer,
        maps claims (email, name) → creates/updates federated user
        (pre-signup Lambda may link to existing native user)
  → Cognito redirects code to yesly://auth
[Expo app] → exchanges code+verifier at /oauth2/token → Cognito Id/Access/Refresh JWTs
[Expo app] --Bearer <access JWT>--> FastAPI (unchanged JWKS verification)
```

## Setup — Entra admin center (client side)

1. **App registration** (entra.microsoft.com → App registrations → New):
   - Supported account types: see "tenant choice" below — for a consumer app like Yesly you want *"Personal Microsoft accounts and any org directory"* if consumers are the audience, but note the Cognito issuer gotcha.
   - Platform: **Web**; Redirect URI: `https://<your-domain>.auth.ap-south-1.amazoncognito.com/oauth2/idpresponse`
2. **Client secret**: Certificates & secrets → New client secret. **Max lifetime 24 months** (portal caps it; 6 months recommended by MS). Record expiry in the ops calendar — silent expiry = every Microsoft login breaks with a token-exchange 401.
3. **Optional claim `email`**: Token configuration → Add optional claim → ID token → `email` (also grant the implied `email` permission when prompted). Entra does **not always emit `email`** otherwise (org accounts may have no mail attribute; claim is mutable/unverified — treat accordingly).
4. Note **Application (client) ID** and **Directory (tenant) ID**.

### Tenant choice & the Cognito issuer gotcha

| Audience setting | Authorize endpoint tenant | `iss` in id_token | Works with Cognito OIDC? |
|---|---|---|---|
| Single tenant | `{tenantId}` | `https://login.microsoftonline.com/{tenantId}/v2.0` | Yes — issuer is fixed, exact match |
| Any org (multi-tenant) | `organizations` | per-user home tenant GUID | **No** — Cognito requires exact `iss` match; `common`/`organizations` discovery advertises `{tenantid}` templated issuer and each user's token carries their own tenant GUID |
| Orgs + personal (`common`) | `common` | home tenant GUID, or `9188040d-6c67-4c5b-b112-36a304b66dad` for personal MSAs | **No**, same reason |
| Personal accounts only (`consumers`) | `consumers` | `https://login.microsoftonline.com/9188040d-6c67-4c5b-b112-36a304b66dad/v2.0` | Yes — the consumer tenant GUID is fixed, so configure that as the issuer |

**Practical recommendation for Yesly (consumer RIA app):** register the Entra app for *personal Microsoft accounts* and configure Cognito issuer = `https://login.microsoftonline.com/9188040d-6c67-4c5b-b112-36a304b66dad/v2.0` **[verify — test full round trip; some Cognito discovery validations are picky about the consumers endpoint]**. If the client instead only cares about one workspace org, use that single tenant ID. True "any tenant + consumers" via one Cognito OIDC provider is not cleanly supported; the alternative is fronting with Entra External ID (a dedicated CIAM tenant with its own fixed issuer) federated into Cognito.

## Setup — Cognito (ap-south-1)

User pool → Social and external providers → Add identity provider → **OpenID Connect**:

| Field | Value |
|---|---|
| Provider name | `MicrosoftEntraID` (appears in `identity_provider=` param; ≤32 chars, no spaces) |
| Client ID / secret | from the app registration |
| Authorized scopes | `openid profile email` |
| Attribute request method | GET |
| Issuer URL | `https://login.microsoftonline.com/{tenant-or-consumer-GUID}/v2.0` — click "Run discovery"; endpoints auto-fill from `/.well-known/openid-configuration` |
| Attribute mapping | `email → email`, `name → name`, `sub → username` (automatic), optionally `preferred_username → custom attr` |

Then add `MicrosoftEntraID` to the app client's enabled identity providers.

## Expo app changes

Same path as Google today: Amplify `signInWithRedirect({ provider: { custom: 'MicrosoftEntraID' } })`, or raw `expo-auth-session` against the Cognito hosted `/oauth2/authorize` with `identity_provider=MicrosoftEntraID`, redirect `yesly://auth`, PKCE. No Microsoft SDK (MSAL) needed on-device — Cognito Managed Login brokers everything.

## Key configuration / endpoints

| Item | Purpose | Key values |
|---|---|---|
| Entra app registration | OAuth client Cognito uses | client ID + secret (≤24-mo expiry), platform=Web |
| Redirect URI (in Entra) | where Microsoft returns the code | `https://<domain>.auth.ap-south-1.amazoncognito.com/oauth2/idpresponse` |
| Issuer URL (in Cognito) | discovery + `iss` validation | `https://login.microsoftonline.com/{tenant}/v2.0` (consumer tenant GUID `9188040d-…` for personal accounts); `common`/`organizations` NOT usable |
| Authorize endpoint | user login | `https://login.microsoftonline.com/{tenant}/oauth2/v2.0/authorize` |
| Token endpoint | code exchange by Cognito | `https://login.microsoftonline.com/{tenant}/oauth2/v2.0/token` |
| Scopes | claims requested | `openid profile email` |
| Attribute mapping | Cognito user attributes | `email→email`, `name→name` |
| App client (Cognito) | enable provider | add `MicrosoftEntraID` to supported IdPs; callback `yesly://auth` |
| Pre-signup Lambda | account linking | `AdminLinkProviderForUser(native user, MicrosoftEntraID_<sub>)` |

## Auth model

- **App ↔ Cognito**: authorization code + PKCE via hosted Managed Login (public client, no secret in the app).
- **Cognito ↔ Entra**: confidential-client authorization code; Cognito holds the Entra **client secret** server-side and validates `iss`/`aud`/signature of Microsoft's id_token against the configured issuer's JWKS.
- **App ↔ FastAPI**: unchanged — Cognito access JWT, verified via the pool's JWKS. Microsoft tokens never reach the backend.

## Pricing (INR)

- **Microsoft side: ₹0** at Yesly's scale. Plain Entra ID app registrations/sign-in are free. If Entra **External ID** (CIAM) were used, the first **50,000 MAU/month are free** (core tier); beyond that ~US$0.03/MAU ≈ **₹2.6/MAU [verify current rate + FX ~₹86/USD]**. SMS/premium add-ons cost extra; not needed here.
- **Cognito side (already being paid)**: Lite tier free ≤10k MAU then ~$0.0055/MAU (~₹0.47) tapering to $0.0025; **Essentials** (needed for managed email OTP) free ≤10k MAU then **$0.015/MAU ≈ ₹1.3/MAU [verify]**. Federated Microsoft users count as ordinary user-pool MAUs — adding the provider itself costs nothing.

## India / compliance notes

- Login traffic transits Microsoft's global endpoints (`login.microsoftonline.com`); Microsoft doesn't guarantee India-resident processing of consumer sign-in metadata — fine under DPDP Act 2023 (no data-localisation mandate for this class of data) but note it in the privacy policy's cross-border transfer section. SEBI RIA record-keeping is unaffected: the system of record (Cognito user + your Postgres) stays in ap-south-1.
- Only `email`/`name` are pulled — minimal personal data, consistent with DPDP purpose limitation.

## Pitfalls

1. **Issuer exact-match**: Cognito rejects tokens whose `iss` ≠ configured issuer, so multi-tenant `common`/`organizations` is effectively unsupported. Pick single tenant, `consumers`, or an External ID tenant.
2. **Client secret expiry (≤24 months)**: hard failure on expiry; calendar the rotation, rotate by adding a second secret then updating Cognito.
3. **Missing `email` claim**: add the optional claim + `email` scope; still handle null email (some org accounts) — Cognito sign-in fails if a required mapped attribute is absent.
4. **Duplicate accounts**: Cognito treats each provider identity as a new user (`MicrosoftEntraID_xxxx`) even when the email matches an existing native/Google user. Use the existing **pre-signup Lambda + `AdminLinkProviderForUser`** pattern (same as Google) — link *before* first federated sign-in; linking after the federated profile exists requires deleting it first.
5. **Email is unverified/mutable in Entra tokens** — never use it as the primary key for linking without your own verification step (Microsoft explicitly warns against email-based authorization).
6. **Provider name is immutable** in Cognito once created and appears in usernames (`MicrosoftEntraID_<sub>`); choose it deliberately.
7. Logout: Cognito `/logout` ends the Cognito session but not the Microsoft session (SSO cookie remains in the system browser); usually acceptable, mention in QA.

## Sources

- AWS: [OIDC IdPs in user pools](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-oidc-idp.html), [Cognito pricing](https://aws.amazon.com/cognito/pricing/)
- Microsoft Learn: [Optional claims](https://learn.microsoft.com/en-us/entra/identity-platform/optional-claims), [ID token claims reference](https://learn.microsoft.com/en-us/entra/identity-platform/id-token-claims-reference), [App credentials](https://learn.microsoft.com/en-us/entra/identity-platform/how-to-add-credentials), [External ID pricing](https://learn.microsoft.com/en-us/entra/external-id/external-identities-pricing), [Migrate off email claim authorization](https://learn.microsoft.com/en-us/entra/identity-platform/migrate-off-email-claim-authorization)
- AWS re:Post: [Azure AD as OIDC provider in Cognito — issuer must be tenant-specific](https://repost.aws/questions/QUsw4UcwOlS1izHJcCUY8lVA/error-while-implementing-azure-ad-as-oidc-provider-in-aws-cognito-401-error-getting-token)
