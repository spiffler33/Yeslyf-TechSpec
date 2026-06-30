# Yesly — What Is Being Built: A Sourced Analysis

> **Purpose of this document.** This file consolidates *everything* the supplied materials say about Yesly (the product House of Alpha is building) into one explained reference. **Every claim below is followed by a bracketed source pointer** so any statement can be traced back to the file, sheet, row, or chart it came from.
>
> **Prepared:** 2026-06-24; **updated 2026-06-26** with the Figma prototype screenshots (`UX_images/`, S9) — see the new **§6B step-by-step visual flow breakdown**. **Scope:** the six documents in this directory, the 19 prototype screenshots, plus notes on the Boardmix board and the pasted email (see *Source Inventory & Gaps*).

---

## 0. Source Inventory & Gaps

| # | Source | What it is | Status |
|---|--------|-----------|--------|
| S1 | `yesly_architecture.html` — *"Yesly — Technical Architecture & Flow Reference" v0.2 DRAFT, 25 May 2026* | The master technical doc: architecture overview + 13 flowcharts. Prepared for handover to a development partner. | ✅ Fully read |
| S2 | `Yesly - UI Workflow.xlsx` | Screen-by-screen UI spec. 3 sheets: **Yesly Provided Flow**, **AA Integrated Flow**, **AA Process**. | ✅ Fully read |
| S3 | `Profile Matching Cards.docx` — *"YALPHO / Yalpho · Profile Matching Decision Tree v1.0"* | The logic that picks one of six "protagonist" personas from 3 onboarding answers. | ✅ Fully read |
| S4 | `New User Flow 03-06-2026.xlsx` | Earlier/parallel screen list + review feedback. 3 sheets: **New User**, **Sheet2**, **Sheet1**. | ✅ Fully read |
| S5 | `Yesly.xlsx` | Project hub. 5 sheets: **Links**, **New User Flow**, **Third Party Integrations**, **Workstream**, **Community**. | ✅ Fully read |
| S6 | `SG & HS NWGC OCT 2025.xlsm` | The actual financial-planning engine spreadsheet (NWGC). 30 sheets. | ✅ **Parsed sheet-by-sheet — full I/O schema in §6C** |
| S7 | **Boardmix board** — editor link `…/app/editor/9UYebZQ2eZ3z1VTGtF_fCw` (login-gated); public share link `…/app/share/CAE.CMyBCyABKhD1Rh5tlDZ5nfPVVMa0X98LMAZAAQ/L8l5Ga` (found in S8) | The user-journey diagram board. | ◑ Not separately transcribed — substance mirrored in S2 (see note). |
| S8 | **Email thread — "Re: Requirements for app"** (Spinach Experience Design ⇄ House of Alpha, 4 May – 18 Jun 2026) | Partner-onboarding correspondence. | ✅ **Received & integrated** (see §2A). |
| S9 | **`UX_images/` — 19 Figma screenshots** (captured 25 Jun 2026) | A single linear scroll-through of the **conversational (chat) prototype** of the onboarding→paywall journey. | ✅ **Read & mapped screen-by-screen (see §6B)** |

> **Boardmix (S7).** The originally-supplied `…/app/editor/…` link is login-gated (it redirects to a Boardmix sign-in form). The email thread (S8) reveals the correct **public read-only share link** (`…/app/share/CAE…/L8l5Ga`) and shows the design partner *itself* repeatedly hit access problems with the board in May 2026 [S8 → Eshani, 5–8 May; Somil, 6 May]. Per S2's *AA Process* sheet, *"Phase wise flow is the actual suggestion present in Boardmix"* [S2 → AA Process sheet], and S8 confirms the board carries the **user-journey** artefacts [S8 → Somil, 4 May].
>
> **2026-06-26 — partial Boardmix content now received.** A **master Stage 0–6 flow table** from the board was supplied as a screenshot and is transcribed in **§6.5** (it is richer than S2/S4 and conflicts with them on several points — flagged there). The board is reported to *also* contain: **versioned flows (e.g. "V3", "V5")** — consistent with the "Amendment 3/6" labels in §6.5, so confirm the canonical version; **per-provider Account-Aggregator flows (Setu and Finvu)** — relevant to the app, cf. §6.3/Chart 05; **detailed screens of *other* apps** — most likely competitor/reference benchmarking, **not Yesly's own** (to be confirmed); and **~3–4 flows** overall (likely the cold/warm/AA/subscribed journeys, or version variants). These remaining artefacts are **not yet transcribed** — pending the user supplying them.

> **Email (S8) — received & integrated.** The full thread is now in scope; its material content is summarised in **§2A** and woven into §3, §6, §7 and §11. Each email-sourced statement is cited as `[S8 → <sender>, <date>]`. (Also confirmed during this pass: the Google-Sheets "UI Flow" link Somil shared is **byte-identical to the local `Yesly - UI Workflow.xlsx` (S2)** — same 116,868 bytes — so it adds no new content.)

---

## 1. The Big Picture — What Yesly *Is*

Yesly is **"a mobile-first conscious money platform for Indian earners aged 25–45"** from House of Alpha (HoA) [S1 → Doc 00, *What Yesly Is*]. More precisely it is:

- **A financial planning + advisory funnel** delivered through community → planning → recommendations → human-advisor handoff [S1 → Doc 00].
- **A four-tier monetised product:** Free → Financial Plan (one-time) → Investment Plan (one-time) → Ongoing Subscription (monthly) → Human Advisory (off-app) [S1 → Doc 00, *What Yesly Is*; expanded in §3].
- **A read-consumer of users' financial data** via Account Aggregator (AA) consent, with CAS PDFs and statement uploads as fallback [S1 → Doc 00].
- **A SEBI RIA-compliant advisory channel** routing qualified users to House of Alpha advisors [S1 → Doc 00].
- **An execution-*initiating* platform** that places trades for subscribed users via **smallcase Gateway** and **BSE StAR MF** — but **Yesly is not the broker of record**; smallcase and BSE StAR are [S1 → Doc 00].
- **An in-house algorithmic core:** asset-allocation logic, rebalancing triggers, and (v2) idea generation are built by the **internal HoA engine team**, not the development partner [S1 → Doc 00, *What Yesly Is*; reinforced as a "Critical scope split" in *Internal Containers*].

### What Yesly is explicitly *not*
Not a broker · not a wallet/custodian · not a card issuer · not a lending platform · not on UPI P2P rails (UPI is used only as a subscription-mandate channel) · not a market-data terminal · not PCI-DSS in scope (card data is tokenised at the gateway, never touches Yesly servers) [S1 → Doc 00, *What Yesly Is Not*].

> **Why this framing exists (commercial intent).** The architecture doc states this scope *"eliminates approximately 40–60% of a generic Indian-fintech reference architecture"* and instructs that the partner *"must quote against this scope, not against a generic UPI-rails-and-cards diagram"* [S1 → Doc 00, *Cost implication for partner*]. So a major thing being "built around Yesly" is **a tightly-scoped partner engagement / RFP**, not just an app.

### Brand notes
- The community brand and member identity is **"YesLifers"**; a user is **"a YesLifer."** New users see *"Welcome to YesLifers"* and *"Before we plan, meet the tribe"* [S1 → Chart 01; S2 → A01 "Welcome to YesLifers"; S4 → New User row 2 "Popup – Welcome to Yeslifers"].
- The persona/plan layer carries the working name **"YALPHO / Yalpho, by House of Alpha"** in the profile-matching spec [S3 → header/footer].
- The landing page is live at **`https://yeslifers.vercel.app/`** and Figma prototypes live under the **"House of Alpha"** Figma file [S5 → Links sheet].

---

## 2. Who Is Building It & On What Timeline (Workstreams)

The project is run as ~14 workstreams with owners and deadlines [S5 → Workstream sheet]:

| Workstream | Objective | Priority | Deadline | Primary / Secondary | Status |
|---|---|---|---|---|---|
| Technical Architecture | Finalize architecture & execution roadmap | High | 2026-05-24 | Vatsal / Gaurav | **Complete** |
| Developer Finalisation | Finalize technology partner, architecture, roadmap | High | 2026-06-08 | Vatsal / — | **In-Discussion** |
| Content for marketing & landing page | Community landing-page + marketing content | High | 2026-06-11 | Somil / Bhuvanaa | — |
| Content for app | Close out flow, content, precise app requirements | High | 2026-06-08 | Somil / Bhuvanaa | — |
| Finalising Designs with Ihstiaq & BBC | UI/UX direction, branding, design system | High | 2026-06-12 | Somil / Bhuvanaa | — |
| Prepare the Sample Plan | Sample user/advisory plans for the platform | High | 2026-05-26 | Somil / Bhuvanaa | To be discussed with Core |
| Community Flow & App Flow | Define complete user journey & engagement | High | 2026-06-08 | Somil / Kajal | — |
| Community App & Integration Ecosystem | Build community platform + integrations | High | 2026-06-14 | Vatsal / Kajal | — |
| MVP for Financial Planning | Launch first version of the planning module | High | 2026-06-14 | Somil / Bhuvana | — |
| Advisory Framework | Advisory servicing & consultation model | Mid-High | 2026-06-21 | Somil / Harish | — |
| Operations & Governance | Operational structure, ownership, reporting | Mid-High | Ongoing | Somil / Kajal | — |
| Implementation of Advisory | Execution workflows for investments | Low | TBD | Kajal / Somil | — |
| Growth & Engagement Strategy | Community acquisition, engagement, retention | Low | TBD | Bhuvana / Vatsal | — |
| Third-Party Integrations | AA / BSE / Smallcase | — | — | — | — |

**People referenced across the docs:** Vatsal (architecture, developer finalisation, integrations — also the account owner, email per session context), Gaurav (architecture secondary), Somil (content/design/plan/MVP/advisory — the most-loaded owner), Bhuvanaa/Bhuvana (content, design, growth), Kajal (community, ops, advisory implementation), Harish (advisory framework; also features in app videos — "Harish's Video on IP" [S4 → New User row 26]), Ihstiaq & **BBC** (external design/branding partners; "to be re-done by BBC Team for all the 6 avataars" [S4 → New User row 7]). [All: S5 → Workstream sheet unless noted.]

> **Read:** the technical architecture is *done*; the live open question as of these docs is **which development partner** builds it (Developer Finalisation = "In-Discussion", due 2026-06-08), and **closing the app content/flow** (multiple 2026-06-08/06-14 deadlines). The whole architecture doc (S1) exists to support that partner-selection and quoting process [S1 → Doc 00, *How to read this document* + *How to Use This Document With a Partner*].

---

## 2A. Partner Engagement & the June-2026 Conversational Pivot (Email Thread — S8)

This is the single most important *new* context the email adds: **who is building the app, and a late change of direction that post-dates every other document here.**

### Who is involved
- **Spinach Experience Design** — the **UI/UX + design-systems + engineering partner** being onboarded ("UI UX Design | Design Systems | Engineering"). Led by **Eshani Kumbhani** (Co-founder & UX Director); team members **Saquib** and **Sahil** also need board access [S8 → signatures; Eshani, 5 May].
- **House of Alpha (HoA)** — **Somil Gogri** (product/requirements owner) and **Vatsal Parikh** (account owner, `vatsalparikh@gmail.com`) [S8].
- **Greenware Solutions** — **Gaurav Kohli** answers the technical/architecture scoping questions on HoA's behalf — i.e. the **technical-architecture lead** (consistent with S5's "Gaurav" on the *Technical Architecture* workstream) [S8 → Gaurav, 5 May; cross-ref S5 → Workstream].
- *(Distinct from the "BBC team" that re-renders the six avatar sample plans [S4 → row 7], and "Ihstiaq & BBC" in S5's design workstream — those are the branding/content side; Spinach is the UX/engineering partner.)*

### Engagement timeline (all [S8])
- **4 May 2026** — Somil shares the **Boardmix user-journey board** + the **Google-Sheets UI-flow** (= S2) as the references for Spinach.
- **5 May** — Eshani sends **10 scoping questions**; Gaurav answers inline (below). Spinach also asks whether **TPM / BA / PO** roles must be covered by them; offers to sign an **NDA** first.
- **5–6 May** — Spinach repeatedly **can't access the Boardmix board** even after access is "provided"; Vatsal re-shares the Excel instead.
- **8 June** — Eshani asks whether the Boardmix artefacts + UI-flow sheet are **still the final source of truth for Figma** (notes the board has "many artefacts").
- **10 June** — **the design pivot** (below).
- **16–18 June** — Eshani requests the **final third-party-integration list**; Somil sends it (**identical to S5's list**); Spinach will filter it to the "absolutely final" providers.

### ⚠️ The 10-June-2026 pivot — one single conversational interface
After an internal HoA meeting, Somil told Spinach the app direction changed [S8 → Somil, 10 June]:
1. **User journey:** the *entire* journey — intro video → Financial Plan → Investment Plan → Action Plan — is to be delivered **within a single conversational interface**, feeling like an **interactive chatbot**, with users progressing seamlessly in the same window.
2. **Output experience:** non-interactive outputs are shown in the chat window **plus a download-as-PDF** option; a **floating star icon (bottom-right)** acts as a **repository of all generated outputs** (tap any to revisit); outputs appear in the sequence **Headline Response** (e.g. *"We are processing your data and preparing your recommendations."*) → **App-Viewable Output** → **PDF Download**.

> **Ambiguities Spinach flagged on this pivot — now RESOLVED by the Figma prototype (S9, §6B):**
> - Is the "chatbot" experience **just a video**, or a **real coded chat UI**? Eshani: *"To ensure we've understood this — it is only a video, not a UI UX chat bot experience that needs to be coded."* → **ANSWERED: it is a real coded chat UI.** The S9 screenshots show a fully conversational interface — yesly avatar bubbles, user reply bubbles, in-line option chips/sliders, embedded video cards — not a single video [S9, all screens; see §6B].
> - What exactly are **"non-interactive outputs"** → **ANSWERED:** they are the engine-generated result cards (persona card, Clarity Insight, Full Picture) shown inline in the chat **with a "Download PDF" affordance and an "Output N of many" label** [S9 → 23-53-29, 23-53-58, 23-54-23; §6B steps 6/13/18].
> - The **floating-star repository** → **ANSWERED & sampled:** it is the **"dock" bottom-right**, carrying a running count badge (0→1→2→4→…) that increments as each output is generated; copy literally reads *"Saved to your dock at the bottom right"* [S9 → 23-53-18 onward; §6B].
>
> **Implication.** The pivot is **confirmed as the build direction.** §6's A–G *screens* are reframed into conversational *turns* — the underlying questions, captured data, outputs and tiers are unchanged, but **the chat reorders some sections** (income/expenses now come *after* goals + the first Clarity insight; see §6B for the actual prototype order). Treat **§6 as the canonical content/data spec** and **§6B as the as-built presentation order.**

### Scope clarifications from Gaurav (HoA / Greenware), 5 May [S8 → Gaurav, 5 May]
These predate S1 (25 May) and in places refine or sit in tension with it:
- **Platform:** mobile app is the requirement (**both App Store + Play Store, all regions**), **browser as fall-back**, **mobile-first single design language**.
- **Tech decisions:** *none finalised as of 5 May* — so S1's "finalised" stack was locked later (between 5 May and its 25 May date).
- **Scale targets (NEW — not stated in S1): ~100K downloads, ~10K concurrency** — "architecture needs to be planned accordingly."
- **APIs:** "largely **CRUD**-oriented"; design flow already in place, UI being developed.
- **Products at launch:** "largely a financial planning and investment tool." **Online investment supported for Mutual Funds only to start.** **Stocks are placed via the customer's own external trading app — "Our app will not place orders in stocks."** (This refines S1/§11: smallcase + BSE StAR cover MF/basket execution; Yesly never places equity orders.)
- **Integrations:** **BSE StarMF** (possibly) + **Account Aggregation**; **Mar-tech = WebEngage** ("etc.") — note S1 names **Mixpanel**; WebEngage is the email's stated choice (unresolved).
- **Admin panel:** *"We don't currently envisage an extensive admin panel. A basic panel would however be helpful."* — **direct tension with S1's Chart 13**, which scopes a serious **7-surface, 4-role** console (4–6 eng-weeks). S1 is the later, more-detailed doc; the SEBI/compliance surfaces in it are likely non-optional even if the broader console is trimmed. **Reconcile with client.**
- **Existing systems:** HoA has an **internal tool already holding client data + financial plans**; some data comes from there (this is the "HoA planning APIs" / NWGC engine of §10).
- **Pre-estimation artefacts:** architecture diagrams, service contracts, security requirements *not yet available* as of 5 May (S1 later supplies the architecture).

### Process items still open in the thread [S8]
A **final UI-flow sheet showing the delta between *mapped* vs *pending* screens** (Spinach's explicit request, 10 June) · confirmation of **TPM / BA / PO** role ownership · the **NDA** · and Spinach narrowing the third-party list to "absolutely final" vendors.

---

## 3. The Monetisation Model — Four Tiers

This is the spine of the product [S1 → Doc 00 *The Four Tiers*; Chart 03 *Plan Tier Progression*].

| Tier | Product | Pricing | What it includes |
|---|---|---|---|
| **Free** | Community + Sample | — | YesLifers community access, sample plan PDF for the matched archetype, persona reveal, limited Clarity & Blindspots [S1 → Four Tiers] |
| **Tier 1** | **Financial Plan** | One-time, low (**~₹249 indicative**) | Comprehensive household-aware plan: goals, life events, budget, "Hell Yes" decisions, *generic* asset-allocation framework, risk profile (SEBI suitability) [S1 → Four Tiers; Chart 03] |
| **Tier 2** | **Investment Plan** | One-time, premium ("not cheap") | Portfolio-specific allocation against *actual holdings*. **Hard-gated on portfolio data** (AA / CAS PDF / statement upload / manual) [S1 → Four Tiers; Chart 01 *Portfolio Data Required*] |
| **Tier 3** | **Ongoing Subscription** | Monthly recurring | Live dashboard, drift detection, rebalancing alerts, curated idea notifications, in-app execution (v1.5; deep-link in v1), monthly summaries, check-ins [S1 → Four Tiers; Chart 04] |
| **Tier 4** | **Human Advisory** | Off-app pricing | Direct planner conversation *outside* the app. App only provides a context window for the advisor + check-in surface. HoA-internal opt-in only [S1 → Four Tiers; Chart 03] |

**Tier intent in one line each** [S1 → Chart 03]:
- Free → *lead* (value to user: low; value to Yesly: lead).
- Tier 1 → *paid lead qualifier* ("cheap entry; heavy deliverable").
- Tier 2 → *meaningful one-time revenue + proof-of-value before the subscription ask.*
- Tier 3 → *recurring revenue engine (MRR base).*
- Tier 4 → *AUM acquisition channel for HoA's broader business* (top of the value pyramid).

**Tier badges & dashboard density** (from the UI spec) [S2 → AA Integrated Flow sheet, *Tier and dashboard density*]:

| Tier | Badge | Colour | Dashboard density | Primary emotion |
|---|---|---|---|---|
| Free | Free | Warm grey | Low – snapshot + community | Curiosity & possibility |
| Option 1 | Financial Plan | Blue `#2C5F8A` | Medium – plan + pathway + community | Agency – "I know what to do next" |
| Option 2 | Plan + Advisory | Green `#2D6A4F` | High – live portfolio + advisory | Control – "I can see everything" |
| Option 3 | Full Service | Gold `#A8803A` | Curated – advisor + health score | Trust – "someone has this" |

> **Naming caution:** S1 uses "Tier 1–4"; S2 uses "Option 1/2/3" + Free; S4/S5 talk about "Paywall 1/2/3." These are the **same ladder**: Financial Plan = Tier 1 = Option 1 = Paywall 1; Investment Plan = Tier 2 = Option 2 = Paywall 2; Subscription = Tier 3 = Option 3/"Ongoing Serv" = Paywall 3 [reconciled across S1 Four Tiers, S2 Tier table, S4 Sheet2, S4 New User rows 23–25].

### Downgrade / churn & tier-skipping logic [S1 → Chart 03]
- Subscription cancelled → retain Tier 2 (keep Investment Plan, lose live features).
- Investment Plan refunded in window → back to Tier 1. Financial Plan refunded → back to Free.
- Subscription `past_due` beyond grace → auto-downgrade to Tier 2.
- Dormancy 90 days → reactivation campaign; 365 days → hard churn.
- DPDP erasure request → **anonymise** (audit retained per SEBI; see §9).
- **Tier-skipping** is allowed: Free → Tier 2 (skip FP, if portfolio data + risk profile present), Free → Tier 3 (discouraged, promo/cohort only), Tier 1 → Tier 3 (still needs portfolio data, so effectively forces Tier 2 work; bundle pricing possible). The Tier-2 gate is **portfolio data + risk profile**, not prior-tier purchase.

---

## 4. The End-to-End User Journey (Funnels)

The architecture defines two acquisition funnels plus the subscribed loop.

### 4.1 Cold funnel — New User → First Plan [S1 → Chart 01]
Marketing/SEO/referral → Landing page → Email/phone signup → OTP verify → **Community-first on-ramp** (Welcome "Before we plan, meet the tribe" → optional YesLifers Pledge → community access with daily prompt + "Today / The Road / Decisions" → 1–3 light engagement sessions) → **Financial onboarding** (3 life-stage questions → persona/archetype reveal → avatar creation → free sample-plan PDF preview) → if "wants their own plan" → **Tier-1 data collection** (household → income+expenses → assets+liabilities *manual, no AA yet* → goals → SEBI risk-profile questionnaire → life events → budget → Clarity & Blindspots) → **Paywall 1 (Financial Plan, ~₹249)** → Razorpay one-time → plan delivered (PDF + in-app + email) → **data gate** (Investment Plan teaser → pick portfolio data source) → **Paywall 2 (Investment Plan)** → **Paywall 3 (Monthly Subscription)** → onboarded to live app (Chart 04).

- **Pledge is optional, not a gate** — a `pledge_taken` flag drives a visible community badge; users who decline can pledge later [S1 → Chart 01 Notes].
- **Three distinct paywall events.** FP (cheap entry, heavy doc) → IP (AA-gated) → Subscription [S1 → Chart 01 Notes].
- **Risk profile is part of Tier 1** because SEBI requires suitability before any specific recommendation [S1 → Chart 01 Notes].
- **Confidence flags:** AA path = high; CAS-PDF parsing = moderate (covers NSDL+CDSL+most MFs, no bank balances); bank/card statement parsing = moderate (formats vary, OCR may be needed); manual entry = low (users will abandon) [S1 → Chart 01 *Confidence flags*].

### 4.2 Warm funnel — YesLifers Member → First Plan [S1 → Chart 02]
Already-in-community user is triggered by a founder nudge / "Get your plan" CTA / event participation / cohort campaign / a "I want to plan" decision thread → Yesly intro in-context → **context inherited** (identity already verified, email confirmed, display name/avatar, pledge status) → **community on-ramp skipped** → same financial onboarding & paywall ladder from Paywall 1 onward.

- Warm users skip pledge prompt, avatar creation (if set), welcome on-ramp → **est. 4–7 fewer screens** than cold [S1 → Chart 02 Notes].
- **Cross-system identity bridge is v1 scope:** YesLifers runs on Supabase, Yesly on AWS Cognito; inherited identity is propagated via a cross-system user mapping during the warm-funnel transition [S1 → Chart 02 Notes].
- Conversion at every paywall **should measurably exceed** the cold funnel — that's the commercial logic of community-first; trigger moments need **frequency capping** [S1 → Chart 02 Notes].

### 4.3 Subscribed monthly lifecycle (where the engineering effort lives) [S1 → Chart 04]
A repeating 4-week loop:
- **Week 1 – Portfolio health check:** refresh AA data → compute drift vs target → if drift > threshold → generate rebalancing rec → **compliance gate (curated review in v1)** → notify → rebalancing screen → if user accepts → v1 deep-link to smallcase/BSE StAR (v1.5 in-app) → order placed.
- **Week 2 – New idea surfaced:** HoA publishes curated idea → compliance gate → suitability-match to user → if match → notify → idea detail → if interested → execute.
- **Week 3 – Behavioural check-in:** "how's the journey" → if flagged → escalate to Tier 4 advisor.
- **Week 4 – Plan health summary:** monthly summary screen.
- **Billing (any week):** Razorpay monthly charge → on failure, dunning days 1/3/5/7 → if unrecovered, auto-downgrade to Tier 2.
- **Escalation (any week):** user-initiated "talk to advisor", major life event in check-in, or drift/risk threshold → route to HoA advisor scheduling → in-app calendar slot → off-app conversation → advisor enters summary via lightweight console → summary visible in-app.
- Cadence is a rhythm, not rigid; **notification fatigue is a real risk** (comms preferences are v1 scope); **the compliance gate before any user-facing recommendation is non-negotiable** [S1 → Chart 04 Notes].

---

## 5. The Persona / Avatar System — "Profile Matching" (Yalpho)

A central, distinctive thing being built is a **6-persona archetype engine** that mirrors the user back to themselves before asking for data. Source: the entire **`Profile Matching Cards.docx`** [S3].

### 5.1 How it works
After the three onboarding questions (Q1 life stage, Q2 emotional state, Q3 aspiration), server-side logic selects **one of six protagonist profiles** and shows a pre-rendered profile card (Stage 2B). *"Selection takes <100ms. The user should never perceive a loading state between Q3 and the profile card."* [S3 → Overview + System Note]. Each profile is defined by **a dominant "financial flaw" archetype** — the hidden pattern holding them back [S3 → *The Six Profiles*].

### 5.2 The six profiles [S3 → *The Six Profiles*]
| Profile | Flaw | Primary match (Q1 · Q2 · Q3) | Who they are |
|---|---|---|---|
| **Rajesh** | Self-Neglect | Life Stage C · Emotion A/C · Aspiration B/D | Established, responsible, caring for everyone, deprioritising himself. *Also the default fallback profile.* |
| **Arjun & Meera** | Overconfidence | A · A · A/C | High earners with a FIRE plan on optimistic, un-stress-tested assumptions |
| **Nisha** | Directionlessness | A · D · A/D | High earner investing without structure or goal-mapping |
| **Vikram & Priya** | Inattention | B · B/D · B/D | Young family, dual income, reactive financial decisions |
| **Aditya** | Conflation | D · A/B · A | Business owner; personal & business finances blurred (most *specific* profile) |
| **Deepika & Sanjay** | Outdated Assumptions | D · C · B/D | Income transition — one income now irregular / dual-income plan under strain |

### 5.3 Scoring system [S3 → *Scoring System* + the three score tables]
- Additive: **Q1 + Q2 + Q3 = total.** Max 9 (3+3+3). **Minimum selection threshold = 5.** If nothing scores ≥5 → fall back to **Rajesh**.
- Weights: **Q1 carries the most predictive weight** (primary filter), Q2 secondary, Q3 the tiebreaker [S3 → System Note].
- Full Q1/Q2/Q3 → profile score tables (0=unlikely, 3=primary indicator) are tabulated in the doc, plus a **"Full Combination Matrix"** listing the highest-confidence (★★+) Q1+Q2+Q3 → profile mappings [S3 → score tables + matrix]. Example primary (★★★) rows: `C·A·B → Rajesh`, `A·A·A → Arjun & Meera`, `A·D·A → Nisha`, `B·C·B → Vikram & Priya`, `D·A·A → Aditya`, `D·C·B → Deepika & Sanjay` [S3 → Full Combination Matrix].
- ⚠️ **Implementation note:** in the raw score tables, **Aditya and Deepika & Sanjay share a single column** ("Aditya / Deepika") — the additive score alone *cannot* tell them apart. They are separated only by the tiebreaker rules (§5.4): `Q1=D + Q3=A → Aditya`; `Q1=D + Q2=C → Deepika`. If the engine is built from the score tables alone, those rules must be coded explicitly [S3 → score tables].

### 5.4 Edge cases & override rules [S3 → *Edge Cases & Fallback Rules*]
1. **Default fallback** → Rajesh if no profile scores ≥5 (most universally relatable, lowest misID risk).
2. **"Show me a different profile"** → show next-highest scorer; never repeat; **max 3 profiles/session**. Store the full ranked score array at match time; don't recalculate.
3. **Aditya special case** → if Q1=D but no other signal (Q2≠A/B or Q3≠A), default to Deepika & Sanjay (over-assigning the business-owner profile risks misID). Suggestion: add a soft "Are you self-employed?" Y/N qualifier when Q1=D.
4. **Tone calibration override** → if Q2=C (Anxious), apply the *gentle tone variant* through Stages 3–4; deliver the blind-spot "with extra care."
5. **Aspiration anchoring** → Q3 is stored separately and passed to Stage 4's "What Could Be" generator: A=foreground freedom/time, B=safety/legacy, C=permission/enjoyment, D=confidence/peace-of-mind.

### 5.5 Implementation notes [S3 → *Implementation Notes*]
- Stage 2A stores three values — **L1** (life stage), **E1** (emotional state), **G1** (aspiration) — in the **session only** (not persisted), used for (a) profile matching, (b) tone calibration, (c) Stage 4 anchoring [S3 → Data Storage]. *(Note: S2's stored-as columns use the same L1/E1/G1 labels — see §6.1.)*
- Pseudocode `matchProfile(L1,E1,G1)` sums the three score-table lookups, sorts descending, returns `'rajesh'` if top score <5, applies tiebreaker on ties, and stores the ranked list [S3 → *Scoring Algorithm — Pseudocode*].
- **All six cards are pre-rendered static screens** at app init; the matcher returns a profile-ID string and the app navigates to the cached card — *no dynamic content generation at match time* [S3 → *Profile Card Rendering*].
- **A/B-test target:** cards tested for resonance within 3 months; track time-on-card, tap-through to Stage 3, "show different profile" rate. **Target tap-through >85%**; review any card below 75% [S3 → System Note].

> **Cross-doc link:** these six personas are the "six avataars" the **BBC design team** must re-render the before/after sample plans for [S4 → New User row 7: *"Before & After (to be re-done by BBC Team for all the 6 avataars)"*], and the "persona/archetype reveal → avatar creation" step in the funnels [S1 → Charts 01–02].

---

## 6. The Screen-by-Screen UI Specification

Two parallel specs exist. **`Yesly - UI Workflow.xlsx` (S2)** is the most detailed and current (it uses the A/B/C/D/E/F/G flow-letter scheme and a full input schema). **`New User Flow 03-06-2026.xlsx` (S4)** and `Yesly.xlsx → New User Flow` (S5) are an earlier numbered list (1.0–29.0) carrying *review feedback*. Both are reconciled below.

### 6.1 The current flow (S2 → "Yesly Provided Flow" sheet)
Columns captured per screen: Screen ID, name, Step, Tier access, Primary action, Key design intent, Question #, Question text, Input type, Options/Bands, Required, **Stored As**, Feeds Into, **AA can Fill?** [S2 → Yesly Provided Flow header row].

**Flow A — Onboarding (A01–A09)** [S2]:
- A01 Welcome to YesLifers (pre-app) · A02 The Pledge – 8 commitments (each individually tapped; button locked until all 8 tapped) · A03 "Pledge complete – you are a YesLifer" · **A04 Q1 Life stage** ("Where are you in life right now?", single-select 4: *Building foundation / Growing family complexity / Earning well but uncertain / In a transition*, stored as **L1**) · **A05 Q2 Emotional state** ("How does your financial life feel right now?": *Confident but uncertain underneath / Busy – finances keep getting pushed / Anxious – something feels off / No clear picture yet*, stored as **E1**) · **A06 Q3 Aspiration** ("What would feel like success to you five years from now?": *Financial independence – options not obligations / Security for family / Freedom to make big choices / Not sure yet*, stored as **G1**) · A07 Gap insight – "your pattern" · A08 Persona card – "someone like me" (two-column dark card: name + story + what they worry about + what they want) · A09 "Build your picture – Step 1 begins."

**Flow B — Step 1 "Clarity" (B01–B15)** [S2]: B01 intro (personalised to the Q2 answer); **Emotional sub-section** B02 "what worries you" (free text, stored **E1**), B03 "what you avoid" (multi-select chips: retirement savings / insurance gaps / how much I spend / investment returns / total debt / "Not really – I look at everything", **E2**), B04 "what would feel different" (free text, **E3**; 2 of 3 required to complete section); **Income** B05 (range bands *Under 50K…5L+* + exact, plus spouse/rental/other-income toggles **I1–I4**); **Investments** B06 (MF/EPF/NPS — **INV1–INV3**, range+exact, bands *Under 5L…1Cr+*) and B07 (FD+savings/Gold/ULIP/Home — **INV4–INV7a**, ULIP & home toggles reveal sub-inputs); **Liabilities** B08 (Home/Car/Personal loan + EMI — **L1–L3**, plus a prominent "No liabilities – confirm and continue" button **L4**, "neutral framing, never shaming"); **Protection** B09 (term cover **P1**, health-insurance type employer-vs-personal **P2/P2a**) and B10 (dependants — spouse/children/parents toggles, children reveal age chips 0-5/6-12/13-18/18+ — **P3–P5**); B11 section-completion tab bar (5 pill tabs, dots grey→green, CTA disabled until all 5 green); **Clarity insights** B12 net worth + WAR (two big numbers + realisable/locked/property blocks), B13 projections (current path / better allocation / full potential bars), B14 blindspots (urgent-red / warning-orange / opportunity-gold cards); B15 "Share moment – Step 1 complete" (share with tribe OR keep private, both valid).

**Flow C — Step 2 "Yes Life" – Goals (C01–C09)** [S2]: *"Not a goals form. Your Yes List."* — C02 goal-type picker (chips: Travel/Education/Home/Sabbatical/Career change/Side passion/Family milestone/Other, **YL1**), C03 goal "conversation" (timeline chips + budget range + who-it's-for, **YL1a–c**), C04 goal card, C05 free-form desires (**YL2**), C06 non-discretionary life events (age-at-event + cost, **YL3**), **C07 Financial-independence anchor** ("the most important question in Yes List": FI age chips 45/50/55/60/65/No-fixed **FI1**, + monthly retirement income needed **FI2**), C08 goals-mapped insight, C09 share moment.

**Flow D — Step 3 "Yes Budget" – Expenses (D01–D10)** [S2]: D01 intro (income + EMI pulled read-only from Step 1 — "signals the app remembers"), D02 fixed expenses (**YB1**), D03 variable necessities (**YB2**), D04 **guilt-free spending** ("what you say yes to without explanation", **YB3**), D05 current monthly investing/SIPs (**YB4**, "not EMIs or premiums – those are already counted"), D06 **live surplus calculation** (sticky footer, starts at full income, turns gold when positive), D07 surplus + gap insight (gap card "in gold not red"), D08 19-year projection bars, D09 share moment, **D10 Upgrade prompt** ("Not a paywall – a graduation"; plan teaser with 3-4 specific items from their data; **₹2,999** prominent — *FREE-ONLY screen*).

> **Pricing flag — partly resolved by the prototype (S9):** D10 shows **₹2,999** for "Option 1" [S2 → D10]; S1 gives Tier 1 as *"~₹249 indicative"* [S1 → Four Tiers]. The Figma prototype now shows **two priced cards: "Essential Plan ₹2,999"** (≈ Tier 1 Financial Plan: *"Three biggest blindspots, fixed"*) and **"Growth Plan ₹4,999"** (≈ Tier 2: *"Everything in essential + Goal-based financial roadmap + SIP + tax-optimisation plan"*) [S9 → 23-54-00 & 23-54-25; §6B steps 14/19]. So **₹249 is a stale placeholder**; the working numbers are **₹2,999 (Essential) / ₹4,999 (Growth)**. Subscription (Tier 3) price still unconfirmed. (S1 lists pricing as an *Open Decision*, see §11.)

**Flow E — Step 4 "Hell Yes!" (E01–E08)** [S2] — *Option 2+ only*: E01 decision picker (5 decision cards: Buy a home / Career change / Sabbatical / Second child / Early retirement / Custom, **HY1**), E02 Path-A inputs, E03 Path-B inputs (with Path-A summary shown), E04 scenario summary, **E05 full-scroll life walk-through – 8 stops** (sticky scenario strip, TODAY + 8 ages, two columns side by side), E06 "Hell Yes?" decision (Path A / Path B / Not yet / Talk to advisor), E07 decision recorded + share + advisor booking, **E08 Hell Yes! teaser – locked** (shown to Free + Opt 1; tool description + **3 real WhatsApp conversation excerpts** + upgrade CTA — *"excerpts do the selling"*).

**Flow F — Home / dashboards (F01–F09)** [S2], one per tier: F01 Free home (snapshot recap, blindspot card, plan teaser, upgrade CTA, community events, Hell Yes teaser, tribe feed), F02 Financial-Plan home (most-urgent action red border, goal progress bars, "Yes Pathway" checklist, Hell Yes locked), F03 Plan+Advisory home (AA data-freshness timestamp, three metric blocks, WAR progress bar, holdings with retain/exit/new-SIP tags, SIP-mandate status, rebalancing checklist), F04 Full-Service home (advisor card + next call, circular portfolio-health score, advisor action timeline), F05 My Plan – Yes Pathway checklist (This Week / This Month / Month 5+), F06 My Plan – live portfolio via AA (holdings, XIRR vs benchmark, rebalancing), F07 Tribe feed (read-only, initials avatars, first name + last initial), F08 Community events (RSVP), F09 Profile & settings (incl. **AA consent management**).

**Flow G — System screens (G01–G09)** [S2]: G01 splash (≤2s, no spinner), G02 onboarding error, G03 auto-save confirmation, G04 session resume ("Welcome back, [Name]"), G05/G06 upgrade purchase flows (Option 1 / Option 2, "graduation" framing), G07 advisor booking (Option 3 — calendar), G08 empty states ("specific copy – never generic"), G09 generic error ("never 'Something went wrong' – always specific").

### 6.2 The AA-integrated variant of the same flow (S2 → "AA Integrated Flow" sheet)
This sheet re-lists every screen with **AA can Fill? / AA Source / What AA returns** columns, and is explicitly *"the actual suggestion present in Boardmix"* [S2 → AA Process sheet note]. Key AA mappings [S2 → AA Integrated Flow]:
- **Income I1** ← Bank AA (12-month avg salary credits, exact); **I2 spouse** only works for *joint* accounts; **I3 rental** pattern-detected; **I4 other** irregular credits flagged for user confirmation.
- **MF INV1** ← CAMS RTA + KFin RTA via AA (exact folio-level holdings, NAV, AMC, scheme, units, lock-in); **EPF INV2** ← EPFO via AA; **NPS INV3** ← CRA via AA (Tier 1+2 balance); **FD INV4** ← Bank AA; **Gold INV5** ← *not in AA* (physical gold not available; digital only if SGB via demat); **ULIP INV6** ← Insurance AA (IRDAI FIP); **Home INV7a** ← *AA can't give this; user enters*.
- **Liabilities L1–L3** ← Bank AA (loan account, exact outstanding + EMI).
- **Protection P1** term & **P2** health ← Insurance AA; dependants P3–P5 ← *AA can't help*.
- **Expenses YB1–YB4** ← Bank AA transaction categorisation: fixed ~70% accurate; variable & guilt-free "rougher, needs confirmation"; SIPs via CAMS/KFin RTA (exact mandates).

**AA vs manual — the screen/time savings** [S2 → AA Integrated Flow, *AA path vs manual path*]:
| Section | Manual | AA |
|---|---|---|
| Income | 1 screen + 4 inputs | Auto-filled, **0 screens** |
| Investments | 2 screens + 10 inputs | Auto-filled except gold + home (1 micro-screen) |
| Liabilities | 1 screen + 4 inputs | **0 screens** |
| Protection cover | 1 screen + 3 inputs | **0 screens** if FIPs connected |
| **Total Flow-B** | **15 screens / ~25 inputs / ~15 min to first insight** | **9 screens + AA07 / ~6 inputs / ~6 min** |

> Design honesty mandate: *"be honest in the UI about AA limits … say 'Connect your accounts – we'll pull what we can (investments, bank data, insurance). We'll ask you about the rest.'"* [S2 → AA Process sheet]. Also: *"AA should be done only for the users who want exact data"* [S2 → AA Integrated Flow note].

### 6.3 The AA consent micro-flow (S2 → "AA Process" sheet — via Setu)
7 steps, ~95s total [S2 → AA Process sheet]: (1) FIP selector — Yesly pre-sends up to 6 FIPs (10s); (2) mobile number (10s, often pre-filled); (3) AA-handle discovery — Setu routes to best-performing AA; existing handle → straight to OTP, else quick registration (15–30s); (4) OTP to link accounts (15s); (5) consent review — FIPs, data types, duration, purpose, frequency (20s); (6) consent-approval OTP — single, via "Setu's unified OTP auth" (10s); (7) success / background data fetch + redirect (15–30s). *"bank + MF + insurance consent in one session with just one OTP each."*
- **AA-derived Yesly fields:** Age/DOB ← PAN profile; Monthly surplus ← income credits − recurring debits; EMI as % of income; Fixed-commitments % [S2 → AA Process sheet].
- **Documented AA real-world limits** (cited to `casparser.in/blog/state-of-account-aggregator-2026/`): FD/RD support only ~40% of banks (Axis/SBI/PNB/HDFC/IndusInd + some RRBs); 57 insurers "live" but most on only 1–2 AAs (SBI Life leads with 10, **LIC is on exactly 1 AA – OneMoney**); **joint accounts not supported by any bank**; NRE/NRO accounts not discoverable [S2 → AA Process sheet]. *(This is the closest thing to an external reference embedded in the docs.)*

### 6.4 What the earlier "New User Flow" review tells us (S4 / S5)
The numbered 1.0–29.0 flow [S4 → New User sheet; mirrored in S5 → New User Flow] adds intent + *review feedback* not in S2:
- Heavy use of **video**: Video 1 welcome (non-skippable), Video 2 sample→personal-plan transition (30s, "Need content from BBC"), Video 3 free→paid paywall explainer (30s), Video 4 "Harish's Video on IP" (30s) — multiple "Need Content from BBC" flags [S4 → rows 1, 8, 17, 26].
- **Login = email + name capture; biometric for returning users** [S4 → row 2; Sheet1 "Email login & existing user logging in then biometric"].
- **"Bhuvanaa's intro video (can video-embed the name using AI)"** placed after S3 / before A01 [S4 → Sheet1, Video row].
- **Family-tree concept** wanted for the family-details screen [S4 → row 11].
- **Assets split** into "In the bucket Page 1 (Liquidable)" + "Page 2 (Illiquid)" [S4 → row 12].
- Compliance screen [S4 → row 9]; AA collection of assets/liabilities/bank-statements after Paywall [S4 → row 27]; final CTAs to download FP+IP summary + Action Plan [S4 → rows 24–29].
- **Reviewer verdicts** on the React-prototype screens [S4 → Sheet1]: A01–A03 *Discard*; **A04–A06 "PERFECT"**; A07 *Discard*; **A08 Profile Card "pls give some alternate options"**; A09 sample plan — "original screen should be discarded; only the Result tab from the Sample Financial Plan React; full react downloadable as PDF"; insert a new 15–20s video + "Build my plan – Confirmation"; Feeling Qts B02/B03 *Discard*, B04 *Keep as is*; Income B05 / Expenses B06 — *"Can you give more options of capturing Income and expenses"* with three alternate orderings (Flow 1/2/3) sketched [S4 → Sheet1].
- The login journey is summarised as: **Community Login & Pledge → Avatar identification & Plan → Sample Plan → Paywall → Comprehensive FP / Comprehensive FP+IP+AP / Ongoing Serv (FP+IP+AP) / Existing HOA Services** [S4 → Sheet2].

---

### 6.5 The Boardmix master stage flow (Stages 0–6) — a third, more complete flow spec (S7)

> **Source.** A **Boardmix** master flow table (provided as a screenshot, 2026-06-26) [S7]. This is a **third** flow artefact alongside S2 (A/B/C/D screens) and S4 (numbered 1.0–29.0). It uses the **Stage 0–6 / 2A–2B** nomenclature that matches the persona spec S3, and is **more complete and more structured** than either — it introduces several stages neither other source has. ⚠️ It is *not yet reconciled* with the Figma chat prototype (§6B); **conflicts are flagged inline below.**

| Stage | Step | Screen | Key elements | User action | Notes |
|---|---|---|---|---|---|
| 0 | – | **Start Screen** | Logo, CTA button | Tap to begin | Entry point |
| 1 | 1 | **Splash Screen** | Brand logo, animation | **Wait 5 seconds** | Auto-advance |
| 1 | 2 | **Awareness Questions** | **Q1: Income/month**, profile-matching options | Select answer | Store response |
| 1 | 3 | **Profile Options** | *"Show me my full profile"* · *"Connect AA"* · *"I'll answer questions"* | Choose path | **Three entry paths** |
| 2A | 1 | Life Stage Question | Q1, response variants A–D | Select | Store as **L1** |
| 2A | 2 | Emotional State Question | Q2, response variants A–D | Select | Store as **E1** |
| 2A | 3 | Aspiration Question | Q3, response variants A–D | Select | Store as **G1** |
| 2A | 4 | **Quick Report** | Generated profile summary, *"See in-depth analysis"* CTA | View / proceed | **Rough-profile disclaimer** |
| 2B | 1 | **The Mirror Moment** | Profile Card (1 of 6), **Identification Gate** | **Yes / Only some / Not really** | Amendment 3 |
| 2B | 2 | Profile Matching | 6 profile variants display | Select best match | Profile-specific badge |
| 2B | 3 | **Amendment Fallback** | All-three-rejected fallback | Retry or skip | **Hesitation detection** |
| 3 | 1 | **Section 1** | Income & Cashflow (Q1.1–Q1.4, Q2.1–Q2.3) | Answer | Micro-feedback |
| 3 | 2 | **Section 2** | Investments & Wealth (Q3.1–Q3.3, Q4.1) | Answer | Micro-feedback |
| 3 | 3 | **Section 3** | Protection & Liabilities (Q4.2a–Q4.3) | Answer | Micro-feedback |
| 3 | 4 | **Section 4** | Goals & Timeline (Q4.4–Q4.5) | Answer | Micro-feedback |
| 3 | 5 | **AA Consent** | Consent request for account aggregation | Accept / decline | **Conditional** |
| 3 | 6 | Loading Screen | Rotating content carousel | Wait | Stage 3 closing |
| 4 | 1 | **Carousel Reveal** | 4-slide carousel (slides 1–4) | Swipe through | Moment transitions |
| 4 | 2 | **The Diagnostic** | Analysis of current financial state | View insights | **Moment 1** |
| 4 | 3 | **The Blind Spot** | Hidden financial gaps revealed | View insights | **Moment 2** |
| 4 | 4 | **What Could Be** | Visual card of potential outcomes | View card | **Moment 3** |
| 4 | 5 | **Upgrade Prompt** | Commercial options (**1, 2, 3, 4**), **"Book call"** CTA | Select tier or book | Full detail display |
| 5 | 1 | **Post-Plan Upgrade** | Option 1 → Option 2 → Option 3 paths | Upgrade selection | Trigger-based |
| 5 | 2 | **Post-Plan Mirror Moment** | Re-reflection flow | Re-assess profile | Amendment 6 |
| 6 | 1 | Home Screen | Dashboard by tier, home-screen content | Navigate | Returning user |
| 6 | 2 | **Living Dashboard** | **5 tracking dimensions**, Chart Placement Map | View / interact | Real-time updates |
| 6 | 3 | **Action Completion** | "How It Works" section, task tracking | Complete actions | Progress tracking |
| 6 | 4 | **Nudge System** | **3-layer nudge system**, specific upgrade nudges | Engage / dismiss | Contextual prompts |

**What this adds that no other source has:**
- **Stage 1.2 "Awareness Questions — Q1: Income/month"**: an **income band captured up-front**, before persona matching — likely the input to the *Quick Report*. Not in S2/S4/§6B.
- **Stage 1.3 "Profile Options" = three entry paths** (*Show me my full profile / Connect AA / I'll answer questions*): the **AA-vs-manual fork surfaced at the very start**, not mid-flow. This is the cleanest statement of the AA-decision point (cf. §6.2 "AA decision + consent").
- **Quick Report (2A.4)** — a rough profile summary *with disclaimer* and a "See in-depth analysis" CTA, sitting **between Q3 and the Mirror Moment**.
- **The Mirror Moment + Identification Gate** (Yes / Only some / Not really) and **Amendment Fallback** (hesitation detection) — formalises S3's *"show me a different profile"* (§5.4 rule 2) into explicit gated screens.
- **Stage 3 = four named Sections with real question numbering (Q1.1–Q4.5)** — a cleaner taxonomy than S2's B/C/D split: Section 1 Income & Cashflow · Section 2 Investments & Wealth · Section 3 Protection & Liabilities · Section 4 Goals & Timeline.
- **Stage 4 = Carousel Reveal (4 slides) + 3 "Moments"** — Diagnostic / Blind Spot / What Could Be. "What Could Be" is exactly S3's Stage-4 aspiration-anchored moment (§5.4 rule 5); Diagnostic + Blind Spot map to Clarity insights + Blindspots (B12–B14).
- **Upgrade Prompt = 4 commercial options + "Book call"** — confirms **all four tiers** appear at the paywall, with the advisory-booking CTA inline.
- **Stage 5 = Post-Plan Upgrade (1→2→3) + Post-Plan Mirror Moment (re-reflection)** — a re-assessment loop *after* the plan, absent elsewhere.
- **Stage 6 = Living Dashboard (5 tracking dimensions, "Chart Placement Map") + "How It Works" action tracking + a 3-layer Nudge System** — more specific than S2's Flow-F dashboards (§6.1).
- **"Amendment 3" / "Amendment 6" labels** imply a **versioned amendment system** — consistent with the **"V3 / V5" flow versions** seen on Boardmix. ⚠️ Confirm which amendment/version is canonical.

**⚠️ Conflicts to resolve (this table vs other sources):**
- **Splash duration: "Wait 5 seconds" (S7)** vs **"≤2s, no spinner" (S2 → G01, §6.1).** Direct conflict.
- **Onboarding order:** S7 puts an **income question at Stage 1.2 (before persona)**; the **Figma prototype (§6B) captures income much later (Step 17, after goals)**; S2 puts income inside Step-1 "Clarity" (B05). **Three different placements of income capture** — pick one.
- **AA placement:** S7 offers **AA as an entry path (Stage 1.3)** *and* as a conditional consent inside Stage 3.5; S4/S5 say **AA is post-paywall** (only for Paywall 2/3). Reconcile *when* AA is offered.
- **Question scheme:** S7's **Q1.1–Q4.5 section numbering** does not map 1:1 to S2's field codes (I1–I4, INV1–INV7a, YB1–4, YL1–3, etc.). A crosswalk is needed before build.
- **Persona display:** S7 has **both** a "Mirror Moment (1 of 6)" *and* a separate "Profile Matching — 6 variants display / select best match"; S3 says the matcher picks **one** card server-side (§5.1) and the user may request alternates. Clarify whether the user actively *picks* from 6 or is *shown* one.

> **Status:** treat **§6 (S2) as the field-level data spec**, **§6B (Figma) as the as-built chat presentation**, and **§6.5 (this Boardmix table) as the most complete *stage skeleton*** — but the three disagree on ordering/splash/AA-timing as flagged. **A single reconciled master flow does not yet exist across the sources** — that is the key open item for the flow itself.

---

## 6B. Visual Flow Breakdown — The Figma Prototype, Step by Step (S9)

> **What this is.** The 19 screenshots in `UX_images/` are a single linear scroll-through of the **conversational prototype** of the cold-funnel journey: **splash → welcome → login → 3 onboarding questions → persona reveal → compliance → data capture (family, assets, liabilities, goals, life events, income, expenses) → engine-generated insight cards → paywall.** Each step below carries: the UI image · *what the step does* · *what the user does here* · *external API* · *input* · *output* · *other details*. Field codes (L1, E1, INV1…) cross-reference §6.1; flow letters (A04, B12…) cross-reference §6.
>
> **As-built ordering note.** The chat reorders the §6 spec: **income & expenses are captured *after* goals + life events and *after* the first Clarity insight** — not inside Step-1 "Clarity" as in §6.1. The order below is the *actual prototype order*. The **"dock"** (floating star, bottom-right) increments a badge (0→1→2→4→5…) as each engine output is saved — this is the §2A "floating-star repository," now confirmed.
>
> All screens are **cold-funnel / FREE tier** (top bar shows "Guest" then "FREE"); data here is **manual entry — no AA in this flow** (AA is a Tier-2 path, §6.2/§6.3). The named demo user is **Rahul, 41** (spouse Aarti, 35), matched to the **Rajesh / Self-Neglect** persona (§5.2).

---

### Step 1 — Splash
![Step 1 — Splash](UX_images/Screenshot%20from%202026-06-25%2023-53-14.png)
- **Does:** App-open splash. Brand mark *"yesly — Say Yes to Life!"*; reassurance copy *"Life only gets clearer from here. Almost there. One moment."* Maps to **G01 splash** (§6, "≤2s, no spinner").
- **User does:** Tap the **→** arrow to begin (or auto-advances).
- **External API:** None user-facing. App init / AWS Cognito session check (returning-user detection).
- **Input:** None.
- **Output:** None (transition into chat).
- **Other:** No loading spinner by design; ≤2s budget [S2 → G01].

### Step 2 — Welcome + Bhuvana intro video (Guest)
![Step 2 — Welcome chat + intro video](UX_images/Screenshot%20from%202026-06-25%2023-53-18.png)
- **Does:** Opens the chat. yesly (avatar "y.") greets: *"Hey – I'm Bhuvana. Two minutes before we start. Watch this, then we'll see where you actually are."* Embeds the **Welcome Video** card. Maps to **Video 1 / Bhuvana intro** [S4 rows 1 & "Bhuvanaa's intro video"] in chat form.
- **User does:** Watch, then tap **"Skip the video"** or **"I've watched it – continue."**
- **External API:** Video hosting/CDN (S3 + CloudFront, or embedded player); Mixpanel (video start/complete events).
- **Input:** Video-watched vs skipped (boolean).
- **Output:** None (advances). Dock badge = **0**.
- **Other:** Top bar = **"Guest"** (pre-login). Dock (floating star) already visible bottom-right. *"Bhuvanaa's intro video can AI-embed the user's name"* per S4.

### Step 3 — Login
![Step 3 — Login](UX_images/Screenshot%20from%202026-06-25%2023-53-21.png)
- **Does:** Identity capture before any financial data: *"Before any numbers, let me know who you are. Just login – nothing more."* Maps to **login** [S4 row 2].
- **User does:** Tap **"Sign in with Google"** (social login).
- **External API:** **AWS Cognito** (social/OAuth — Google). Returning users use biometric [S4 Sheet1].
- **Input:** Google OAuth identity (email, name, avatar).
- **Output:** Authenticated session; USER + PROFILE bootstrapped (§8.1).
- **Other:** S4 also specifies **email + name capture** and biometric for returning users. Tier badge flips **Guest → FREE** at/after this point.

### Step 4 — Q1 Life stage + Q2 Emotional state
![Step 4 — Q1 + Q2](UX_images/Screenshot%20from%202026-06-25%2023-53-24.png)
- **Does:** The first two of three persona-matching questions (§5). **Q1** *"Where are you in life right now?"* and **Q2** *"How do you feel about your finances today?"* Maps to **A04 (L1)** and **A05 (E1)**.
- **User does:** Single-select a chip per question. (Demo: Q1 = *"Earning well, not sure where it's going"*; Q2 = *"A bit worried about what's next."*) yesly acknowledges between turns (*"Got it…", "That makes sense."*).
- **External API:** None — **session-only** values per S3 (not persisted). Mixpanel events.
- **Input:** Q1 → **L1** (life stage); Q2 → **E1** (emotional state).
- **Output:** None yet (fed to the profile matcher at Q3).
- **Other:** ⚠️ **Option wording differs from §6.1's A04/A05** (e.g. "Earning well, not sure where it's going" vs spec's "Earning well but uncertain"). Treat the chat copy as the newer wording; the *scoring buckets* (§5.3) are unchanged. Top bar now **"FREE"**.

### Step 5 — Q3 Aspiration
![Step 5 — Q3](UX_images/Screenshot%20from%202026-06-25%2023-53-26.png)
- **Does:** Third matching question *"Last one – five years from now, what does good look like?"* Maps to **A06 (G1)**.
- **User does:** Single-select (options: *Financial independence / Security for my family / Freedom to make bold life choices / …*).
- **External API:** None (session-only). Mixpanel.
- **Input:** Q3 → **G1** (aspiration).
- **Output:** Completes the L1+E1+G1 triple → triggers `matchProfile()` (§5.5).
- **Other:** G1 is stored separately and re-used for Stage-4 "What Could Be" anchoring (§5.4 rule 5).

### Step 6 — Persona reveal (Rajesh) · OUTPUT 1
![Step 6 — Persona card](UX_images/Screenshot%20from%202026-06-25%2023-53-29.png)
- **Does:** *"Analysing your profile… Here's someone who's on the same journey."* Renders the matched **persona card: Rajesh · 41 · Pune · Senior Manager · ₹2.1L/month**, with his worries (*"I'm paying for everything – but am I actually building anything for myself?"*) and a **before/after teaser**: Term cover ₹75L · Home loan ₹2Cr · Monthly SIP ₹25,000 · **Current Path ₹1.67Cr (money runs out age 68) vs With Plan ₹7.79Cr (age 94).** Maps to **A07/A08** + the sample-plan teaser. This is **Rajesh = Self-Neglect**, also the default fallback profile (§5.2).
- **User does:** Read; advance (later: "Build my picture").
- **External API:** None — cards are **pre-rendered static** at app init; matcher returns a profile-ID string, no dynamic generation (§5.5). Numbers are canned sample-plan data.
- **Input:** Profile-ID from matcher (consumes L1/E1/G1).
- **Output:** **"Output 1 of many · Saved to your dock at the bottom right."** Dock badge → **1**.
- **Other:** Confirms the **<100ms, no-loading-state** persona reveal (§5.1) and the BBC-rendered "6 avataars" sample plans (§5 cross-link). A/B target: tap-through >85% (§5.5).

### Step 7 — "Build my picture" + compliance pre-roll + transition video
![Step 7 — Build my picture + transition video](UX_images/Screenshot%20from%202026-06-25%2023-53-40.png)
- **Does:** User commits with **"Build my picture."** yesly: *"Before any more questions – quick compliance moment. SEBI requires this. I'll keep it short,"* then embeds the **"Transition Video from Sample Plan to Personal Plan"** (Video 2, [S4 row 8, "Need content from BBC"]).
- **User does:** Tap **"Build my picture"**; watch the transition video.
- **External API:** Video hosting/CDN. Mixpanel.
- **Input:** Intent confirmation (boolean).
- **Output:** None. Dock badge = **1**.
- **Other:** User avatar now appears in the top bar (persona/avatar applied). This is the seam between the free *sample* and the user's *own* plan.

### Step 8 — SEBI compliance disclosure + consent
![Step 8 — SEBI disclosure + consent](UX_images/Screenshot%20from%202026-06-25%2023-53-43.png)
- **Does:** Mandatory RIA disclosure card: **"House of Alpha is SEBI-Registered (INA000018364)"** — *We give advice, we don't sell products or earn commissions · Your data stays with us, used only to build your plan, never sold · Past performance isn't a guarantee, projections are directional · You can revoke consent or close your account anytime.* Then *"This is your starting point…"* Maps to **compliance screen** [S4 row 9] + §9 SEBI posture.
- **User does:** Tap **"I understand and agree to proceed with my plan,"** then **"Let's begin with what you feel."**
- **External API:** Internal API → writes **CONSENT_RECORD** + **immutable AUDIT_EVENT** (§8.1, §9 — every advice-bearing consent is logged, append-only).
- **Input:** Explicit consent acceptance (tap + timestamp).
- **Output:** Logged consent/audit record (no user-facing artifact).
- **Other:** Surfaces the actual **SEBI RIA reg-no INA000018364** — first time it appears in any source. Disclosures must be *shown and logged* (§9).

### Step 9 — Step-1 "Clarity": money-journey + family tree
![Step 9 — Money journey + family tree](UX_images/Screenshot%20from%202026-06-25%2023-53-45.png)
- **Does:** Emotional opener *"How has your money journey been so far?"* then *"Money never sits alone – there's usually people around it"* → the **family-tree** widget (FATHER/MOTHER/YOU/SPOUSE/CHILD 1/CHILD 2, each "Add"). Maps to **B01–B04 emotional sub-section** + **B10 dependants** + the **family-tree concept** [S4 row 11].
- **User does:** Pick a money-journey chip (demo: *"Some hits, some misses"*); tap **Add** on family nodes to add members.
- **External API:** None — manual entry → RDS.
- **Input:** Money-journey sentiment (→ **E-series** emotional fields); household members (self, spouse, children, parents) → **DEPENDENT** + USER role_in_household (§8.1).
- **Output:** None yet.
- **Other:** "YOU = Rahul, 41." Household-as-root data model (§8.1) is visible here — finances will hang off the household.

### Step 10 — Spouse added → liquid assets begins
![Step 10 — Spouse added + liquid assets](UX_images/Screenshot%20from%202026-06-25%2023-53-47.png)
- **Does:** Confirms *"Spouse Details Added"* (family tree now YOU Rahul 41 + SPOUSE Aarti 35), warm ack *"Lovely family. I'll make sure their futures are part of every number we plan,"* then opens **liquid assets**: *"What do your liquid assets look like today?"* (Mutual Funds ₹50L…). Maps to **B06 Investments (INV1–INV3)** / S4's *"In the bucket Page 1 (Liquidable)"* [S4 row 12].
- **User does:** Confirm family (**Continue**); begin entering liquid-asset amounts.
- **External API:** None (manual). *(In the Tier-2 AA path these auto-fill from CAMS/KFin RTA — §6.2.)*
- **Input:** Dependent details (spouse name/age); liquid-asset values.
- **Output:** None yet.
- **Other:** —

### Step 11 — Liquid-assets summary + illiquid assets
![Step 11 — Liquid summary + illiquid](UX_images/Screenshot%20from%202026-06-25%2023-53-50.png)
- **Does:** Summarises liquid assets (**Mutual Funds ₹50L · Savings+FDs ₹80L · Direct Stocks ₹50L · Gold (SGB+Digital) ₹50L**), then asks **illiquid assets** *"What do your illiquid assets look like today?"* (**EPF/PF, NPS, PPF/long-term deposits, ULIP** — sliders). Maps to **B06/B07 (INV1–INV7a)** / S4's *"Page 2 (Illiquid)"* [S4 row 12].
- **User does:** **Add more / Continue / Modify** on liquid; set illiquid sliders.
- **External API:** None (manual). *(AA path: MF←CAMS/KFin, EPF←EPFO, NPS←CRA, FD←Bank, ULIP←Insurance AA; Gold & Home not in AA — §6.2.)*
- **Input:** **INV1** MF, **INV2** EPF, **INV3** NPS, **INV4** FD+savings, **INV5** Gold, **INV6** ULIP, **INV7/7a** PPF/home.
- **Output:** None yet.
- **Other:** Range-band sliders (Under 5L … 1Cr+) match §6.1 B06/B07.

### Step 12 — Liabilities + "What you own / What you owe"
![Step 12 — Liabilities + net-worth blocks](UX_images/Screenshot%20from%202026-06-25%2023-53-56.png)
- **Does:** *"Is there anything you still owe?"* — **HOME LOAN-OUTSTANDING, CAR/VEHICLE LOAN, PERSONAL LOAN** sliders. Then a running **"What you own ₹52.0L"** (Savings & cash ₹4.2L · Mutual funds ₹8.5L · EPF+PPF ₹11.3L · Home ₹28.0L) and **"What you owe"** (Home loan ₹29.6L · Car loan…). Maps to **B08 Liabilities (L1–L4)** + a pre-cursor to **B12**.
- **User does:** Set liability sliders; **Continue.**
- **External API:** None (manual). *(AA path: loan accounts + EMI from Bank AA — §6.2.)*
- **Input:** **L1** home loan, **L2** car loan, **L3** personal loan (+EMI). (Also the "No liabilities – confirm" path **L4** exists in spec.)
- **Output:** Aggregated own/owe blocks (precursor to net-worth output).
- **Other:** "Neutral framing, never shaming" mandate (§6.1 B08).

### Step 13 — Clarity Insight 1 of 3 (Net Worth) · OUTPUT 2
![Step 13 — Clarity Insight 1 of 3](UX_images/Screenshot%20from%202026-06-25%2023-53-58.png)
- **Does:** *"Hang on – I'm pulling your picture together…"* (progress bar) → **CLARITY INSIGHT 1 OF 3 — "Here's what life looks like today": Total net worth ₹1.54 Cr · Total liabilities ₹62L · Realisable ₹54L · Locked ₹12L · Property ₹1.5 Cr.** Footer: *"Clarity 1 of 3 · 3 Pages · ⬇ Download PDF."* Then *"Two more reveals coming — projections, then the blindspots."* Maps to **B12 net worth + WAR** → **B13 projections** → **B14 blindspots**.
- **User does:** Read; tap **"Show me what's next"** (and may **Download PDF**).
- **External API:** **HoA Planning Engine / Container 4 (NWGC-derived)** computes net worth, realisable/locked/property split, WAR (§7.2, §10). **PDF generation** → S3.
- **Input:** All captured assets + liabilities (Steps 10–12).
- **Output:** Net-worth insight card + **downloadable PDF**. **"Output 2 of many · Saved to your dock."** Dock badge → **2**. (Projections = output 3, blindspots = output 4 follow — see Step 14 badge.)
- **Other:** This is the **headline → app-viewable output → PDF** sequence from the §2A pivot, live. NWGC sheets `Networth`/`WAR` are the source maths (§10).

### Step 14 — Free→Paid transition video + Essential Plan card (₹2,999)
![Step 14 — Free→Paid video + Essential Plan](UX_images/Screenshot%20from%202026-06-25%2023-54-00.png)
- **Does:** *"Now you know where things really stand… Most people avoid this step for years,"* embeds the **"Transition Video from Free to Paid User"** (Video 3 paywall explainer [S4 row 17]), then surfaces the **"Essential Plan ₹2,..."** card (*"Three biggest blindspots, fixed"*). First **paywall nudge** — maps to **D10 upgrade prompt** ("not a paywall, a graduation").
- **User does:** Watch; consider the Essential Plan.
- **External API:** Video hosting/CDN. (Razorpay invoked only on purchase.)
- **Input:** None (informational).
- **Output:** Plan teaser. Dock badge → **4** (projections + blindspots were saved as outputs 3 & 4 between Steps 13–14).
- **Other:** **Essential Plan = ₹2,999 ≈ Tier 1 Financial Plan** (resolves the §6.1 pricing flag). The journey then *continues collecting data* (goals/events/income) before the full paywall at Step 19.

### Step 15 — Step-2 "Yes Life": goals
![Step 15 — Goals picker + goal detail](UX_images/Screenshot%20from%202026-06-25%2023-54-02.png)
- **Does:** *"What are you saying yes to?"* — **goal chips** (Travel, Education, Sabbatical, Career change, Family milestone, A Home, A side thing, Retire on my terms…), then *"Let's make it real"* per-goal detail: **When (1yr–10+yr)** + **rough budget (<1L–25L+)** sliders. Maps to **C02 (YL1)** + **C03 goal-conversation (YL1a–c)**.
- **User does:** Multi-select goals (demo: *Education* + *A Home*); set timeline + budget per goal.
- **External API:** None — RDS **GOAL** entities (household-owned, §8.1).
- **Input:** **YL1** goal types; **YL1a** timeline, **YL1b** budget, **YL1c** who-it's-for.
- **Output:** None yet (feeds Full Picture + plan).
- **Other:** Framed as *"your Yes List,"* not a goals form (§6.1 Flow C).

### Step 16 — Life events (+ certainty)
![Step 16 — Life events](UX_images/Screenshot%20from%202026-06-25%2023-54-03.png)
- **Does:** *"Add things you don't want to miss out on"* — **life-event chips** (Child's Wedding, Child's Education, Parents Care, Home Renovation, Sibling Support, Medical Buffer…), then per-event *"When does it hit?"* (years-from-now) + *"How much?"* (₹, demo ₹80L) + *"How sure are you this happens?"* Maps to **C06 non-discretionary life events (YL3)**; sources NWGC `Key Life Events` (§10).
- **User does:** Pick events (demo: *Child's Education*); set timing, cost, certainty.
- **External API:** None — RDS **LIFE_EVENT** (household-owned).
- **Input:** **YL3** event type, age/years-to-event, cost, probability/certainty.
- **Output:** None yet.
- **Other:** Certainty weighting is new vs §6.1 (which had age+cost only) — capture it in the data model.

### Step 17 — Income + expenses
![Step 17 — Income + expenses](UX_images/Screenshot%20from%202026-06-25%2023-54-21.png)
- **Does:** *"What is your monthly income? Exact numbers help, but ranges are fine"* — **Monthly take-home, Spouse/partner income, Rental income, Freelance/business income** sliders; then *"What are your monthly expenses?"* — **Rent, Food, …** outflow sliders. Maps to **B05 Income (I1–I4)** + **D02–D05 Budget (YB1–YB4)**.
- **User does:** Set income sliders → **Continue**; set expense sliders.
- **External API:** None (manual). *(AA path: income from 12-mo salary credits, expenses from txn categorisation — §6.2/§6.3.)*
- **Input:** **I1** take-home, **I2** spouse, **I3** rental, **I4** freelance/business; **YB1** fixed, **YB2** variable, **YB3** guilt-free, **YB4** SIPs.
- **Output:** None yet (feeds surplus calc).
- **Other:** ⚠️ **As-built reorder:** income/expenses come here (after goals/events + first insight), *not* inside Step-1 Clarity as §6.1 places them.

### Step 18 — Full Picture (pre-plan) · OUTPUT 5
![Step 18 — Full Picture summary](UX_images/Screenshot%20from%202026-06-25%2023-54-23.png)
- **Does:** *"Hang on – I'm pulling your full picture together…"* → **"YOUR FULL PICTURE · PRE-PLAN — Everything in one view": Income ₹2.10L · Expenses ₹1.96L · Surplus +₹15K · Total assets ₹54L · Total liabilities ₹12L · Net worth ₹1.5 Cr · Goal timeline: Home 2028, Aanya college 2038.** Footer *"Full Picture · 5 Pages · ⬇ Download PDF."* Then *"Now – let's activate this. Two plan options to choose from."* Maps to **D06 surplus calc** + **D07/D08** consolidation.
- **User does:** Read; tap **"Show me the plan options"** (may **Download PDF**).
- **External API:** **HoA Planning Engine (Container 4 / NWGC)** computes surplus, full picture, goal timeline (`Budget`, `Cashflow Working`, `YoY Assets` — §10). PDF gen → S3.
- **Input:** Everything captured (assets, liabilities, goals, life events, income, expenses).
- **Output:** Full-picture card + **downloadable PDF**. **"Output 5 of many · Saved to your dock."**
- **Other:** Surplus shown **in green / gold** when positive (§6.1 D06). This is the value-proof immediately before the paywall.

### Step 19 — Paywall: Growth Plan (₹4,999) vs Free
![Step 19 — Growth Plan paywall](UX_images/Screenshot%20from%202026-06-25%2023-54-25.png)
- **Does:** **"Growth Plan ₹4,999"** — *✓ Everything in essential · ✓ Goal-based financial roadmap · ✓ SIP + tax-optimisation plan*; trust strip *"256-bit secure · SEBI-compliant · Cancel anytime."* Maps to the **paywall / upgrade purchase** (G05/G06; Tier ladder §3).
- **User does:** Tap **"Start Growth – ₹4,999"** (purchase) or **"Continue with free plan."**
- **External API:** **Razorpay** (one-time/subscription payment, §7.1). **KYC vendor** (Digio/IDfy) triggers at Tier-2 entry (§9 — *open: SEBI may require at Tier 1*). On success → plan delivery (PDF + in-app + email via SES).
- **Input:** Plan selection + payment.
- **Output:** Purchased tier → unlocks the corresponding home/dashboard (F02/F03, §6.1) and the deeper plan deliverables.
- **Other:** Two-card ladder confirmed: **Essential ₹2,999 (Tier 1)** [Step 14] vs **Growth ₹4,999 (Tier 2)** [here]. Tier-3 subscription price still unconfirmed (§11.2 open decision 6).

---

## Steps 20+ — The Rest of the Journey (Post-Paywall, **spec-sourced — no Figma yet**)

> 🖼️ **No Figma screens exist beyond Step 19** — the client's prototype stops at the paywall. The steps below are reconstructed **from the source docs** (S1 architecture charts, S2 Flow E/F/G, S4/S5 flow sheets, S6 engine) in the same per-step format, so the breakdown is complete end-to-end. Each is tagged with its source. **Ordering note (S4 "New User" sheet + S5 Integrations):** AA and KYC happen **after** the paywall, and **only on the Investment-Plan (Option-2) / Subscription (Option-3) paths** — the Financial-Plan (Option-1) path runs entirely on the manually-entered data and goes straight to delivery.

### Step 20 — Payment / purchase (Razorpay)
- **Does:** Completes the tier purchase from Step 19. Two purchase flows: **G05 (Option 1 — Financial Plan ₹2,999)** and **G06 (Option 2 — Growth/Investment Plan ₹4,999)**; both use "graduation" framing — *"Your plan is being built"* [S2 → G05/G06].
- **User does:** Pay; on success, branch by tier.
- **External API:** **Razorpay** (one-time; subscription mandate for Tier 3) [S1 §7.1]. On success → INVOICE + SUBSCRIPTION records (§8.1).
- **Input:** Plan selection + payment instrument (tokenised at gateway — never touches Yesly servers, §1).
- **Output:** Purchased tier unlocked; conversion event logged. **Option-1 → jump to Step 27 (FP + Action Plan delivery).** Option-2/3 → continue to KYC.
- **Other:** Refund window is an open decision (7/14/30 days, §11.2 #10) and governs the downgrade paths (§3 churn logic).

### Step 21 — KYC (Layer 1 — HoA RIA) · *Tier 2+ only*
- **Does:** Identity verification + advisory-agreement signing, triggered when the user buys the Investment Plan [S1 → Chart 11]. Sequence: **KYC-required screen → PAN entry → Aadhaar OTP / DigiLocker → bank-account verification (penny-drop) → selfie / liveness → Advisory Agreement display (HoA RIA reg, fee structure, scope of advice, conflict-of-interest, grievance redressal) → accept.**
- **User does:** Enter PAN, complete Aadhaar OTP, verify bank, take selfie, accept the advisory agreement.
- **External API:** **KYC vendor — Digio / IDfy / Hyperverge** (validates PAN with IT dept, Aadhaar OTP, name+DOB match, sanctions screening) [S1 §7.1, Chart 11]. DigiLocker for Aadhaar.
- **Input:** PAN, Aadhaar (OTP/DigiLocker), bank account, selfie, agreement acceptance.
- **Output:** `user.kyc_status = verified/rejected`; **KYC docs stored encrypted in S3, restricted ACL** (CRITICAL PII, §8.2); signed advisory agreement stored; immutable AUDIT_EVENT. On reject → reason + support; **3 failed attempts → grievance escalation.**
- **Other:** ⚠️ **Open: trigger tier.** Chart 11 assumes Tier 2; SEBI may require it at **Tier 1** if the Financial Plan counts as "advice" — compliance to adjudicate (§9, §11.2 #5). **Layer 2 (broker KYC)** is separate and happens at first execution (Step 30) — Yesly is blind to it in v1. Re-KYC cron-nudged 30 days before expiry (low-risk 10y / med 8y / high 2y).

### Step 22 — Account-Aggregator consent micro-flow (via Setu) · *Tier 2+ only*
- **Does:** The 7-step consent journey to pull real financial data, replacing the manual entry of Steps 9–17. *"Connect your accounts — we'll pull what we can (investments, bank data, insurance). We'll ask you about the rest"* [S2 → AA Process; S1 → Chart 05]. **~95s total.** Maps to screens **AA01–AA07**.
- **User does:** (1) FIP selector — pick from up to 6 pre-sent FIPs (10s); (2) enter mobile number (10s); (3) AA-handle discovery — Setu routes to best AA (15–30s); (4) OTP to link accounts (15s); (5) consent review — FIPs, data types, duration, purpose, frequency (20s); (6) single consent-approval OTP (10s); (7) success + background fetch (15–30s).
- **External API:** **Account Aggregator — Setu / OneMoney / CAMSFinServ / Anumati** (consent + data fetch via `POST /aa/consent/initiate` → vendor consent UI → webhook → `POST /fi-request` → `GET /fi-data` → backend decrypts with private key) [S1 → Chart 05].
- **Input:** Mobile number, FIP selection, two OTPs (link + approve), consent parameters.
- **Output:** **ACTIVE consent record** (CONSENT_RECORD, §8.1) + encrypted FI data bundle → normalised to internal schema → recompute net worth → trigger plan refresh. Scheduled refresh per `consent.frequency` (the Tier-3 live-data engine).
- **Other:** Per-field AA coverage (§6.2): MF←CAMS/KFin RTA (exact), EPF←EPFO, NPS←CRA, FD/loans←Bank AA, term/health/ULIP←Insurance AA; **Gold, home value, dependants = AA can't fill → user enters.** Real-world limits (§6.3): FD support ~40% of banks, **joint accounts unsupported by any bank**, LIC on only 1 AA (OneMoney), NRE/NRO not discoverable. *"AA should be done only for users who want exact data."* Revocation/expiry handled in F09 (consent management).

### Step 23 — Engine run → outputs generated (HoA planning APIs)
- **Does:** Once all data is collected (manual and/or AA), the **HoA Planning Engine (Container 4, NWGC-derived)** computes every plan output — *"For all the outputs — after collecting all the data"* [S5 → Integrations]. This is the same engine that produced the inline Clarity/Full-Picture cards (Steps 13/18); here it generates the **full Investment Plan** against actual holdings.
- **User does:** Wait (headline copy: *"We are processing your data and preparing your recommendations"* — §2A pivot).
- **External API:** **HoA planning APIs** (internal, partner builds only the wrapper — §7.2 critical scope split). PDF generation → S3.
- **Input:** Complete household financial picture (assets, liabilities, income, expenses, goals, life events, risk profile) + the active **Assumption Set** (e.g. `v2026.Q2`).
- **Output:** Plan objects locked to `assumption_set_id` (§8.1); see **§6C for the full field-level output schema.**
- **Other:** Plans regenerate only when an admin activates a new assumption set (Admin Surface 5, §8.3).

### Step 24 — Video 4: "Harish's Video on IP"
- **Does:** A 30-second **non-skippable** explainer on the Investment Plan, fronted by Harish, shown on the Option-2 path before the IP is revealed [S4 → row 25, *"Need content from BBC"*].
- **User does:** Watch.
- **External API:** Video hosting/CDN; Mixpanel.
- **Input/Output:** None.
- **Other:** One of four BBC-produced videos (Welcome / Sample→Personal / Free→Paid / IP) — content still pending from BBC (§2, §6.4).

### Step 25 — Investment Plan delivery · *Tier 2*
- **Does:** Reveals the **portfolio-specific Investment Plan** against actual holdings: current vs target allocation, gap analysis, **action report (buy/sell/hold)**, risk-adjusted suitability per holding [S1 → Chart 03; see §6C]. Preview is watermarked; full plan delivered as **PDF + in-app view**.
- **User does:** Review; download PDF; proceed to Action Plan / Hell Yes / subscription offer.
- **External API:** PDF gen (S3); email copy (Amazon SES).
- **Input:** Engine output (Step 23).
- **Output:** Investment Plan PDF + in-app render; logged as an advice-bearing AUDIT_EVENT (§9).
- **Other:** Hard-gated on portfolio data (AA/CAS/statement/manual) — the gate is **portfolio data + risk profile**, not prior-tier purchase (§3).

### Step 26 — Step-4 "Hell Yes!" decision tool (Flow E) · *Tier 2+; teaser to Free/Opt-1*
- **Does:** Two-path life-decision simulator [S2 → Flow E]. **E01** decision picker (5 cards: Buy a home / Career change / Sabbatical / Second child / Early retirement / Custom, **HY1**) → **E02** Path-A inputs (**HY2a**) → **E03** Path-B inputs with Path-A summary (**HY2b**) → **E04** scenario summary → **E05** full-scroll life walk-through (sticky strip, TODAY + 8 age stops, two columns) → **E06** "Hell Yes?" decision (Path A / Path B / Not yet / Talk to advisor) → **E07** decision recorded + share + advisor booking. **E08** is the *locked teaser* shown to Free/Opt-1 (tool description + **3 real WhatsApp conversation excerpts** + upgrade CTA — *"excerpts do the selling"*).
- **User does:** Pick a decision, enter both paths' assumptions, walk the 8-stop projection, choose a path.
- **External API:** HoA Planning Engine — the **NWGC `NWGC - AS IS` vs `NWGC - SCENARIOS`** two-scenario projection (§6C/§10).
- **Input:** **HY1** decision type; **HY2a/HY2b** per-path inputs (e.g. salary/growth/retirement-age vs home-price/down-payment/EMI-tenor/rate).
- **Output:** Side-by-side projection (net worth, "money runs out at age", three key numbers per path) + recorded decision; optional advisor booking (→ Step 31).
- **Other:** Neutral framing; E07 "celebration moment" adds to tribe feed.

### Step 27 — Final CTAs: plan & Action-Plan downloads
- **Does:** Closes the purchase journey with the deliverable downloads [S4 → rows 23–28]. **Option-1:** *"download the FP & Action Plan."* **Option-2:** after AA → *"Download FP+IP (Summary in the app)"* → **"Action Plan."**
- **User does:** Download the FP / FP+IP summary and the Action Plan.
- **External API:** PDF gen (S3); SES email.
- **Input:** Engine outputs.
- **Output:** **Financial Plan PDF**, **Investment Plan summary**, **Action Plan** (the prioritised "what to do next" — realised in-app as the **"Yes Pathway" checklist**, F05) [§6C].
- **Other:** Delivery is always **PDF + in-app + email** [S1 → Chart 03]; outputs also land in the **dock** repository (§2A pivot).

### Step 28 — Live home / dashboards (Flow F) · *per tier*
- **Does:** The persistent app after onboarding — one home per tier [S2 → Flow F]. **F01 Free** (snapshot recap, blindspot card, plan teaser, upgrade CTA, events, Hell-Yes teaser, tribe feed) · **F02 Financial-Plan** (most-urgent action with red border, goal-progress bars, "Yes Pathway" checklist, Hell-Yes locked) · **F03 Plan+Advisory** (AA data-freshness timestamp, three metric blocks, WAR progress bar, holdings tagged retain/exit/new-SIP, SIP-mandate status, rebalancing checklist) · **F04 Full-Service** (advisor card + next call, circular portfolio-health score, advisor action timeline). Plus **F05** My Plan / Yes Pathway (This Week / This Month / Month 5+), **F06** live portfolio via AA (holdings, XIRR vs benchmark, rebalancing), **F07** tribe feed, **F08** community events/RSVP, **F09** profile & settings incl. **AA consent management.**
- **User does:** Track actions, view portfolio, manage consents/notifications.
- **External API:** AA (scheduled refresh) for F03/F06; engine for health score/WAR/drift.
- **Input:** Tier; live AA data.
- **Output:** Dashboard render; check-off actions on F05.
- **Other:** Dashboard **density rises with tier** (Free=low → Full-Service=curated) with tier badge colours (§3 table).

### Step 29 — Subscribed monthly lifecycle (Tier 3) · *the recurring loop*
- **Does:** A repeating 4-week loop [S1 → Chart 04]. **Week 1 — Portfolio health check:** refresh AA → compute drift vs target → if drift>threshold → generate rebalancing rec → **compliance gate** → notify → rebalancing screen → accept → execute. **Week 2 — New idea:** HoA publishes curated idea → compliance gate → suitability-match per user → notify → idea detail → execute. **Week 3 — Behavioural check-in:** *"how's the journey"* → if flagged → escalate to advisor. **Week 4 — Plan health summary.** **Billing (any week):** Razorpay monthly charge → on fail, dunning days 1/3/5/7 → if unrecovered, auto-downgrade to Tier 2.
- **User does:** Respond to notifications; accept/decline rebalances & ideas; check in.
- **External API:** AA (refresh), HoA engine (drift/match), Razorpay (billing), Expo Push / SES / WhatsApp BSP (notifications), broker gateways (execution).
- **Input:** Live holdings; user accept/decline; check-in responses.
- **Output:** Rebalancing recs, idea matches, monthly summaries, billing state; escalations to Tier 4.
- **Other:** **The compliance gate before any user-facing recommendation is non-negotiable** (it's also the v2 control surface). Recommendation pipeline (Chart 06): author → compliance review (APPROVE/REJECT/CHANGES) → suitability match → notify → decision; separation of duties (author≠reviewer≠matcher) is SEBI-relevant. Notification fatigue managed via frequency capping / comms prefs (v1 scope).

### Step 30 — Order execution (smallcase / BSE StAR)
- **Does:** Places the trade when a user accepts a rebalance or idea [S1 → Chart 07]. **v1 (deep-link):** compliance gate re-verifies suitability at click time → construct deep-link (smallcase basket / BSE StAR MF) → user completes **outside Yesly** → returns → app asks *"Did you complete the trade?"* (self-reported; portfolio confirmed only at next AA refresh — **up to 7 days stale**). **v1.5 (in-app):** compliance gate → market-hours check → broker-linked check (OAuth-style linking, stores only "linked" status) → place order via smallcase Gateway / BSE StAR MF API → broker webhook (EXECUTED/PARTIAL/REJECTED) → update internal **order ledger** → daily **reconciliation** (broker vs AA holdings).
- **User does:** Confirm/execute; (v1) self-report completion.
- **External API:** **smallcase Gateway + BSE StAR MF** (Yesly is **not** the broker of record); broker runs its own Layer-2 KYC.
- **Input:** Order intent (user, idea_id, amount).
- **Output:** v1 = execution intent + self-reported outcome (no ledger); v1.5 = order state + holdings-ledger update + reconciliation.
- **Other:** **Equity/stock orders are never placed by Yesly** — those go via the customer's own external trading app; only MF/baskets execute here (§2A Gaurav). In-app execution is explicitly **v1.5**, deep-link in v1 (§11.3).

### Step 31 — Human-advisory handoff (Tier 4) · *off-app, HoA opt-in only*
- **Does:** Routes qualified users to a HoA advisor [S1 → Chart 03/04/11]. **Not self-served** — HoA opts a user in via an internal *"promote to Tier 4"* admin action. App role = **pre-meeting context summary** (advisor sees full user state) + **in-app calendar slot** → off-app conversation → **advisor enters post-meeting notes via the lightweight Advisor Console** → summary visible in-app + check-in reminders.
- **User does:** Book a slot; have the off-app conversation; read the summary in-app.
- **External API:** Scheduling — **Calendly / Cal.com / manual (open decision #8/#9)**. Advisor Console = Container 6 (React, role-gated; build-as-separate-app vs route-in-yeslifers-web is open).
- **Input:** Escalation trigger (user-initiated "talk to advisor", major life event in check-in, or drift/risk threshold); calendar selection.
- **Output:** Advisor summary (in-app); AUDIT_EVENT for the advisory interaction.
- **Other:** Tier 4 is the **AUM acquisition channel** for HoA's broader business; *"should feel premium, not transactional."* Advisor sees read-only state via Admin Surface 3 (household, tier history, plans, subscription, AA/KYC state, audit events).

### 6B.1 What the prototype confirms / changes vs the written spec
- **Confirmed built:** real coded **chat UI** (not video-led); the **dock / floating-star output repository** with running count; **inline result cards + Download-PDF**; the **headline → output → PDF** output pattern (§2A pivot — all now resolved).
- **Pricing resolved:** **Essential ₹2,999 / Growth ₹4,999** (₹249 was stale).
- **New facts surfaced:** SEBI **reg-no INA000018364**; **Google social login** as the primary auth; **life-event certainty/probability** as a captured field; **family-tree** UI realised.
- **Reordering vs §6.1:** income/expenses moved to *after* goals + first insight; the journey **interleaves engine outputs between data-capture sections** rather than batching all capture then all insights.
- **Beyond S9 (now filled from the spec, Steps 20–31):** payment, KYC, AA consent + data fetch, engine run, IP delivery, "Hell Yes" (Flow E), final downloads, the per-tier dashboards (F01–F09), the subscribed monthly loop, order execution, and advisor handoff. The exact **field-level contents of the FP/IP/Action-Plan deliverables** — the last gap from the prior coverage review — are now specified in **§6C** (extracted from the NWGC engine workbook, S6).
- **Still genuinely missing (no source anywhere):** *Figma screens* for everything past the paywall (Steps 20–31 are spec-only); the BBC video content (all 4 videos); and confirmation of the open decisions in §11.2.

---

### 6B.2 Input capture & storage — what we collect at each step and how it persists

> **Why this section.** Each step above lists its *Input* and *Output*; this section adds the missing third axis — **where that input goes and how it is stored** — mapped to the §8.1 data model and the §8.2 retention rules. Read §8 for the full model; this is the per-step lens.

**The storage model in one breath** [S1 → Chart 09; Doc 00 *Data Categories*]:
- **`HOUSEHOLD` is the root.** *Most financial entities hang off the household, not the user* — so a spouse never pays or re-enters data twice.
- **`USER`** (with `role_in_household` = primary / spouse / dependent) ↔ **1:1 `PROFILE`** (display_name, dob, persona, archetype, avatar, risk_profile).
- **Household-owned:** `GOAL`, `DEPENDENT`, `LIFE_EVENT`, `ASSET`, `LIABILITY`, `PLAN`, `SUBSCRIPTION`, `INVOICE`.
- **User-owned:** `AUTH_SESSION`, `NOTIFICATION_PREF`, `COMMUNITY_PROFILE`, `CONSENT_RECORD`, and the **immutable `AUDIT_EVENT`**.
- **`ASSET.source`** ∈ `manual / aa / cas_pdf / statement_pdf / broker` — the *same* asset record can be re-sourced (manual now, AA later); AA writes overwrite/augment with `source='aa'`.

**The five storage tiers (where bytes actually live)** [S1 → Doc 00 *Data Categories*]:

| Data category | Store | Encryption | Retention |
|---|---|---|---|
| Identity & auth | **AWS Cognito** | provider-managed | account lifetime |
| Profile & onboarding | RDS Postgres (Mumbai) | standard | account lifetime |
| **Financial profile** (income, assets, liabilities, goals, holdings) = **HIGH PII** | RDS | **field-level encryption** | **5–7 yr** (SEBI RIA) |
| **KYC documents** (PAN, Aadhaar, statements, CAS) = **CRITICAL PII** | **S3** (Mumbai) | encrypted, **restricted ACL** | per regulation |
| Plan outputs & advice | RDS + **S3** (PDFs) | field-enc / S3-enc | 5 yr from end of relationship |
| **Audit log** | RDS | **append-only**, SHA-256 `payload_hash` | **5+ yr** (no UPDATE/DELETE) |
| Analytics events | **Mixpanel** | pseudonymised | 13 months |

> **Three nuances worth pinning:** (1) the persona answers **`L1`/`E1`/`G1` are session-only — never persisted** [S3 → Data Storage]; only the resulting persona is written to `PROFILE`. (2) **Card data is tokenised at Razorpay and never touches Yesly servers** (so Yesly is *not* PCI-DSS in scope, §1). (3) **DPDP erasure = anonymise, not delete** — PII → tokens, but plans/audit/consent are retained in anonymised form (§9).

**Per-step capture → storage map:**

| # | Step | Input captured | Stored as (entity) | Class · store · retention |
|---|---|---|---|---|
| 1 | Splash | — | — | session bootstrap; Cognito session check |
| 2 | Welcome video | watched/skipped | analytics event | pseudonymised · Mixpanel · 13 mo |
| 3 | Login | email, name, avatar | `USER` + `AUTH_SESSION` | identity in **Cognito** (account lifetime) |
| 4 | Q1 + Q2 | `L1`, `E1` | **session only — not persisted** | held in session; drives matching/tone |
| 5 | Q3 | `G1` | **session only — not persisted** | reused for Stage-4 anchoring |
| 6 | Persona reveal | matched persona + ranked scores | `PROFILE` (persona, archetype) | RDS · account lifetime |
| 7 | Build / compliance | intent flag, avatar | `PROFILE` | RDS |
| 8 | SEBI consent | consent acceptance + timestamp | `CONSENT_RECORD` + `AUDIT_EVENT` | immutable · append-only · 5+ yr |
| 9 | Family tree | self, spouse, children, parents | `HOUSEHOLD` (root) · `USER` · `DEPENDENT` | RDS · household-owned |
| 10 | Liquid assets | `INV1–INV3` | `ASSET` rows (`source='manual'`) | **HIGH PII** · RDS field-enc · 5–7 yr |
| 11 | Illiquid assets | `INV4–INV7a` | `ASSET` rows (`source='manual'`) | HIGH PII · field-enc |
| 12 | Liabilities | `L1–L4` | `LIABILITY` rows | HIGH PII · field-enc |
| 13 | Net-worth insight | *(output)* | `PLAN`/insight + PDF | advice · RDS + **S3** · 5 yr |
| 14 | Free→Paid / teaser | — | Projections/Blindspots = plan artifacts | RDS + S3 |
| 15 | Goals | `YL1`, `YL1a–c` | `GOAL` rows | HIGH PII · field-enc |
| 16 | Life events | `YL3` (+ certainty) | `LIFE_EVENT` rows | HIGH PII · field-enc |
| 17 | Income + expenses | `I1–I4`, `YB1–YB4` | household financial-profile data (⚠ no explicit income/expense entity in Chart 09) | HIGH PII · field-enc · 5–7 yr |
| 18 | Full picture | *(output)* | `PLAN` + PDF | advice · RDS + S3 |
| 19 | Paywall | plan selection + payment | `SUBSCRIPTION` + `INVOICE` | card tokenised at **Razorpay** (not PCI) |
| 20 | Payment | payment token | `INVOICE`, `SUBSCRIPTION` | token only; conversion logged |
| 21 | KYC | PAN, Aadhaar, bank, selfie, agreement | KYC docs in **S3** (restricted ACL); `USER.kyc_status` (per Chart 11); `CONSENT_RECORD(kyc)`; `AUDIT_EVENT` | **CRITICAL PII** · per regulation |
| 22 | AA consent | consent params; fetched holdings | `CONSENT_RECORD(aa)`; `ASSET`/`LIABILITY` (`source='aa'`) | HIGH PII · field-enc |
| 23 | Engine run | full picture + assumptions | `PLAN` (engine_version, `assumption_set_id`) | household-owned · RDS |
| 24 | Harish video | watched | analytics event | pseudonymised · 13 mo |
| 25 | Investment Plan | *(output)* | `PLAN(investment)` + PDF; `AUDIT_EVENT` | advice · RDS + S3 · 5 yr |
| 26 | Hell Yes | decision (Path A/B) | plan/decision record + `AUDIT_EVENT` | advice-bearing · immutable |
| 27 | Downloads | — | PDFs served from S3 | — |
| 28 | Dashboards | notification & consent edits | `NOTIFICATION_PREF`, `CONSENT_RECORD` | user-owned |
| 29 | Monthly loop | accept/decline, check-ins | `USER_IDEA_MATCH`, decisions, `AUDIT_EVENT`, `INVOICE` | mixed · audit immutable |
| 30 | Execution | order intent | `ORDER` (idea_id nullable) | v1 self-reported flag; v1.5 full ledger |
| 31 | Advisor handoff | escalation, calendar | advisor-summary record + `AUDIT_EVENT` | advice · immutable |

---

## 6C. The Plan Output Schema — What the FP / IP / Action-Plan Deliverables Actually Contain (S6)

> **This closes the output-side gap.** Earlier reviews could describe the *inputs* at each step but not the *outputs*, because the engine's contents were a black box. The NWGC engine workbook (`SG & HS NWGC OCT 2025.xlsm`, S6) has now been parsed sheet-by-sheet — below is the field-level schema of what the engine computes and what each deliverable PDF contains. (Demo data in the workbook is the *"SG & HS"* family — a different sample from the Figma's Rajesh/Rahul.)

### 6C.1 Engine inputs (what the engine consumes)
- **Personal Details** (per person — Client/Spouse): Name, **PAN, Aadhaar**, job title, employer, years of experience, main income source, address, **DOB**, city of birth, free-text *Purpose* statement. Dependents block: Name + DOB for Child 1/2/3, Father, Mother, Other. *(PAN/Aadhaar here feed the KYC step; DOB drives the age-based projection.)*
- **Net-worth inputs** (per owner column → Total): **Realisable** — cash/bank, FD, equity stocks, equity MF, equity PMS, debt MF, hybrid, physical gold/silver, SGBs, gold/silver ETFs, vested ESOPs, international equities. **Unrealisable** — residential/commercial property, land, SSA, structured products, start-ups, NSC, life insurance/ULIPs, unvested ESOPs, PPF, EPF, NPS. **Liabilities** — home/car/education loan.
- **Life-event inputs**: name, amount, one-time/recurring, year, duration, inflation rate, linked asset, down-payment %/value, loan value/term/start.
- **Budget inputs**: itemised monthly income sources + 11 expense categories (Utilities, Personal Care, Housing, Food, Health, Family Care & Donations, Transportation, Recreation, Premiums & Fees, Loan EMIs) captured for **Current Lifestyle** *and* **At Retirement**; plus regular investments (PPF/EPF/NPS/SSA/model portfolio).

### 6C.2 Assumption set (the locked `assumption_set_id`)
From the `Assumptions` sheet (cached values): **Inflation 8% · Income growth 8% · Life expectancy 100 yrs.** Expected post-tax returns: **Indian Equity 11% · International Equity 8% · Debt 5% · Real Estate 3% · REITs/InvITs 7% · Gold 7% · SGBs 8%** (with per-instrument overrides: Bank 3.5%, EPF 8.1%, PPF 7.1%, NPS 8.5%, start-ups 12%).
> ⚠️ **Assumption divergence to reconcile:** these workbook values (**inflation 8%, debt 5%**) differ from the admin-console example set `v2026.Q2` quoted in S1 Chart 13 (**inflation 6%, debt 7%, equity 11%**). The engine team must confirm which set is canonical for v1 — it materially changes every projection. [S6 `Assumptions` vs S1 → Chart 13; relates to Admin Surface 5, §8.3.]

### 6C.3 Output sections → which deliverable they feed
Every output below is produced by the engine; the deliverables (Financial Plan / Investment Plan / Action Plan) are assembled from these. Source sheet in brackets.

| Output block | Key fields the user sees | App screen | Engine sheet | In FP? | In IP? |
|---|---|---|---|---|---|
| **Net Worth** | Total Realisable, Total Unrealisable, Total Assets, Total Liabilities, **Net Worth** (per owner + total) | B12 / Clarity Insight 1 (Step 13) | `Networth` | ✅ | ✅ |
| **Weighted Avg Return (WAR)** | Per-asset weight %, ERR, WAR contribution; **portfolio WAR %** (sample ≈7.57%); equity/debt/RE/commodity/startup sub-totals | B12 "WAR" | `WAR`, `WAR - Rev` | ✅ | ✅ |
| **Projections / Accumulation** | Year-by-year (age→100): net worth at begin/end, returns, investments, goals, expenses, loan o/s; "current path vs better allocation vs full potential" | B13 / D08 projection bars | `working`, `NW CHARTS`, `Accumulation Chart`, `YoY WAR`, `YoY Assets` | ✅ | ✅ |
| **Budget / Cashflow** | Income (nominal + inflation-adjusted), 11 expense categories, **surplus/deficit per year**, current-vs-retirement lifestyle | D06 surplus / Full Picture (Step 18) | `Yearly Budget`, `Cashflow Working` | ✅ | — |
| **Life Events timeline** | Inflation-adjusted cost per event, down-payment, loan value/term, withdrawal year | C06 / Yes List | `Key Life Events` | ✅ | — |
| **Goal / Scenario (NWGC)** | Two side-by-side projections **AS-IS vs SCENARIO** (age·year·income·expenses·life-events·investments·withdrawals·realisable·unrealisable·liabilities·**total net worth**·YoY-WAR); retirement age; **"money runs out at age"** | Hell Yes Flow E (Step 26) | `NWGC - AS IS`, `NWGC - SCENARIOS`, `Withdrawal Working` | ✅ | — |
| **Protection — HLV** | Existing cover, net assets to liquidate, % expenses required, O/S liabilities → **Additional Insurance Required** (term-cover gap; sample ₹2.92Cr) | B09 Protection | `HLV Calc`, `HLV Working` | ✅ | — |
| **Protection — Health** | Risk score from age/location/health & family history/lifestyle → **Final Score + Category**; recommended **Base + Top-up cover, Critical Illness (1× annual expenses), Accidental (2× annual income)** | B09 Protection | `Health Insurance` | ✅ | — |
| **Wheel of Wealth** | Scored radar (0–10) across **Risk cover · Cashflow · Asset growth · Will-the-money-last · Succession readiness**; concentration/inflation/liquidity sub-scores | Health-score visual (F04) | `WoW - Working` | ✅ | — |
| **Loan / EMI** | Per-loan amortization (balance, EMI, interest, principal, rate, term, payoff date) | B08 Liabilities | `Loan Amortization` | ✅ | — |
| **Allocation gap & action report** | Current vs **target allocation**, gap analysis, **buy/sell/hold per holding**, risk-adjusted suitability per holding | F06 portfolio / IP | (engine, against AA holdings) | — | ✅ |

### 6C.4 The three deliverables, defined
- **Financial Plan (Tier 1) — "20+ page document"** [S1 → Chart 01]. Built from *user-entered* data. Contains: goal mapping (short/medium/long), life-events impact, budget guidance, **generic** asset-allocation *framework*, Hell-Yes decisions, household-aware planning, completed SEBI risk profile, net worth + WAR + projections + protection (HLV/health) + Wheel of Wealth. Delivery: **PDF + in-app + email.**
- **Investment Plan (Tier 2)** — portfolio-*specific*, against **actual holdings** (AA/CAS/statement/manual). Contains: current allocation, **target allocation**, gap analysis, **action report (buy/sell/hold)**, risk-adjusted suitability per holding. Delivery: **PDF + in-app** (watermarked preview).
- **Action Plan** — the prioritised "what to do next," delivered as the **"Yes Pathway" checklist** (This Week / This Month / Month 5+, screen F05) and downloadable [S4 → rows 23/28]. Sequenced from the gap analysis + most-urgent actions.

> **Net:** the engine produces a far richer output set than the app currently surfaces inline — only Net Worth, WAR, projections and surplus appear as Figma cards (Steps 13/18). HLV insurance gap, health-cover recommendation, Wheel-of-Wealth score and the full year-by-year scenario tables exist in the engine and belong in the FP PDF, but have **no Figma screen yet** — design input needed for how/whether to surface them in the chat vs PDF-only.

---

## 7. Technical Architecture

### 7.1 Seven external integrations (each a separate partner line-item) [S1 → Doc 00 *System Context*]
1. **App Stores** (Apple, Google) — distribution.
2. **Account Aggregator** — Setu / Anumati / OneMoney / CAMSFinServ (read-only financial data via consent; the S5 requirements sheet also names **Finvu**).
3. **KYC vendor** — Digio / IDfy / Hyperverge (triggered at Tier 2+).
4. **Broker gateways** — smallcase Gateway + BSE StAR MF (v1 deep-link; v1.5 in-app).
5. **Payment gateway** — Razorpay (one-time + subscription).
6. **Notification channels** — Expo Push, Amazon SES email, WhatsApp BSP.
7. **Analytics/observability** — Mixpanel + Sentry.

### 7.2 Eight internal containers [S1 → Doc 00 *Internal Containers*]
Containers 3–8 sit on **AWS Mumbai (ap-south-1)** for DPDP residency. Yesly's stack is *distinct from* the YesLifers community + GTM cockpit (which run on **Supabase**); unifying them is out of v1 scope.

| # | Container | Tech | Owner |
|---|---|---|---|
| 1 | Mobile App (all user-facing) | React Native via Expo | Partner |
| 2 | Marketing Web (landing, SEO, blog) | React Vite on Vercel | Partner |
| 3 | API Backend (auth, logic, integrations, webhooks) | **Python FastAPI preferred** (Node acceptable) | Partner |
| 4 | **Financial Planning Engine** (plan gen, allocation, drift, rebalancing) | Python, **NWGC-derived** | **Internal (HoA)** |
| 5 | Notification Dispatcher (push/email/WhatsApp fan-out) | Async worker (Celery/BullMQ) | Partner |
| 6 | Advisor Console (read-only user state + post-meeting notes) | React web, role-gated | Partner |
| 7 | Internal Admin Console (7 surfaces — see §8) | React web, role-gated | Partner |
| 8 | Database + Storage | AWS RDS Postgres (Mumbai) + S3 | Partner-built, new for Yesly |

> **Critical scope split (repeated for emphasis in the doc):** Container 4 is **built & maintained by the internal HoA engine team**; the partner only builds an API wrapper around it. *"This split must be explicit in the partner contract. Without it, the partner will scope engine work into their quote — billing for work the internal team is already doing."* [S1 → Doc 00 *Critical scope split*].

### 7.3 Tech stack with rationale ("constraints, not options") [S1 → Doc 00 *Tech Stack — With Reasoning*]
React Native/Expo (EAS Build) — shared codebase with YesLifers · React Vite on Vercel — existing HoA pattern · **FastAPI** — matches the engine's language, removes a language boundary · **AWS RDS Postgres, Mumbai** — DPDP residency, familiar to SEBI reviewers · **AWS Cognito** auth (email OTP + social) · **AWS S3** (Mumbai, field-level encryption for KYC) · **Expo Push** (wraps FCM+APNs) · **Amazon SES** email · **WhatsApp:** AiSensy / Gupshup / Wati (deferred) · **AA:** Setu/Anumati/OneMoney/CAMSFinServ (deferred) · **KYC:** Digio/IDfy/Hyperverge (deferred) · **Razorpay** (Cashfree alt) · **smallcase + BSE StAR MF** brokers · **Mixpanel** analytics · **Sentry** · **GitHub Actions** CI/CD · **Vercel** web hosting.

### 7.4 Environments [S1 → Doc 00 *Environments*]
Dev (local + CI, local/RDS-dev, Expo Go) → Staging (internal QA, advisor UAT, compliance review; separate RDS with anonymised seed; TestFlight + Play Internal) → Production (real users, multi-AZ RDS Mumbai, App Store + Play Store). *"The partner ships through this pipeline. They do not get their own staging environment that no one else can see."*

---

### 7.5 System sequences & lifecycles (Charts 05–08) — the technical view

> The §6B steps give the *user* view of these flows; this captures the *system* view from the architecture doc — actors, API calls, states, webhooks and error paths. Four of the doc's 13 charts are dedicated system sequences; **Chart 08 (billing) had no prior coverage in this analysis.**

#### 7.5.1 Chart 05 — Account-Aggregator consent & data fetch (the most complex v1 integration) [S1 → Chart 05]
Actors: `USER · APP · BACKEND · AA VENDOR (Setu/…) · USER'S FI`.
1. **Initiate consent** — App → `POST /aa/consent/initiate { household_id, requested_fi_types, purpose, frequency, duration }` → Backend validates suitability, logs intent, calls vendor `POST /consent` → vendor returns **consent handle + redirect URL** → user approves in the AA UI → vendor webhook *"consent approved"* → Backend sets state **ACTIVE**, triggers initial fetch.
2. **Initial fetch** — `POST /fi-request` (consent_handle) → vendor forwards to each FI → FI prepares encrypted data → webhook *"data ready"* → `GET /fi-data` (session key) → Backend **decrypts (private key)**, parses (balances, MF holdings, deposits, loans), normalises to internal schema, stores **field-encrypted** in Postgres, recomputes household net worth, triggers plan refresh.
3. **Scheduled refresh** — cron every N days per `consent.frequency`, re-using the consent handle (the Tier-3 live-data loop).
4. **Revocation** — `POST /aa/consent/revoke` → vendor revokes & notifies FIs → state **REVOKED**, historical data retained per policy, scheduled refresh stops.
5. **Expiry** — cron detects `expires_at < now` → state **EXPIRED** → notify *"re-link your accounts"* → banner.
- **Error paths:** vendor downtime → retry queue + graceful degradation; FI declines → log, notify, allow manual entry; stale data > N days → "outdated" badge; failed decryption → alert engineering.

#### 7.5.2 Chart 06 — Recommendation pipeline (curated, v1) [S1 → Chart 06]
Actors: `HoA ANALYST · ADMIN CONSOLE · COMPLIANCE REVIEWER · BACKEND · APP · USER`.
1. **Authoring** — analyst drafts idea via `POST /admin/ideas (draft)`: thesis, constituent instruments, target allocation, **risk classification**, suitability criteria (min age/income, risk-profile range, horizon), source refs → state **DRAFT**, version-tagged, notifies compliance.
2. **Compliance review** — reviewer applies SEBI RIA checks (no return promises, disclosures present, suitability sensible, conflict declared) → **APPROVE / REJECT / REQUEST_CHANGES** via `POST /admin/ideas/{id}/review`. APPROVED → immutable audit record + triggers matching; else → back to DRAFT with notes.
3. **Suitability matching (per Tier-3 user)** — for each approved idea × user, evaluate: risk profile in range? portfolio not over-concentrated? doesn't break asset mix? not a duplicate holding? meets investable surplus? → create `USER_IDEA_MATCH` or log decline reason.
4. **Notification + decision** — push + in-app inbox → idea detail (thesis, suitability badge, "why this for you", suggested amount, risk warnings, disclosures) → **ACT NOW / SAVE / NOT INTERESTED / TALK TO ADVISOR** via `POST /ideas/{id}/decision` → audited. ACT NOW → order (Chart 07); TALK → Tier-4 escalation.
5. **Post-publication drift** — daily cron recomputes thesis health; if broken → notify analyst → amend/**RETIRE** (notify users who acted).
- **Separation of duties** (author ≠ reviewer ≠ matcher) is SEBI-relevant; suitability is a server-side filter, not a UI hide. **v2 changes only step 1** (algorithm-generated drafts); 2–5 are identical — which is why the compliance gate must be solid in v1.

#### 7.5.3 Chart 07 — Order placement (v1 deep-link · v1.5 in-app) [S1 → Chart 07]
- **v1 deep-link:** `POST /orders/intent { user, idea_id, target_amount }` → **compliance gate re-verifies suitability at click time** → build deep-link (smallcase basket / BSE StAR MF) → user executes **outside Yesly** → returns → app asks *"Did you complete the trade?"* → log **self-reported** outcome (no system confirmation; portfolio confirmed only at next AA refresh — up to 7 days stale).
- **v1.5 in-app:** `POST /orders/place` → compliance gate → **market-hours check** (closed → queue) → broker-linked check (OAuth-style linking; stores only "linked" status) → construct payload → smallcase Gateway / BSE StAR MF API → broker places → `order_id`, state **PLACED** → async webhook **EXECUTED / PARTIALLY_FILLED / REJECTED / CANCELLED** → update **internal order ledger** → notify.
- **Reconciliation (v1.5, daily cron):** fetch broker holdings + AA holdings → merge → discrepancy? flag for review : update consolidated portfolio, recompute drift / net worth / goal progress. **v1 has no order ledger.**

#### 7.5.4 Chart 08 — Subscription billing lifecycle (Tier 3) [S1 → Chart 08] — *previously uncaptured*
Actors: `USER · APP · BACKEND · RAZORPAY`. *"Razorpay does the heavy lifting; Yesly stores state and reacts to webhooks."* Five flows, **all required in subscription v1:**
1. **Creation** — tap "Subscribe" → `POST /billing/subscribe` → Backend `POST /subscriptions (Razorpay) { plan_id, customer_email, start_date }` → returns `short_url` → state **PENDING** → hosted checkout → user authorises card/UPI **mandate** → first charge → success → webhook `subscription.activated`; fail → `subscription.charge_failed` (no sub created; app retries).
2. **Monthly charge** — Razorpay recurring cron charges the method → success → webhook `subscription.charged` → mark month paid, extend access window, log invoice, notify *"Payment received"*; fail → webhook `payment.failed` → begin dunning.
3. **Dunning** — mark **PAST_DUE**, start grace window (~7 days): **Day 1** email → **Day 3** email + in-app banner → **Day 5** push + intensified banner → **Day 7** final email *"pauses tomorrow"*. Resolved? **YES** → reactivate, resume cycle. **NO** → **downgrade to Tier 2** (keep last plan readable, revoke live features, "we miss you" email).
4. **Cancellation** — Settings → "Cancel" → reason picker + downgrade implications → `POST /subscriptions/{id}/cancel` → Razorpay sets `cancel_at_period_end = true` → state **CANCEL_PENDING**, access retained until period end → webhook `subscription.completed` → downgrade to Tier 2, preserve plan + history, "come back anytime" email.
5. **Refund** — **not self-service:** user requests via support → HoA support reviews/decides → Admin Console `POST /admin/refund` → Backend `POST /payments/{id}/refund (Razorpay)` → webhook `refund.processed` → log refund, adjust state, notify.
- **GST / invoicing (cross-cutting):** Razorpay generates **GST-compliant invoices** (Yesly's GSTIN configured at the Razorpay account); invoice PDF → object storage; user downloads from in-app receipts; quarterly GST returns filed via HoA's CA. *GST is handled at the Razorpay layer, not in Yesly's code — operational, not engineering.*
- These five states map exactly to `SUBSCRIPTION.state ∈ {pending, active, past_due, cancel_pending, canceled}` in §8.1.1.

---

## 8. Data Model & Internal Admin Console

### 8.1 Household & user data model [S1 → Chart 09]
- **HOUSEHOLD is the root entity.** A USER belongs to a household; **most financial entities hang off the household, not the user.** *"This is the core structural decision."* [S1 → Chart 09 Notes].
- USER (role_in_household: primary/spouse/dependent) → 1:1 PROFILE (display_name, dob, persona, archetype, avatar, risk_profile, risk_assessed_at).
- **Household-owned:** GOAL, DEPENDENT, PLAN (version, tier financial/investment, state, engine_version, **assumption_set_id**), LIFE_EVENT, ASSET (with optional `owner_user_id`; `source` ∈ aa/cas_pdf/statement_pdf/manual/broker), LIABILITY.
- **User-owned:** AUTH_SESSION, NOTIFICATION_PREF, COMMUNITY_PROFILE (pledge_accepted_at), **AUDIT_EVENT (immutable!)**, CONSENT_RECORD (type aa/kyc/marketing/subscription).
- **Subscription is per household, not per user** ("the spouse does not pay separately") → SUBSCRIPTION + INVOICE.
- **Recommendation/Order (v1.5):** IDEA (global, once) → USER_IDEA_MATCH (per-user, suitability cached) → ORDER (broker smallcase/bse_star; `idea_id` nullable for rebalances).
- **Scope split:** HoA designs the entity model & relationships ("product expression, cannot be outsourced"); the partner builds implementation (indexes, partitioning, migrations, query optimisation, pooling, backups/replicas) as a paid consultant in the discovery sprint [S1 → Chart 09 *Scope split*].
- `PLAN.assumption_set_id` locks each plan to the assumptions it was generated against — *that's how "what happens when assumptions change" is handled* [S1 → Chart 09 Notes].

#### 8.1.1 Full entity schema — column-level, verbatim from Chart 09 [S1 → Chart 09]
> This is the **complete logical schema** as drawn in the architecture doc (19 entities, with keys, enums and JSON fields). It is a *design-level* schema — it deliberately omits physical data types, indexes, constraints and migrations, which are the partner's job (see *Scope split* above). Notation: `(PK)` primary key, `(FK)` foreign key, `∈ {…}` enum value-set.

**Core identity**

| Entity | Columns |
|---|---|
| **HOUSEHOLD** *(root)* | `id (PK)` · `created_at` · `primary_user_id` · `household_name` · `household_currency` |
| **USER** | `id (PK)` · `household_id (FK)` · `email` · `phone` · `role_in_household ∈ {primary, spouse, dependent}` · `created_at` |
| **PROFILE** *(1:1 with USER)* | `user_id (PK,FK)` · `display_name` · `dob` · `persona` · `archetype` · `avatar` · `risk_profile` · `risk_assessed_at` |

**Household-owned (the financial picture)**

| Entity | Columns |
|---|---|
| **GOAL** | `id (PK)` · `household_id (FK)` · `name` · `type ∈ {retirement, home, children_ed, travel, …}` · `target_amount` · `target_date` · `priority` · `created_at` |
| **DEPENDENT** | `id (PK)` · `household_id (FK)` · `name` · `relationship ∈ {child, parent, other}` · `dob` · `financial_link ∈ {full, partial, future}` |
| **PLAN** | `id (PK)` · `household_id (FK)` · `version` · `tier ∈ {financial, investment}` · `state ∈ {draft, active, archived}` · `generated_at` · `pdf_url` · `engine_version` · `assumption_set_id` |
| **LIFE_EVENT** | `id (PK)` · `household_id (FK)` · `type ∈ {marriage, kid, home, job_change, sabbatical, …}` · `date (planned or actual)` · `financial_impact` · `linked_goal_id` |
| **ASSET** | `id (PK)` · `household_id (FK)` · `owner_user_id (FK, optional)` · `type ∈ {equity, mf, fd, real_estate, gold, crypto}` · `value` · `as_of_date` · `source ∈ {aa, cas_pdf, statement_pdf, manual, broker}` · `metadata (JSON)` |
| **LIABILITY** | `id (PK)` · `household_id (FK)` · `owner_user_id (FK)` · `type ∈ {home_loan, personal_loan, credit_card}` · `principal` · `rate` · `tenure` · `emi` · `source` |

**User-owned (not household)**

| Entity | Columns |
|---|---|
| **AUTH_SESSION** | `user_id (FK)` · `token` · `expires_at` |
| **NOTIFICATION_PREF** | `user_id (FK)` · `category` · `channel` · `enabled` |
| **COMMUNITY_PROFILE** | `user_id (FK)` · `pledge_accepted_at` · `display_handle` · … |
| **AUDIT_EVENT** | `id (PK)` · `user_id (FK)` · `household_id (FK)` · `event_type` · `payload (JSON)` · `created_at` **(immutable!)** |
| **CONSENT_RECORD** | `id (PK)` · `user_id (FK)` · `type ∈ {aa, kyc, marketing, subscription}` · `granted_at` · `revoked_at` · `expires_at` |

**Billing (per household)**

| Entity | Columns |
|---|---|
| **SUBSCRIPTION** | `id (PK)` · `household_id (FK)` *(per household, not per user)* · `razorpay_id` · `state ∈ {pending, active, past_due, cancel_pending, canceled}` · `started_at` · `current_period_end` |
| **INVOICE** | `id (PK)` · `subscription_id (FK)` · `razorpay_id` · `amount` · `gst` · `status` · `period_start` · `period_end` · `invoice_pdf_url` |

**Recommendation & orders (v1.5)**

| Entity | Columns |
|---|---|
| **IDEA** | `id (PK)` · `version` · `state` · `thesis` · `constituents (JSON)` · `risk_class` · `suitability_rules (JSON)` · `reviewer_id` · `approved_at` |
| **USER_IDEA_MATCH** | `id (PK)` · `idea_id (FK)` · `user_id (FK)` · `matched_at` · `decision ∈ {act, save, decline, advisor}` · `decided_at` |
| **ORDER** | `id (PK)` · `user_id (FK)` · `household_id (FK)` · `idea_id (FK, nullable — rebalances)` · `broker ∈ {smallcase, bse_star}` · `state` · `amount` · `placed_at` · `executed_at` · `broker_ref` |

> **Note** `IDEA.risk_class` + `IDEA.suitability_rules` are how a recommendation is matched to users at runtime (Chart 06) — this is *per-idea* classification, not a user-bucket taxonomy. (A separate "users assigned to 16–20 allocation classes" idea was raised verbally but is **not** in any source document, so it is not modelled here.)

### 8.2 Data categories, retention & storage [S1 → Doc 00 *Data Categories*]
Identity+auth (account lifetime, Cognito) · Profile+onboarding (RDS) · **Financial profile = HIGH PII** (income/assets/liabilities/goals/holdings; **5–7 yrs per SEBI RIA**; RDS field-encrypted) · **KYC documents = CRITICAL PII** (PAN/Aadhaar/statements/CAS; per regulation; S3 encrypted, restricted ACL) · Plan outputs+advice (5 yrs from end of relationship; RDS + S3 PDFs) · **Audit log = immutable** (every advice event/acceptance/order; 5+ yrs append-only) · Analytics events (13 months; Mixpanel, pseudonymised).

### 8.3 Internal Admin Console — 7 surfaces, 4 roles [S1 → Chart 13]
*"Often dismissed as 'just CRUD.' It is not… the operational backbone of Yesly."* **Estimated honest scope: 4–6 engineering weeks for v1; most agencies will quote 1–2.**
- **Roles:** Analyst (authors ideas/plans, can't approve own) · Compliance Reviewer (reviews/approves, audits, searches logs, can't author) · Support (user lookup, refunds within threshold, grievances; can't author/modify financial data) · Admin/Engineer (everything + assumption-version management + config; **no one can edit the audit log**).
- **Surfaces:** (1) Idea/Basket publishing · (2) Compliance review queue · (3) User lookup & state inspection (every sensitive view is itself audited) · (4) Refund/payment handling (escalates above Support limit) · (5) **Plan/Assumption version management** (e.g. active set `v2026.Q2`: equity 11%, debt 7%, expense inflation 6%, ~14 assumptions; create → review → activate → optionally regenerate plans) · (6) Grievance/support ticketing (SLA timer, escalation) · (7) Audit-log search (filter/export to CSV/PDF for SEBI; *"accessing the audit log is itself audited — by design"*).
- Why agencies under-quote it: underestimate suitability matching (Surface 1), ignore SLA tracking (6), treat audit search as "just a query," forget Surface 5 entirely, defer role-gating (a compliance failure) [S1 → Chart 13 Notes]. Role-gating is enforced at the API layer, not just UI hiding.

---

## 9. Compliance Posture (SEBI RIA + DPDP)

Yesly is **a digital channel for a SEBI-registered Investment Adviser (House of Alpha)** [S1 → Doc 00 *Compliance Posture*]. Requirements baked into the architecture:
- Enforce the **SEBI RIA advertisement code**: no return promises, no testimonials, mandatory disclosures **shown *and logged*** [S1 → Doc 00; Chart 10].
- **Immutable audit logs of every advice-bearing interaction**, 5 yrs from end of client relationship [S1 → Doc 00; Chart 10].
- **DPDP Act 2023** data-subject rights — access, correction, erasure — within statutory timelines [S1 → Doc 00; Chart 12].
- **Resolve the SEBI-vs-DPDP conflict by anonymising on erasure** (not deleting): audit/advisory records retained in anonymised form [S1 → Doc 00; Chart 10; Chart 12 A3]. Erasure carries a **30-day cooling-off**; PII → tokens (`email→anon_xxx@yesly-anon.local`, phone→NULL, PAN/Aadhaar→deleted), plans/audit/consent retained anonymised.
- **All personal data in Indian regions (ap-south-1)** [S1 → Doc 00].
- **KYC delegated to a licensed third-party**, triggered at Tier 2 entry (Open question: SEBI may require it at Tier 1 if the Financial Plan counts as "advice" — compliance to adjudicate) [S1 → Doc 00; Chart 11].
- **VAPT clearance before launch and annually** [S1 → Doc 00].

**Audit trail (Chart 10):** every advice-bearing event is logged immutably across 8 categories (Identity, Suitability, Consent, Plan/advice, Idea/recommendation, Order, Advisor, Grievance). The `AUDIT_EVENT` table is **APPEND-ONLY** (no UPDATE/DELETE, DB-level constraint via Postgres triggers / write-only role — *"'we just don't UPDATE in code' is insufficient"*), with `payload_hash` (SHA-256) for tamper detection. It is built to answer SEBI inspection questions ("show me the advice you gave user X", "what was the suitability basis", "who reviewed this", "show the engine version + assumptions") in minutes [S1 → Chart 10].

**KYC — two distinct layers (must not be conflated)** [S1 → Chart 11]: **Layer 1** = HoA RIA KYC + Advisory Agreement (PAN → Aadhaar OTP/DigiLocker → bank penny-drop → selfie/liveness → advisory-agreement accept → KYC vendor validates with IT dept + sanctions screening); **Layer 2** = Broker KYC at smallcase/BSE StAR (broker's own CKYC/CVL flow — Yesly is blind to it in v1 deep-link; v1.5 uses OAuth-style linking and stores only a "linked" status). Re-KYC is cron-nudged 30 days before expiry (SEBI: low-risk 10y / medium 8y / high 2y).

**Grievance redressal (Chart 12 Part B):** "Raise a complaint" surface, categorised intake + severity, ticket SLA (ack 7 days), system-enforced escalation (T+24h not ack'd → manager; T+7d → senior; T+15d → flag for SEBI report), escalation paths to **SEBI SCORES portal** / Ombudsman, monthly complaint-statistics export for HoA to submit to SCORES (manual).

---

### 9.1 The `AUDIT_EVENT` record in full + disclosure triggers (Chart 10, complete) [S1 → Chart 10]
> ⚠️ **Schema reconciliation:** Chart 10 specifies a **richer `AUDIT_EVENT` than Chart 09's** (§8.1.1). The audit table should be built to this **superset**:

`AUDIT_EVENT` = `id (ULID, time-ordered)` · `timestamp (server, UTC)` · `user_id` · `household_id` · `actor_id` · `actor_type ∈ {user, staff, system}` · `event_type` · `event_category` · `payload (JSON)` · `payload_hash (SHA-256)` · `schema_version` · `created_at` **(immutable)**.
- **Enforced by design:** APPEND-ONLY (no UPDATE/DELETE, **DB-level** via Postgres triggers or a write-only role — *"'we just don't UPDATE in code' is insufficient"*); **separate table + separate backup cadence**; ≥5-yr retention; anonymise (not delete) on DPDP erasure.
- **Eight logged categories** (with example events): **Identity** (registered, KYC submitted/approved/rejected) · **Suitability** (risk profile completed/updated, household snapshot) · **Consent** (AA grant/revoke/expire, marketing toggle, subscription mandate) · **Plan/advice** (generated w/ version+engine, paid, delivered, revised) · **Idea** (authored, reviewed, approved, retired, suitability match, user decision) · **Order** (intent v1, placed v1.5, executed, failed/cancelled) · **Advisor** (meeting scheduled, summary entered, advice delivered) · **Grievance** (registered, SLA tracking, resolution/escalation).
- **SEBI inspection readiness — the questions it must answer in minutes:** *"show me the advice given to user X"* → query by `user_id` + category ∈ (plan, idea, advisor); *"suitability basis?"* → suitability + risk_profile events preceding the advice; *"did the user consent?"* → `CONSENT_RECORD` + grant timestamp; *"who reviewed this?"* → `event_type=idea_reviewed`, get `actor_id`; *"engine version + assumptions?"* → `plan.engine_version` + `plan.assumption_set_id` (both immutable); *"grievance log for the year?"* → category=grievance.
- **Disclosure triggers** (every recommendation surface must display **and log**): SEBI RIA registration number · *"Investments are subject to market risk"* · conflict-of-interest declaration · past-performance disclaimer. *The display itself is logged as an event — that is what auditors check.*
- **Six consent moments** (each = one `CONSENT_RECORD` + audit event): account creation (T&C/privacy) · marketing comms · AA data sharing · subscription mandate (recurring debit) · KYC info sharing (Tier 2) · Tier-4 advisory engagement letter.

### 9.2 DPDP data-subject rights — the three user flows (Chart 12 Part A) [S1 → Chart 12]
All three need user-facing surfaces in v1; all write to the immutable audit trail. (Grievance redressal = Part B, already covered above.)
- **A1 — Access ("download my data"):** `POST /dpdp/export-request` → async worker gathers users/profile/household, **all plans (PDF + structured)**, financial data, portfolio history, notifications, relevant audit events, consent records → packages as **JSON + PDFs in a zip** → uploads to a **short-lived signed URL (72 h)** → emails link; download logged; archive auto-deletes after 72 h. **SLA 30 days per DPDP; realistic 24–48 h.**
- **A2 — Correction ("fix my data"):** self-correctable profile fields edited inline; **non-self-correctable items (KYC data, transaction history, audit events)** → admin-mediated correction ticket via the grievance flow; if a correction affects `PLAN` data → optional plan regeneration; every correction audit-logged.
- **A3 — Erasure ("delete my account"):** **anonymise, do not delete** (resolves the SEBI 5-yr-retention vs DPDP-erasure conflict). Confirmation → **30-day cooling-off** ("sign in before then to cancel") → state `ERASURE_PENDING` → at T+30 an async worker tokenises PII (`email→anon_xxx@yesly-anon.local`, `phone→NULL`, `name→"Anonymised User"`, **PAN/Aadhaar/bank → deleted**, avatar→default) while **retaining (anonymised) plans, audit events, consent records, advisory interactions** for SEBI; cancels Razorpay subs, revokes AA consents, invalidates tokens → state `ERASED` (irreversible), audit event `account_anonymised`.

---

## 10. The Financial-Planning Engine (NWGC)

Container 4 is *"Python (NWGC-derived, internal)"* [S1 → Doc 00]. The actual engine spreadsheet is **`SG & HS NWGC OCT 2025.xlsm`** (S6), whose 30 sheets reveal exactly what the engine computes [S6 → sheet list]:
- **Inputs/setup:** `Personal Details`, `Assets`, `Assumptions` (the assumption set referenced by `PLAN.assumption_set_id` / Admin Surface 5).
- **Net-worth & allocation:** `Networth`, `NW CHARTS`, `WAR`, `WAR - Rev`, `YoY WAR`, `Yoy WAR OG` — **WAR = the "Net Worth + WAR" Clarity insight** on screen B12 [maps to S2 → B12 "net worth + WAR"; S4 → row 14 "Clarity Insight – NW + War"].
- **Projections & accumulation:** `YoY Assets`, `Yoy Assets Working`, `Accumulation Chart`, `Withdrawal Working` — feed the projection bars (B13, D08) [maps to S2 → B13/D08].
- **Life events:** `Key Life Events`, `LIFE EVENTS` → the Life-Events module / "Yes List" non-discretionary events [maps to S1 Chart 01; S2 → C06].
- **Budget & cashflow:** `Budget`, `Yearly Budget`, `Yearly Budget (Pie)`, `Cashflow Working`, `Cashflow Chart` → the "Yes Budget" step [maps to S2 → Flow D].
- **Goal-clarity scenarios:** `NWGC - AS IS`, `NWGC - SCENARIOS`, `working` → the "Hell Yes" two-path scenario walk-through (NWGC ≈ Net-Worth/Goal/Cashflow scenarios) [maps to S2 → Flow E].
- **Protection:** `HLV Calc`, `HLV Working` (Human Life Value → term-cover gap), `Health Insurance` → Protection section [maps to S2 → B09].
- **Wheel of Wealth:** `Wheel of Wealth`, `WoW - Working` (a holistic financial-health visual).
- `Loan Amortization` → liabilities/EMI logic [maps to S2 → B08].

> **Read:** the NWGC workbook *is* the source-of-truth maths the app's outputs (Clarity insights, projections, budget surplus, HLV insurance gap, Hell-Yes scenarios) are derived from. The "SG & HS" prefix suggests it's an existing client-plan template ("SG" / "HS" initials). The internal team's job is to port this workbook's logic into the Python engine (Container 4).
>
> **✅ Now extracted (see §6C):** the workbook's full **input fields, assumption set, and output schema** have been parsed sheet-by-sheet and mapped to the FP/IP/Action-Plan deliverables — this closes the prior output-side gap. Headline cached values in the sample: Total Assets ₹6.62Cr, Liabilities ₹11.4L, Net Worth ₹6.51Cr, portfolio WAR ≈7.57%, projections run age→100, HLV additional-cover ₹2.92Cr. ⚠️ The workbook's assumptions (**inflation 8%, debt 5%**) **diverge** from S1 Chart 13's `v2026.Q2` example (**inflation 6%, debt 7%**) — reconcile which set is canonical (§6C.2). *(Cell-level formula logic still not transcribed — flag if the engine team needs the exact formulas, not just the I/O schema.)*

---

## 11. Third-Party Integrations, Open Decisions & Out-of-Scope

### 11.1 Integration requirements (the product-side view) [S5 → Third Party Integrations sheet]
| Integration | Requirement in app |
|---|---|
| **AA (e.g. Finvu)** | If user pays for Paywall 2 & 3, collect details via AA first, then show the Investment Plan |
| **BSE Star** | After execution plan is presented & user wants to execute on our platform |
| **smallcase gateway** | Same — execution on our platform |
| SMS provider (e.g. Twilio) | — |
| Mail provider (e.g. SendGrid) | — |
| Martech (e.g. Mixpanel) | — |
| Blogs (e.g. Ghost) | — |
| **KYC (via KRA, CVL/CAMS/DigiLocker)** | *"Need to check with legal team where we should do the KYC"* |
| Push notification (Firebase/APNS/GCM) | — |
| **HoA planning APIs** | For all outputs — after collecting all the data (this is the Container-4 engine wrapper) |
| Zerodha? | (open) |
| Data source (e.g. CMOTS/Accord/Value Research) | market/instrument data |
| Small integrations | bank IFSC master, etc. |

> Note the slight vendor divergence vs S1: S5 names **Finvu** (AA), **Twilio/SendGrid/Ghost/Firebase** and a **market-data source (CMOTS/Accord/Value Research)** that S1 doesn't dwell on; S1 names **Setu/OneMoney/CAMSFinServ, Amazon SES, Expo Push**. Treat S1 as the *current* architecture and S5 as the *requirements wishlist* [compare S1 Doc 00 vs S5 Third Party Integrations].

### 11.2 Ten open decisions that affect the partner quote [S1 → Doc 00 *Open Decisions*]
1. AA vendor (pricing varies 3–5×) · 2. KYC vendor · 3. WhatsApp BSP · 4. API language (Python vs Node) · 5. **KYC trigger tier (1 vs 2)** · 6. **Pricing (FP / IP / Subscription)** · 7. Subscription model (monthly only vs annual) · 8. Advisor Console (separate app vs route in yeslifers-web) · 9. Tier-4 scheduling (Calendly / Cal.com / manual) · 10. Refund window (7/14/30 days).

### 11.3 Explicit out-of-scope for v1 (kills scope creep) [S1 → Doc 00 *Explicit Out-of-Scope*]
In-app order execution (→ v1.5, deep-link in v1) · full algorithmic idea generation (→ v2; curated-by-human in v1) · UPI P2P · card issuance · lending/BNPL · AI chatbot/generative advisor · voice · tax filing · will/estate planning · insurance comparison/purchase · real-time market data · tablet/desktop native apps · multi-language beyond English · offline mode.

### 11.4 v1 → v1.5 → v2 evolution (collected) [S1 → Charts 03, 06, 07, 11]
- **v1:** deep-link execution (no order ledger, user self-reports completion, portfolio confirmed only at next AA refresh — *dashboard up to 7 days stale post-trade*); human-curated recommendations; KYC at Tier 2 (assumed).
- **v1.5:** in-app execution via smallcase/BSE StAR APIs + an order ledger + daily **reconciliation** (broker vs AA holdings) + market-hours handling.
- **v2:** algorithm-generated idea drafts (Steps 2–5 of the recommendation pipeline stay identical — *that's why the compliance gate must be robust in v1, it's the v2 control surface too*).

---

## 12. Community ("YesLifers")

The community is treated as a first-class, parallel build (5 workstreams, mostly "Week 1") [S5 → Community sheet]:
1. **Community Strategy & Positioning** → Community Charter, Member Guidelines, Value Proposition (vision, mission, target persona, rules/code of conduct) — Week 1.
2. **Branding & Identity** → Community Brand Kit (name, tagline, logo, WhatsApp banner, LinkedIn banner, visual identity) — Week 1.
3. **LinkedIn Marketing Strategy** → 30-Day LinkedIn Content Calendar (content pillars, posting schedule, launch campaign, founder story) — Week 1.
4. **Community Infrastructure** → WhatsApp Community + Discussion Group + Announcement Group + Admin Structure + Welcome Messages — Week 1.
5. **Community Growth** → first **100 founding members** (personal outreach, LinkedIn engagement, referrals, invite campaigns) — Weeks 2–6.

This dovetails with the product's **community-first on-ramp** [S1 → Chart 01], the **Pledge / "YesLifer" identity** [S1 → Chart 01; S2 → A01–A03], the **8-commitment pledge** [S2 → A02], **tribe feed + share moments** [S2 → B15/C09/D09/E07, F07], **community events/RSVP** [S2 → F08], and the use of **real WhatsApp conversation excerpts as sales proof** in the Hell-Yes teaser [S2 → E08].

---

## 13. Key Reference Links (verbatim) [S5 → Links sheet]
- **Technical Architecture:** `https://claude.ai/public/artifacts/27bcb80c-50a2-4662-ae51-44eed0a92fab` *(the source artifact behind `yesly_architecture.html`)*
- **Landing Page (live):** `https://yeslifers.vercel.app/`
- **Figma Designs (House of Alpha):** `https://www.figma.com/proto/YLrKQ3S9MEzmPtaI8rodMo/House-of-Alpha?node-id=3758-4640...`
- **Yesly Charter** — referenced as a linked doc (target not included here)
- **Yesly Logo Images** — referenced as a linked asset (target not included here)
- **Boardmix board (the phase-wise AA flow):** editor link `https://boardmix.com/app/editor/9UYebZQ2eZ3z1VTGtF_fCw?inviteCode=L8l5Ga` (login-gated) · **public share link** `https://boardmix.com/app/share/CAE.CMyBCyABKhD1Rh5tlDZ5nfPVVMa0X98LMAZAAQ/L8l5Ga` *(from S8; content mirrored in S2's AA Integrated Flow — see §0)*
- **Google-Sheets "UI Flow" (shared to Spinach):** `https://docs.google.com/spreadsheets/d/1uNKwIvB6VJ8TdpeZCcPgGitvFcwDWxn_AliXBrXHsD0/edit` *(byte-identical to local `Yesly - UI Workflow.xlsx`)* [S8 → Somil, 4 May]
- **Design/engineering partner:** Spinach Experience Design — `https://www.spinachexperience.design/` [S8]
- **AA real-world-state reference (cited in S2):** `https://casparser.in/blog/state-of-account-aggregator-2026/`

---

## 14. Summary — "What People Are Trying to Build Around Yesly"

Putting every source together, the effort has **six concurrent builds**:

1. **A conscious-money mobile app** for Indian earners 25–45, structured as a 4-step journey — *Clarity → Yes Life → Yes Budget → Hell Yes* — with a distinctive **6-persona "mirror"** before any data entry [S1 Doc 00; S2 Flows A–E; S3].
2. **A 4-tier monetisation funnel** (Free → Financial Plan → Investment Plan → Subscription → Human Advisory) with community-first acquisition [S1 Charts 01–04].
3. **An Account-Aggregator data layer** (Setu/Finvu) that auto-fills the financial picture, cutting first-insight time from ~15 to ~6 minutes, with honest handling of AA's real limits [S2 AA sheets].
4. **An internal financial-planning engine** (NWGC workbook → Python), owned by HoA, not the partner [S1 Doc 00 + Chart 09; S6].
5. **A SEBI-RIA + DPDP compliance spine** — immutable audit trail, anonymise-don't-delete, grievance/SCORES, two-layer KYC, a serious 7-surface admin console [S1 Charts 10–13].
6. **A YesLifers community** (WhatsApp + LinkedIn, 100 founding members) as the top of the funnel and the brand [S5 Community; S1 Charts 01–02].

The architecture is **done and validated**; the **conversational chat UI is confirmed as the build direction** (Figma prototype, §6B), with the **dock output-repository** and **Essential ₹2,999 / Growth ₹4,999** pricing now visible. The live decisions are **partner selection**, **final app content/flow** (incl. the as-built reorder in §6B), **Tier-3 pricing**, and the **vendor choices** (AA / KYC / WhatsApp / analytics) [S5 Workstream; S1 Open Decisions; S9].

---

### Open items
1. **Email (S8)** — ✅ received & integrated (§2A); Google-Sheet confirmed identical to S2.
2. **Boardmix board (S7)** — substance mirrored in S2/§6.2; public share link recorded in §0/§13; not separately rendered per your instruction.
3. **Client confirmations the email thread surfaces** (each affects scope/quote):
   - (a) ✅ **RESOLVED by the Figma prototype (S9/§6B):** the conversational UI **is being built as a real coded chat experience** (not video-led).
   - (b) ✅ **RESOLVED:** the floating-star repository is the **"dock"** (bottom-right, running count badge) — visible across the S9 screens (§6B).
   - (c) ⏳ **Still open:** reconcile **"basic admin panel" (Gaurav) vs the 7-surface console (S1 Chart 13)**.
   - (d) ⏳ **Still open:** confirm analytics vendor — **WebEngage (email) vs Mixpanel (S1)**.
   - (e) ◑ **Partly resolved (S9):** pricing is **₹2,999 Essential / ₹4,999 Growth** (₹249 stale); **Tier-3 subscription price still unconfirmed**.
   - (f) ⏳ **New, from S9:** capture the **life-event certainty/probability** field in the data model; confirm the **as-built reorder** (income/expenses after goals) is intentional vs a prototype artifact; and confirm **Google as the primary/only social-login** provider.

*Optionally also: (a) **cell-level formula** extraction from the NWGC engine (only the I/O schema is done, §6C — not the exact formulas), (b) the linked "Yesly Charter" / the full Figma file transcribed, or (c) Figma screens for the post-paywall journey (Steps 20–31 are currently spec-only).*

> **2026-06-26 update note.** Added **§6B** (19-screen Figma walk-through), **Steps 20–31** (post-paywall, spec-sourced), and **§6C** (engine output schema) — built by parsing S6 (engine), S2 (Flow E/F/G + AA), S1 (Charts 03–07/11), S3 (personas/scoring) and S4/S5 (post-paywall ordering). The end-to-end flow + per-step inputs/outputs + the deliverable schema are now covered; the remaining true gaps are post-paywall *visual design* and the open client decisions (§11.2, Open Items).
