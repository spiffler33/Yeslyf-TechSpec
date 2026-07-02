# Yesly Flow — Consolidated Change Log from the 30-Jun Meetings

**Baseline document:** `Yesly_Flow.html` (presented version, 29-Jun) — 31 steps, 4+1 tiers, 6 persona cards, NWGC engine breakdown, 19-entity data model, admin console (7 surfaces / 4 roles), 10 open decisions.

**Inputs reviewed (all in `Meet_30_Jun/`):**

| Source | Who | What it covers |
|---|---|---|
| 11:30 AM meeting (transcript + summary) | Gaurav, Vatsal, Somil, Kajal | Screen-by-screen review of the flow doc vs Figma; UI corrections; compliance debate |
| 3 PM meeting (transcript + summary) | Bhuvanaa, Harish, Kajal, Somil, Vatsal | Paywall gating decisions, expense taxonomy, portfolio recommendation framework |
| `talks.txt` (dev-agency call) | Eshani, Saquib, Sahil (agency) + Vatsal | What the developers need added to this doc; martech/admin architecture questions |
| `Investment Schema.docx` | Harish | The Tier-2/3 asset-allocation engine: factors, archetypes, allocations, rebalancing |

Legend: ✅ decided · 🔁 changed/reversed during the day · ⚠ open — needs a decision or follow-up work.

---

## 1. Flow & screen-order changes

1. ✅ **Login moves before everything.** Current order is Splash → Bhuvanaa's video → pledge/questions. Gaurav: login should come first, then splash/video. (Kajal relayed and Bhuvanaa accepted at 3 PM.)
2. ✅ **Free-tier sequence confirmed:** three opening questions → avatar → money-journey → family → assets & liabilities → **Clarity Picture (free)**. The free Clarity Picture surfaces only **3 of 7 features** (Figma phrases it as 3 of 15 insights); the rest unlock on payment. This teaser step exists in the Figma but was **missing from the flow doc** — add it explicitly.
3. 🔁 **Family tree / member-level mapping moves post-paywall.** Biggest structural change of the day. Pre-paywall we now collect only **family composition**: who is in the family, how many dependents, their ages — no per-member financials. Detailed member-wise assets/income/family-tree mapping happens only after payment.
4. ✅ **New up-front question: "Is this for you, or you as a family?"** (Harish's resolution to the you-vs-family debate). Every subsequent input (income, assets) is then interpreted at that declared level — "your / your family's income". Post-paywall, the user is prompted to confirm whether entered numbers were personal or family-level, with **bifurcation screens** offered if they want to split them member-wise. If they decline, the plan proceeds on family-level data. The plan is expected to change as post-paywall data (spouse/parent assets via AA) arrives — that is fine.
5. ✅ **Figma slide 16 is missing from the flow doc** ("Add things you don't want to miss out on" — add life events). The doc currently jumps from "Let's make it real" (slide 18 in doc / Figma 15) straight to income. Insert it and re-verify the doc-step ↔ Figma-slide numbering end to end.
6. ✅ **RIA agreement step inserted at the free→paid transition** — between "Start with Essentials" and the first paid screen. It is currently missing from the flow. (See §7 for what the 3 PM meeting decided about *when* it's legally required.)
7. ⚠ **Missing path fork at "Start with Essentials".** If the user does *not* start with Essentials, there is no branch — the flow dead-ends. Also, once the user selects Essentials the subsequent screens must stop being labelled "free". Both need fixing in flow + Figma.
8. ✅ **Two more reveal screens** in the Clarity sequence: reveal 1 = "here's what life looks like now", reveal 2 = **projections**, reveal 3 = **blind spots** (Figma slide 13 has the details). Doc currently under-represents this.

## 2. UI / UX changes (to feed to Ishtiyak, and to add as a "UI/UX note" column in the tech-spec doc)

1. ✅ **All assets & liabilities inputs are sliders with ranges — never free-text.** Rules agreed:
   - Step increments must be sensible for the range (e.g. 0–1 Cr moving in lakh-steps, not ₹1 steps) — units in **lakhs/crores**.
   - **No "I don't know" option** — user either has the asset class or doesn't; instead add an **"Other"** bucket for anything not listed.
   - Exact ranges/steps: ⚠ to be specified. Gaurav's explicit position: do **not** assume "developers will figure it out" — designers/spec must state it. (Vatsal to add these as spec notes.)
2. ✅ **Projections screen: replace text with graphs** — three graphs (current path vs better allocation vs full potential), improved with Ishtiyak. Currently three text blocks.
3. ✅ **One combined downloadable PDF**, not three separate PDFs, spanning the three clarity screens. (Kajal confirmed the design is already one PDF; the doc implied three.)
4. 🔁 **Per-member data becomes card-per-person, not tabs:**
   - **Income** (Step 19): split into one card per earning member — "we've got your data, now we need Anjali's" — take-home, spouse, rental, other. Requires family composition to be captured **before** the income screens (see §1.3).
   - **Insurance covers** (who's-protected screen): also per-member cards.
   - **Expenses stay family-level** (confirmed by Somil — the planning tool never captures expenses per member).
   - General rule: anything captured per-user = multiple cards, one per user; and family/member data entry must be **uniform** (Step 18 is tab-based today vs per-member elsewhere — pick one pattern).
5. 🔁 **HLV question rewording — proposed, then overruled.** Morning (Gaurav): compute HLV from data already collected and ask "Based on your data your HLV is ₹X — do you have cover of that much?" with a "not available" option. **3 PM (Bhuvanaa): rejected — HLV is a paid output and must not be revealed pre-paywall.** Final: keep the insurance questions input-only before payment; the HLV number, required-vs-existing cover and the gap appear only in the paid financial plan.
6. ✅ **Goals:**
   - **Remove the "Who's saying yes to this?" owner field** — all goals are family-level goals; the ownership data was never stored or used.
   - Goal capture needs clearer inputs: **"When do you see this happening?"** (tenure) + **rough budget** per goal.
   - **Date ranges must be sensible per goal type**: a 25–35-year-old's child-education goal is ~20 years out; home can be 8–25 years; retirement needs an **actual number**, not "10+ years".
   - Goal maths is a simple client-side **future-value calculation**; the only server fetch is the inflation rate per goal type (education ~10%, home ~5–6%). No NWGC engine call needed at this step — the doc currently implies an engine step here; mark it "simple maths compute for FV".
7. ✅ **Answer-button behaviour** (the "earning well" selection screens): on tap the selected option highlights/bolds — the duplicated response button below disappears. Instead of echoing after every question, add a Claude-style **conversational summary after the question series** ("So you're X, a bit worried about Y — shall we proceed?"). Feedback goes to Ishtiyak; Ishtiyak's consistency concern noted but overruled by Bhuvanaa/Harish.
8. ✅ **Expense taxonomy — stick to the original input sheet.** Ishtiyak invented categories; revert. Fixes:
   - "**Rent expense**" label is wrong → rename (it's fixed commitments: rent, EMIs, school fees…).
   - **Groceries/fuel/utilities are variable necessities**, not fixed commitments (they fluctuate month to month).
   - Recreation/entertainment and annual items (dining, travel, experiences, hobbies) confirmed present.
   - Everything collected **monthly**.
   - Don't expand to more categories pre-paywall (detailed expense capture is post-paywall anyway, per §3) — the current four-category granularity plus the original sheet's fields is enough to produce the outputs. "Inputs were designed for the outputs — collect exactly those."
9. ✅ **Add a visible flowchart** to the flow doc/UI: a clickable flow diagram that tracks where the current screen sits, shows Yes/No branching, and lets reviewers/developers jump around. (Gaurav: "100% — this doc is going to developers, work with that assumption.")

## 3. Paywall & data-gating rules (3 PM decisions — Bhuvanaa's rulings)

1. ✅ **Principle: paywall sits before outputs, not before data collection.** But respect the user's *time cost* — long, granular questioning is deferred until after payment when the user has an incentive to answer.
2. ✅ **Safe to collect pre-paywall:** the 3 opening questions, avatar, number/ages of dependents, combined (self-or-family) income, combined assets & liabilities as slider ranges, goals/life events, insurance yes/no facts.
3. ✅ **Deferred to post-paywall:** per-member breakdowns, family-tree mapping, detailed expense capture, AA-sourced portfolio pulls, anything "time-costly".
4. ✅ **No calculated output is shown free** — HLV, surplus, gap analyses, plan numbers are all paid. Free tier gets only the partial Clarity Picture (3 of 7 features). "A calculation is worth what you charge for it."
5. ✅ **The plan itself is only generated post-paywall** — free users get a clarity picture, not a plan. Post-paywall the plan recalculates as richer data arrives.

## 4. Compliance changes

1. 🔁 **The morning's blocker was resolved at 3 PM by Harish:**
   - **Data collection does NOT require a signed agreement.** The client isn't yet classified as RIA-track or distribution-track (MFD), so nothing needs signing at data-collection time. Risk-profiling to suggest schemes can live on the MFD track.
   - **The signed RIA agreement is required only before regulated investment advice is delivered** — i.e., at/inside the paid investment-advice tier, not at the free data-collection stage.
   - This supersedes Kajal's morning reading ("agreement before any data collection") and kills Gaurav's cost concern about paying stamp/e-sign fees for users who never pay.
2. ⚠ **Signing mechanics still open:** is email-OTP acknowledgement an acceptable "signature", or is a paid digital signature required? Compliance to define exact triggers and acceptable methods; **engage a lawyer** ("here is where a good lawyer can help us"). Kajal owns getting the compliance/legal opinion.
3. ✅ **No AI in planning logic** — explicitly reaffirmed (Bhuvanaa asked, Vatsal answered): every input→output must be mathematical, deterministic and auditable for SEBI/RBI comfort. Worth stating in the doc's compliance section.
4. Impact on the doc: the existing **Decision 5 (KYC trigger tier)** should be updated with this RIA-agreement clarification; add the agreement-signing step to the flow (§1.6) tagged "method TBD by compliance".

## 5. Investment engine — new section to add (Harish's Investment Schema, presented 3 PM)

This is the Tier-2/3 recommendation engine the doc currently shows only as a green "our system" placeholder. The framework:

**Basic inputs** (all already captured in the flow): age · monthly income · monthly committed expenses · investment horizon (Short <3y / Medium 3–7y / Long 7y+) · self-declared risk comfort · goals/life events.

**Five internal factors** computed from inputs:

| Factor | Meaning |
|---|---|
| SIR — Surplus Income Ratio | Savings as % of income; drives how aggressive the model can be. Existing wealth folds in. |
| LPS — Liquidity Preference Score | How much must stay liquid; function of age, emergency funds (62-yr-old ≫ 25-yr-old) |
| THC — Time Horizon Cluster | Short/medium/long, derived from goals & life events |
| RDI — Risk Disposition Index | Stated risk appetite *adjusted down* by the data (low surplus or 2-yr horizon ⇒ lower RDI) |
| TES — Tax Efficiency Score | Optional/debated for v1 — Harish leans toward including it for the mass-retail audience |

**Six archetypes** (Harish expects more; the last is open-ended):

| Archetype | Profile | Primary need | Equity | Debt | Alt/Gold | Liquid |
|---|---|---|---|---|---|---|
| The Accumulator | High SIR, Low LPS, Long THC | Aggressive wealth building | 70–80% | 10–20% | 5–8% | 2–5% |
| The Balancer | Moderate SIR/LPS, Mixed THC | Balanced growth + safety | 45–55% | 25–35% | 8–12% | 8–12% |
| The Protector | Any SIR, High LPS, Short THC | Capital preservation + liquidity | 20–30% | 35–45% | 3–7% | 25–35% |
| The Tax Optimizer | High TES, high bracket | Tax-efficient growth (NPS, ELSS, debt) | 45–55% | 25–35% | 3–7% | 10–15% |
| The Life Eventer | Specific 2–5-yr goal (home, child) | Goal-specific earmarking | 35–45% | 30–40% | 3–7% | 15–25% |
| The Course Corrector | Chaotic existing portfolio | Restructuring + rationalisation | 40–50% | 25–35% | 8–12% | 10–15% |

Ranges are base weights; **RDI tilts within the range**. Harish is not convinced Tax Optimizer should be a standalone archetype (it emerged as an output; may instead be a flag on the other archetypes).

**Curated product shelf, tagged on 5 axes** (asset class · liquidity T+0→lock-in · tax treatment · min investment · risk band 1–5): Equity-Growth (large-cap index, flexi-cap active, ELSS, mid-cap index), Debt-Stable (PPF, NPS Tier I, corporate bond, short-duration), Liquid-Buffer (liquid MF, overnight, arbitrage, penalty-free FD), Alternatives-Hedge (Gold ETF/SGB, REITs, international MF), Goal-Specific (Sukanya Samriddhi, capital-protection FoF, target-date hybrids). Within a bucket the product split is **standardised, not per-client** (e.g. within 35% liquid: first 10% → X, next 10% → Y).

**Rebalancing engine — exactly two triggers:**
1. **Client-level / drift:** cash-flow calendar (time-to-goal shrinking) + time-based strategic-allocation rebalance (half-yearly/annual) with profit-booking when drift occurs (e.g. equity 50→60 after a rally).
2. **Market-regime overlay:** Nifty PE/PB vs 15-yr historical percentile (90th pct ⇒ reduce equity tilt) · India VIX level (defensive vs neutral tilt) · a 3-state macro flag (expansion/neutral/contraction). Factors not final.

**Exact outputs:** model portfolio allocation **with product break-up** → existing portfolio (via Account Aggregator / NSDL CAS — *not* an input to the model portfolio) → **action plan**: what to redeem, what to reinvest, delivered as tasks + nudges. Instruments can be **earmarked to goals** (the "car in 6 months" example) so users don't redeem them.

⚠ **Open on the engine:** number of archetypes; the scoring algorithm behind the factors; product-level segregation logic (subjective inputs like "already holds direct equity"); TES in/out; market-regime factor selection. **Harish will supply input/output specs (not algorithm code); Vatsal to define desired output formats to inform the data model and to brief Ishtiyak on the output screens.**

## 6. Tech-spec doc & architecture changes (dev-agency call — `talks.txt`)

What the developers (Saquib, Sahil, Eshani) need before sign-off:

1. ⚠ **Make every "External API" (brown) chip clickable**, showing exactly which fields are pulled from that API, in what structure (JSON), and **mapped into the internal data-model tables**. Today the chips are unlabelled placeholders and the data model is internal-only. Vatsal owns this; it's the gate for the "core project is ready" milestone.
2. ⚠ **Tier-3 asset-allocation inputs/outputs are not in the doc at all** — add them (content now exists: §5 above). Vatsal's own estimate: doc is at ~80% once external APIs + Tier 3 are in.
3. ✅ **"Marketing website" in the technical architecture is a documentation error** — there is one app/website; what was meant is the martech stack (CRM, Twilio/MoEngage-type tooling) attached to admin. Remove/correct in `yesly_architecture.html`.
4. ✅ **Admin console = advisory console = one console** (plus the martech surface), desktop-accessible React/Next app, distinct from the React Native mobile app:
   - *Advisory part* — analyst role: maintains the profile→allocation→product mappings (the §5 engine's parameters, which keep changing).
   - *Admin part* — support, refunds, complaints, compliance reviewer, admin engineer.
   - *Martech part* — the new third leg (below).
5. ⚠ **Martech / CRM is a newly-opened workstream** ("we haven't thought deeply about this at all"):
   - Need an **omni-channel lead manager** (SFMC vs NetCore discussed) covering prospects (cold) vs leads (warm), including off-app leads (LinkedIn, newsletter).
   - Define the action set: nudges, notifications, emails, discounts — CRM APIs hitting the app for "next best action".
   - Decide where app data lands (raw DB vs CRM vs CMS) — avoid duplicate databases for similar activities.
   - **Plan agreed:** finish external-API mapping first → then daily workshops with the agency (order: admin panel → advisor panel → marketing/martech). Friday checkpoint: agency presents their understanding doc rebuilt on top of the flow doc; Monday onward daily "clarity" calls.
6. ✅ **Update the technical architecture doc after this doc is finished** so architecture + flow doc + Figma stay congruent — agency flagged they're "one step ahead" of the current architecture diagram.
7. ✅ **Deliverable boundary restated:** HoA supplies this doc + Figma + calculation sheets/Python; the **asset-allocation algorithm's actual calculation comes later** (non-trivial), but its inputs/outputs and DB schema come now so frontend/backend can proceed. Database schema will be deep (financial-security classes etc.) — agency may rename keys as they see fit.
8. ⚠ **Prototype persistence:** the HTML prototype "saves" only in the presenter's local browser (GitHub Pages, no backend). Decide GitHub Pages vs Firebase/Supabase for anything demo-shared. (Flagged 11:30 AM.)
9. ✅ **Doc-as-single-source-of-truth confirmed** as the working method: this doc = Figma = New User Flow spreadsheet, plus persona cards, admin roles, all assumptions, and **A/B-testing controls** (e.g. swapping the 3 opening questions, tweaking calculations from the admin tab) documented in one place.

## 7. New logic/content to be designed (owners assigned)

| Item | What's needed | Owner |
|---|---|---|
| **Key concerns array** | Full catalogue of possible key concerns + a formula that picks which apply (e.g. skip liquidity concern if liquid assets sufficient) | Bhuvanaa + Somil |
| **"What success looks like" logic** | Derived from opening Q3 (success definition) + collected numbers; logic schema to substantiate with figures | Bhuvanaa + Somil |
| Slider ranges & steps | Exact ranges, increments, units for every A&L slider | Spec (Vatsal) — *not* left to developers |
| Engine input/output spec | Formal I/O for financial-planning, investment-plan, asset-allocation engines | Harish → Vatsal |
| UI/UX notes column | Every change above tagged in the doc as a "UI/UX note" for Ishtiyak | Vatsal |
| Compliance/legal opinion | Signature triggers + acceptable methods (OTP vs e-sign) | Kajal |

## 8. Still open after 30-Jun (add to the doc's Open Decisions section)

1. Which designer opinion prevails on the answer-button/summary pattern once Ishtiyak responds (Bhuvanaa's call stands for now).
2. Exact slider ranges/step increments (per §2.1).
3. OTP vs digital signature for the RIA agreement; lawyer to be found.
4. Number of archetypes + scoring algorithm + TES inclusion (§5).
5. Product-level segregation rules for subjective holdings.
6. Martech stack choice (SFMC vs NetCore vs other) and the whole where-does-data-live diagram.
7. Backend persistence for the prototype (GitHub Pages vs Firebase/Supabase).
8. Paywall gating rules for *specific* calculations/screens (agreed in principle §3; per-screen list pending).
9. The doc's pre-existing 10 open decisions (AA vendor, KYC vendor, WhatsApp BSP, Python/Node, KYC tier, Tier-3 pricing, billing cadence, advisor-console hosting, scheduling vendor, refund window) — none were closed on 30-Jun; Decision 5 (KYC/RIA trigger) is now partially informed by §4.1.

---

### Quick "who said what" on the two reversals

- **HLV pre-paywall reveal:** proposed by Gaurav (11:30) → **rejected by Bhuvanaa (3 PM)** — outputs stay paid.
- **Agreement-before-data-collection:** asserted by Kajal (11:30, per her compliance reading) → **resolved by Harish (3 PM)** — collection is free of signature; only RIA advice delivery requires the signed agreement.

*Note: `Yesly_Flow_updated.html` and `Yesly_Flow_v2.html` (both dated 30-Jun) exist alongside the presented `Yesly_Flow.html`; some 11:30 AM notes (e.g. "all goals are family level") were typed into the doc live during that meeting and may already be reflected there.*
