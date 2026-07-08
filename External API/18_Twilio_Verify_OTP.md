# 18. Twilio Verify for Phone OTP — vs Cognito SMS OTP vs Self-Managed OTP on Indian Gateway

**Context:** Yesly authenticates via AWS Cognito email-OTP today. Adding phone-number verification (signup, phone-change, step-up for sensitive advisory actions) raises three options: (A) Twilio Verify, (B) Cognito's built-in SMS OTP via SNS, (C) self-managed OTP over the Indian gateway chosen in doc 17 (MSG91).

---

## 1. Twilio Verify — how it works

Managed OTP-as-a-service: Twilio generates, sends, stores, expires, and checks codes; you never store OTPs.

**Flow:**
1. Create a **Verify Service** once (console or `POST /v2/Services`) → get `VA...` Service SID. Configure code length (4–10), channels, Fraud Guard.
2. For India: complete Twilio's regulatory bundle — DLT still applies. You register your PE + templates on an Indian DLT portal, then submit Entity ID/Template ID/sender screenshots to Twilio (senderid@twilio.com pre-registration form); Twilio's aggregator chain does the scrubbing.
3. User enters phone in RN app → FastAPI calls
   `POST https://verify.twilio.com/v2/Services/{VA_SID}/Verifications`
   form params: `To=+91XXXXXXXXXX`, `Channel=sms` → response `status: "pending"`.
4. Twilio sends the OTP SMS (via its India DLT-registered route).
5. User submits code → FastAPI calls
   `POST /v2/Services/{VA_SID}/VerificationCheck`
   params: `To`, `Code` → `status: "approved"` (or `pending` on wrong code; 404 after expiry/max attempts).
6. On approval, mark phone verified in Cognito (custom attribute / admin API) and your DB.

**Channels:** sms, call (voice), whatsapp, email, TOTP, push, silent network auth. WhatsApp channel is notably cheaper/more deliverable in India.
**Auth:** HTTP Basic — Account SID + Auth Token (or API Key SID/Secret, preferred). Server-side only.
**Rate limiting / Fraud Guard:** built-in SMS-pumping fraud detection (blocks anomalous destination patterns), configurable rate limits per phone/IP — genuinely valuable; SMS-pumping fraud is rampant against Indian OTP endpoints.

**Pricing (INR):**
- $0.05 per **successful** verification (unconverted ≈ ₹4.2? No — see note) [verify]: Verify fee is $0.05/successful verification **plus** per-SMS India fee ~$0.0083–0.03 per message segment for India routes; net effective cost widely reported at **~₹0.45+ per delivered OTP** after forex + Twilio's currency margin — roughly **3–4× MSG91's ~₹0.12–0.15 self-managed cost**. [verify current rate card: twilio.com/en-us/verify/pricing and /sms/pricing/in]
- USD billing; forex spread ~2–3%; GST reverse-charge applies for import of services.

**Pitfalls:** India DLT onboarding via Twilio takes 3–7+ days and template changes require emailing Twilio; fixed Verify message templates limit branding unless you use approved custom templates; charged per successful verification but SMS attempts also billed; unverified trial accounts can't send to arbitrary Indian numbers; latency of US-hosted API from ap-south-1 is fine but data leaves India (DPDP: cross-border transfer is permitted but document it).

---

## 2. Option B: Cognito built-in SMS OTP (SNS)

- Cognito can do SMS MFA / phone verification out of the box via Amazon SNS.
- **India problem:** SNS SMS to India also requires the full DLT registration (Entity ID, template IDs configured in SNS "sender ID + template" settings for India). SNS India transactional SMS ≈ $0.002–0.003 [verify] — cheap — but SNS deliverability to India is mediocre, DLR visibility is poor, no OTP-specific features, and Cognito's message customization is rigid (must match DLT template exactly via a custom-SMS-sender Lambda to be safe).
- Cognito SMS also imposes SNS spending limits and sandbox exit friction.
- Verdict: works, cheapest to wire since Cognito is already in place, but weakest deliverability and worst debugging story in India.

## 3. Option C: Self-managed OTP on MSG91 (from doc 17)

- Use MSG91's OTP API (send/verify/retry) — MSG91 stores and checks the code, so "self-managed" is really "gateway-managed": FastAPI just proxies send + verify calls; no OTP storage in your DB.
- Cost ~₹0.15–0.20/SMS, no per-verification fee. Direct Indian routes → best deliverability + voice-OTP fallback.
- You own rate limiting / abuse protection (per-number and per-IP throttles in FastAPI, CAPTCHA on repeated requests) — this is the real cost vs Twilio's Fraud Guard.
- Ties into Cognito via a custom auth flow (Define/Create/VerifyAuthChallenge Lambdas) if phone becomes a login factor, or simply as an app-level verified attribute if it's only KYC-ish verification.

---

## 4. Recommendation for Yesly

**Use MSG91 self-managed OTP (Option C).** Reasons: ~3–4× cheaper per OTP than Twilio Verify, best India deliverability, INR billing, one vendor for both OTP and transactional alerts (doc 17), and data stays in India (clean DPDP story). Twilio Verify's genuine advantages — Fraud Guard and multi-channel — are outweighed at Yesly's stage; mitigate SMS-pumping with FastAPI rate limits + device attestation. Revisit Twilio Verify only if Yesly goes multi-country or OTP fraud losses materialize. Cognito/SNS SMS is not recommended for India deliverability reasons.
