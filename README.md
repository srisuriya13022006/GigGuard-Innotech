# GigGuard 🛡️
### AI-Powered Parametric Income Protection for Swiggy Delivery Partners

> **One Line:** GigGuard automatically detects external disruptions in a rider's zone, estimates income loss using AI, validates the claim, and sends a payout to UPI — with zero manual filing.

---

## How It Works — 3 Steps

```
1. Rider subscribes weekly  →  AI calculates a personalised premium
2. System monitors disruptions every 15 minutes (rain, heat, AQI, closures)
3. Threshold crossed → AI estimates loss → fraud check → instant UPI payout
```

| Stat | Value |
|------|-------|
| Entry premium | Rs. 29 / week |
| Trigger to payout | < 8 minutes |
| Manual claim forms | 0 |
| AI modules in pipeline | 4 |

---

## Who Is GigGuard Built For?

**Persona — Priya, Chennai, 28**

| Attribute | Detail |
|-----------|--------|
| Platform | Swiggy delivery partner |
| Zone | Velachery (primary) · T Nagar (secondary) |
| Schedule | 6 days/week · Lunch + Dinner slots |
| Weekly income | Rs. 5,500 — fully outdoor, no sick leave |
| Core problem | 2-hr rain event = Rs. 400–600 lost. No compensation. No insurance. |
| Insurance today | None — traditional insurers need employment proof gig workers can't provide |
| Tech profile | Android · UPI · WhatsApp-native |

---

## Priya's End-to-End Journey

**Step 1 — Onboarding (4 min, one-time)**  
Priya enters zone, schedule, earnings, UPI ID. Module A runs immediately.

**Step 2 — AI Risk Assessment (Module A)**  
Score: **74/100 — HIGH RISK.** Reasons: Velachery waterlogging + 48 hrs/week outdoor + Dinner peak overlap. Shown in plain language.

**Step 3 — Premium + Policy (Module B)**  
Recommended: **Standard Plan — Rs. 39/week — up to Rs. 1,200 protected.** Priya pays via UPI. Policy activates instantly.

**Step 4 — Trigger Detected**  
Tuesday 7:15 PM — 48mm rainfall in Velachery over 2.5 hrs. Threshold exceeded. Claim auto-initiated for all active policy holders in zone.

**Step 5 — Impact Estimation (Module C)**  
Zone match ✔ · Dinner slot overlap ✔ · Peak multiplier 1.5× · Estimated loss = Rs. 160/hr × 2.5 hrs × 0.83 × 1.5 = Rs. 498, capped at Rs. 400 (Standard plan).

**Step 6 — Fraud Validation (Module D)**  
8 rule checks run. Zone match ✔ · App active 6:45–8:20 PM ✔ · No duplicate ✔ · Zone stable 45 days ✔. Isolation Forest anomaly score: low. **Fraud Risk Score: 14/100 — VERIFIED.**

**Step 7 — Payout Released**  
Rs. 320 sent to Priya's UPI. Push notification sent. **Total time: under 8 minutes.**

---

## Parametric Triggers

| Trigger | Threshold | Data Source |
|---------|-----------|-------------|
| Heavy Rain | > 30 mm/hr for 1+ hr | IMD / OpenWeatherMap |
| Extreme Heat | > 40°C for 3+ hrs in active slot | IMD / weather station |
| Severe AQI | AQI > 300 for 2+ hrs | CPCB / AQI India API |
| Zone Closure | Official closure in rider's zone | Municipal API / admin |

> A Python background service polls APIs every 15 minutes. When a threshold is crossed, Module C estimates impact, then Module D validates before payout. Phase 1 uses mock data for demo.

---

## 4-Module AI Pipeline

```
Onboarding → [Module A] → [Module B] → Policy Active
                                              ↓
                                     Trigger Detected
                                              ↓
                                        [Module C]
                                              ↓
                                        [Module D]
                                              ↓
                                       Payout Released
```

---

### Module A — Rider Risk Profiling

**Question:** How exposed is this rider to income-loss disruptions?  
**Model:** Random Forest (interpretable, works well with synthetic + IMD historical data)

**Inputs:**
- Zone, earnings, working days/hours, active slots, work type
- Zone risk indices: rain frequency, heat risk, AQI risk, closure frequency

**Engineered features:** `peak_hour_overlap` · `outdoor_work_intensity` · `disruption_exposure_score`

**Output:** Risk score 0–100 + Low/Moderate/High category + top 3 plain-language risk factors

```json
{ "risk_score": 74, "risk_category": "High",
  "top_factors": ["High rainfall exposure", "Evening shift overlap", "High weekly outdoor hours"] }
```

---

### Module B — Weekly Premium Prediction

**Question:** How much should this rider pay per week?  
**Model:** XGBoost Regressor + business rule constraints

**Formula:**
```
Final = Round(RiskWeight × EarningsFactor × DisruptionProbability × CoverageMultiplier) + RiskMargin
Floor: Rs. 29/week  |  Ceiling: Rs. 250/week
```

| Tier (Risk Score) | Weekly Premium | Protected Income |
|-------------------|---------------|-----------------|
| Basic (0–40) | Rs. 29 / week | Up to Rs. 800 / week |
| Standard (41–70) | Rs. 39 / week | Up to Rs. 1,200 / week |
| Plus (71–100) | Rs. 49 / week | Up to Rs. 1,500 / week |

> AI estimates the base premium. Business rules round to a clean price and apply floor/ceiling caps.

---

### Module C — Disruption Loss Estimation

**Question:** How much income did this disruption actually cost this rider?  
**Model:** XGBoost Regressor + threshold classifier for impact label

**Formula:**
```
Loss = HourlyEarnings × OverlapHours × SeverityWeight × PeakMultiplier

Severity:        Low = 0.4  |  Moderate = 0.7  |  High = 1.0
Peak Multiplier: Lunch / Dinner slot = 1.5×  |  Off-peak = 1.0×
```

**Why this matters:** Without Module C, a 3 AM rain and a 7 PM dinner-hour downpour pay identically. Module C prevents this by checking schedule overlap.

**5-step overlap check:**
1. Zone match — does trigger zone == rider's operating zone?
2. Active day check — does rider normally work this day?
3. Work slot overlap — did the trigger hit an active delivery slot?
4. Peak slot detection — Lunch or Dinner? Apply 1.5× multiplier
5. Duration × Severity → final estimated loss

**Output:**
```json
{ "estimated_income_loss": 320, "impact_level": "High",
  "payout_recommendation": "Partial payout",
  "impact_factors": ["Dinner slot overlap", "High rain intensity", "2.5 hour duration"] }
```

---

### Module D — Fraud Detection & Validation

**Question:** Is this payout event genuine or suspicious?  
**Architecture:** Hybrid — 8 deterministic rules + Isolation Forest anomaly detection

**Fraud risk score formula:**
```
fraud_risk = (rule_flag_score × 0.6) + (ml_anomaly_score × 0.4)

0–30   → VERIFIED   → Auto-payout proceeds
31–60  → REVIEW     → Hold — soft re-validation
61+    → FLAGGED    → Block — escalate to admin
```

**8 Rule Checks:**

| Check | PASS | FAIL Impact |
|-------|------|-------------|
| Zone match | Claim zone == onboarding zone | +30 to score |
| Activity log | Platform app active during trigger window | +25 to score |
| Intent match | Rain claim + declared rain willingness | +25 to score |
| Work slot overlap | Trigger time within active slots | +20 to score |
| Zone recency | Zone unchanged for 7+ days | +20 to score |
| Duplicate claim | First and only claim for this event | Instant reject |
| Active policy | Valid policy for this week | Instant reject |
| Claim frequency | ≤ 3 claims in 7 days | Hold for review |

**ML layer (Isolation Forest):** Trained on normal rider behavior — no labelled fraud examples needed in Phase 1. Catches hidden patterns: unusual claim frequency, behavioral drift, suspicious zone switches.

```json
{ "fraud_risk_score": 82, "status": "Flagged", "decision": "Manual review",
  "fraud_flags": ["Zone mismatch", "No platform activity", "Zone updated 3 days ago"] }
```

---

## Market Crash Scenario

**Scenario:** Swiggy order volume in Chennai drops 60% in 48 hours. No weather event fires. Income collapses.

| Phase | Response |
|-------|----------|
| Phase 1 | **Zone Closure trigger** covers government shutdowns, Section 144, civic restrictions automatically |
| Phase 2 | **Platform Order Density Drop** — zone-level order volume drops > 40% vs 30-day rolling average triggers payout |
| Phase 3 | **Mass Disruption Event** — admin activates city-wide payout for all active policy holders when a declared emergency is issued |

> GigGuard is trigger-extensible. Adding a new disruption type requires only: a threshold definition + a new data source + a severity weight in Module C. The payout engine and fraud layer need zero changes.

---

## Adversarial Defense & Anti-Spoofing

GigGuard does not trust raw GPS as proof of disruption exposure. GPS can be spoofed — but consistent work pattern, zone stability, activity continuity, and multi-rider behavioral signals cannot all be faked simultaneously.

### Genuine Rider vs Spoofing Attacker

| Genuine Rider Shows | Spoofing Attacker Shows |
|---------------------|------------------------|
| Stable zone matching onboarding history | Sudden zone appearance without prior history |
| Normal work-slot alignment | Trigger-zone presence with no app activity |
| App session continuity near disruption | Claims only during payout-trigger events |
| No burst of simultaneous suspicious claims | Coordinated behavior across multiple riders |

### Signals Validated Beyond GPS

- **Onboarding baseline** — declared zone, slots, earnings (trusted reference)
- **Zone stability history** — consistent operating area over time
- **App activity logs** — platform session presence during trigger window
- **Slot overlap** — disruption time vs declared active hours
- **Claim behavior** — frequency, duplicates, cross-zone repetition
- **Cohort anomaly** — cluster of riders suddenly appearing in same zone

### Fraud Ring Detection

GigGuard also asks: **"Does this group of riders look artificially coordinated?"**

Signals: mass zone appearance without history · synchronized claim timing · identical event response signatures · rapid enroll-and-claim patterns on new policies.

### Tiered Response (Protecting Honest Riders)

| Risk Level | Action | Rider Experience |
|------------|--------|-----------------|
| Low (0–30) | Auto-approve | Payout sent immediately |
| Medium (31–60) | Brief hold — soft re-validation | Most honest riders still pass |
| High (61+) | Hold for review | Clear message — no harsh rejection |

> A rider facing network drop in a rainstorm should not lose protection because one signal is missing. GigGuard uses fallback confidence from zone history and activity patterns before any adverse decision.

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Mobile app | Flutter — iOS + Android, single codebase, native push + UPI |
| Module A, B | scikit-learn: XGBoost / Random Forest |
| Module C | XGBoost Regressor + threshold classifier |
| Module D | scikit-learn: Isolation Forest (unsupervised — no fraud labels needed) |
| API backend | FastAPI (Python) — one endpoint per AI module |
| Database | MongoDB — flexible schema for rider profiles + claim documents |
| Auth + real-time | Firebase Auth (phone OTP) + Firestore (live claim state sync) |
| Trigger monitor | Python background service — polls APIs every 15 minutes |
| Payout | UPI integration (Phase 2) — mocked in Phase 1 demo |

---

## Phase 1 — What Is Built

**Scope:** 8 Flutter app screens · all 4 AI modules running inference on synthetic data · rule-based fraud engine + Isolation Forest · simulated rain trigger demo · MongoDB + Firebase deployed · UPI payout mocked.

| Built in Phase 1 | Planned for Phase 2 / 3 |
|-----------------|------------------------|
| 8 Flutter screens, end-to-end navigable | Live IMD / weather API trigger |
| All 4 AI modules running real inference | Real Swiggy activity log integration |
| Rule-based fraud engine + Isolation Forest | Live UPI payout disbursement |
| MongoDB + Firebase deployed | GPS spoofing detection (Phase 3) |
| Simulated rain event demo | Platform Order Density Drop trigger (Phase 2) |

---

## App Pages — Rider Flow

```
Welcome → Onboarding → AI Risk Summary → Policy Confirmation → Worker Dashboard
    → Trigger Monitor → Trigger Detected → Fraud Validation → Payout Status
```

| Page | Purpose |
|------|---------|
| Welcome | Introduce GigGuard — income protection, not health/accident insurance |
| Onboarding | Collect zone, schedule, earnings, payout method (hybrid API + user input) |
| AI Risk Summary | Show risk score, recommended plan, pricing explanation |
| Policy Confirmation | Review and activate weekly plan |
| Worker Dashboard | Live protection status, earnings card, risk widgets |
| Trigger Monitor | Real-time disruption tracking per zone |
| Trigger Detected | Auto-claim initiation with event details |
| Fraud Validation | Behavioral baseline check results |
| Payout Status | Approved amount, UPI transfer reference |

**Admin flow:** Admin Login → Dashboard (portfolio KPIs) → Trigger Analytics → Claims + Fraud Review

---

## Onboarding — Hybrid Data Model

GigGuard uses a hybrid approach — auto-fetch from Swiggy API where possible, ask user only for what the API cannot provide.

**Auto-fetched from API:** name · mobile (verified) · city · primary zone · work type · earnings history · activity logs · hotspots

**User must enter:** preferred delivery slots · rain/heat/pollution willingness · confirm weekly earnings · preferred protection level · UPI ID · 3 consent checkboxes

> **Why intent fields matter:** A rider who declares they don't work in rain cannot later claim a rain payout — this forms the fraud behavioral baseline from day one.

---

## Phase Roadmap

| Phase | Key Deliverables |
|-------|-----------------|
| Phase 1 — March 2026 | Flutter app (8 screens) + 4 AI modules on synthetic data + simulated trigger demo + README + 2-min video |
| Phase 2 — Real Data | Live IMD trigger + Swiggy activity logs + UPI payout + model retraining + Platform Order Density Drop |
| Phase 3 — Advanced | GPS spoof detection + weather API cross-reference + fraud ring graph analytics + Mass Disruption Event + multi-platform |

---

## Honest Limitations — Phase 1

- **Data:** Models trained on synthetic profiles + public IMD historical data. Accuracy improves with real rider-event data in Phase 2.
- **Triggers:** Demo uses a simulated rain event. Live API integration is Phase 2.
- **Payout:** UPI is mocked in Phase 1. Real disbursement is Phase 2.
- **GPS spoofing:** Phase 1 uses zone match + activity log checks. Full GPS sequence analysis is Phase 3.
- **Market Crash trigger:** Zone Closure covers civic disruptions now. Platform Order Density Drop requires Swiggy API access — Phase 2.

---

*GigGuard — Phase 1 | March 2026 | Swiggy Delivery Partner Persona*