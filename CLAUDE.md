# CenSai — MVP Prototype Context

## What This Is

CenSai is the AI-native consumption intelligence system pitched in Hunter's Salesforce panel presentation. This directory is a working prototype of the codex architecture — the four-pillar system (triage workflows, metric definitions, endorsed tables, transformation definitions) built out as YAML files.

The prototype uses fictional but structurally realistic data (see `../spoof-data/`) to demonstrate how the system would actually function. All table paths reference a fictional Snowflake schema (`SFDC_CONSUMPTION.*`), not real Salesforce infrastructure.

**Purpose of this directory:**
1. Demonstrate system depth during panel Q&A — every pillar in the pitch is actually built out here
2. Serve as a reference spec if CenSai gets built for real
3. Test or extend the prototype architecture as needed

See `../CLAUDE.md` for full interview context, presentation framework, and Hunter's background.

---

## File Structure

```
censai/
├── semantic/                        # Metric definitions
│   ├── ccr.yaml                     # Consumption Coverage Ratio
│   ├── abr.yaml                     # Activation Breadth Rate
│   ├── uts.yaml                     # Usage Trajectory Score
│   ├── ers.yaml                     # Expansion Readiness Score (derived from CCR/UTS/ABR)
│   ├── projected-month-end-ccr.yaml
│   ├── projected-ccr-in-2-weeks.yaml
│   └── days-since-last-event.yaml
├── endorsement/                     # Snowflake table schemas
│   ├── tbl-consumption-events.yaml
│   ├── tbl-account-entitlements.yaml
│   ├── tbl-user-activation.yaml
│   ├── tbl-opportunities.yaml
│   ├── tbl-platform-event-log.yaml
│   ├── tbl-ccr-benchmarks.yaml
│   ├── tbl-activation-benchmarks.yaml
│   ├── vw-account-details.yaml
│   ├── vw-billing-calendar.yaml
│   └── dim-date.yaml
├── routing/
│   └── intent-router.yaml           # Intent classification + routing logic
└── triages/
    ├── last-week-performance.yaml
    ├── forward-looking.yaml
    └── expansion-qualification.yaml
```

---

## Semantic Layer — Metrics

### Core Metrics

| Metric | File | Grain | What It Measures |
|--------|------|-------|-----------------|
| CCR | ccr.yaml | account-week | MTD consumption vs. pace-adjusted contracted commitment |
| ABR | abr.yaml | account-week | % of enabled users active in trailing 30 days |
| UTS | uts.yaml | account-week | 4-week rolling avg WoW consumption growth % |
| ERS | ers.yaml | account-week | Composite expansion readiness score (derived from CCR/UTS/ABR) |

### Derived / Projection Metrics

| Metric | File | What It Calculates |
|--------|------|--------------------|
| Projected Month-End CCR | projected-month-end-ccr.yaml | Daily run rate extrapolated to billing month-end |
| Projected CCR in 2 Weeks | projected-ccr-in-2-weeks.yaml | 14-day horizon projection; collapses to month-end when horizon exceeds days remaining |
| Days Since Last Event | days-since-last-event.yaml | Point-in-time dormancy flag |

### Metric Thresholds

**CCR (Consumption Coverage Ratio)**
- Overage: > 100%
- Healthy: 80–100%
- Watch: 60–79%
- At-risk: < 60%

**ABR (Activation Breadth Rate)**
- Healthy: ≥ 60%
- Watch: 35–59%
- At-risk: < 35%

**UTS (Usage Trajectory Score)**
- Accelerating: ≥ +10%
- Healthy: +5% to +9%
- Watch: -4% to +4%
- Declining: -5% to -14%
- Critical: ≤ -15%
- *Early-ramp override (< 8 weeks since first event): watch threshold relaxed to -10%; declining < -10%*

**ERS (Expansion Readiness Score)** — weighted composite, scale 1–10
- Weights: CCR × 0.4 + UTS-normalized × 0.3 + ABR × 0.3
- CCR capped at 100% for normalization (overage triggers separate intervention, not expansion)
- Expansion ready: ≥ 8.0, sustained 2+ consecutive weeks
- Approaching: 6.0–7.9
- Not ready: < 6.0

**Days Since Last Consumption Event**
- Active: 0–6 days
- Dormant: ≥ 7 days (forward-looking triage entry condition)
- Stale: ≥ 14 days
- Inactive: ≥ 30 days

---

## Endorsement Layer — Tables

All queries must use endorsed tables. Free-form queries requiring a non-endorsed table should surface the gap rather than querying.

| Table | Schema Path | Grain | Primary Use |
|-------|-------------|-------|-------------|
| TBL_CONSUMPTION_EVENTS | SFDC_CONSUMPTION.CORE | One row per event | Fact table for all consumption metrics |
| TBL_ACCOUNT_ENTITLEMENTS | SFDC_CONSUMPTION.CORE | Account per contract | Contracted units, tier, billing cycle |
| TBL_USER_ACTIVATION | SFDC_CONSUMPTION.CORE | User per account | Provisioning status, last activity (for ABR) |
| VW_ACCOUNT_DETAILS | SALESFORCE_CRM.PUBLIC | Account | Segment, tier, AE/CSM, renewal date |
| VW_BILLING_CALENDAR | SFDC_CONSUMPTION.CORE | Account per month | Days elapsed/remaining for pace math |
| TBL_CCR_BENCHMARKS | SFDC_CONSUMPTION.PLANNING | Segment + tier + period | CCR targets by account profile |
| TBL_ACTIVATION_BENCHMARKS | SFDC_CONSUMPTION.PLANNING | Segment + period | ABR floors by segment |
| TBL_PLATFORM_EVENT_LOG | SFDC_CONSUMPTION.CORE | Platform event | Outages/degradations for root cause triage |
| TBL_OPPORTUNITIES | SALESFORCE_CRM.PUBLIC | Opportunity | Active expansion flag for triage |
| DIM_DATE | SHARED.DW | Calendar date | Week/month/fiscal period grain |

**Key conventions across all queries:**
- Always filter `ad.is_test_account = FALSE` from VW_ACCOUNT_DETAILS
- CCR pace math uses `VW_BILLING_CALENDAR` — never compute days elapsed manually
- ABR excludes `user_type != 'HUMAN'` to avoid inflating with service/API accounts
- Active contract filter: `ent.is_active = TRUE` on TBL_ACCOUNT_ENTITLEMENTS

---

## Routing Layer

**File:** `routing/intent-router.yaml`

| Mode | Trigger | Behavior |
|------|---------|----------|
| `triage` | Semantic intent matches a pre-defined trigger phrase | Routes to one of three triage workflows |
| `account_deep_dive` | Account name or ID appears in the question | Runs all three triages scoped to that account, synthesizes output |
| `free_form` | No triage trigger matched (fallback) | LLM constructs Snowflake query from endorsed tables + semantic metrics |
| `metadata_logging` | All free-form questions | Logged to `SFDC_CONSUMPTION.CENSAI.TBL_QUESTION_LOG` for roadmap review |

**Triage intent samples:**
- Last-week performance: "how did we do", "what happened last week", "why did [account] drop", "root cause"
- Forward-looking: "how are we tracking", "any accounts at risk", "what do I need to know this week"
- Expansion qualification: "who is ready to expand", "expansion pipeline", "who should the AE be talking to"

**Free-form guardrails:**
- Only query endorsed tables; surface the gap if a needed table is not endorsed
- Apply all `required_filters` from the relevant endorsement file
- Use metric calculations exactly as defined in the semantic layer — no improvisation
- Log the question text regardless of outcome

---

## Triage Workflows

### 1. Last-Week Performance (`triages/last-week-performance.yaml`)
**Entry condition:** CCR WoW change ≥ ±5pp  
**Purpose:** Root cause analysis for CCR movements — distinguish platform issues, adoption disruptions, structural declines, and overages  
**Key terminals:**
- `adoption_disruption` (P1) — ABR dropped >10pp alongside CCR; engagement-level issue, not a data problem
- `platform_cause` (P2) — platform event log entry explains the timing
- `structural_failure` (P1) — multi-week CCR decline + negative UTS; systemic, not episodic
- `active_overage` (P1) — CCR crossed 100%; triggers commercial conversation
- `healthy_ramp` — CCR increase consistent with normal ramp trajectory, no action needed

### 2. Forward-Looking Risk & Opportunity (`triages/forward-looking.yaml`)
**Entry condition:** Runs for all active accounts weekly  
**Purpose:** Project month-end CCR at current trajectory; surface both risk and opportunity before they materialize  
**Key terminals:**
- `at_risk_with_window` (P1) — projected miss but 3+ weeks remaining; intervention window still open
- `high_probability_miss` (P1) — ≤2 weeks remaining, trajectory negative; window closed, set expectations
- `pre_overage_window` (P1) — projected to cross 100% CCR within 2 weeks; optimal proactive tier conversation
- `strong_expansion_candidate` (P2) — sustained high UTS (3+ weeks above +15%) + ERS ≥ 8.0
- `accelerating_narrow` (P2) — high UTS but ABR < 40%; expansion premature, broaden adoption first
- `consumption_gap` (P1) — zero events in 7 days; immediate data/engagement triage
- `portfolio_clean` — no accounts flagged; proceed to expansion qualification

### 3. Expansion Qualification (`triages/expansion-qualification.yaml`)
**Entry condition:** Runs for all active accounts weekly  
**Purpose:** Generate tiered expansion queue with per-account barrier diagnosis  
**Key terminals:**
- `forced_expansion` (P1) — CCR already > 100%; commercial conversation is the only resolution
- `unworked_opportunity` (P1) — ERS ≥ 8.0 sustained, no active expansion opp in CRM
- `ccr_barrier` (P3) — CCR not supporting expansion; focus on consumption recovery first
- `trajectory_barrier` (P2) — CCR ok but UTS declining; expansion pitch premature
- `breadth_barrier` (P2) — CCR/UTS ok but ABR < 40%; adoption too narrow for expansion to land

---

## Spoof Data

**Location:** `../spoof-data/`  
**Scripts:** `generate.py` (creates all 11 CSVs), `verify.py` (validates join integrity and story point presence)

### Test Accounts

| Account | Segment | Tier | Profile |
|---------|---------|------|---------|
| ACC-001 — Acme Corp | Enterprise | Enterprise | Stable healthy consumption ~850 units/week. Active expansion in CRM (NEGOTIATION, $250K). AE: Sarah Chen. |
| ACC-002 — TechStart Inc | Commercial | Growth | Intentional ramp: starts at 30% capacity, reaches 100% around week 30. Renewal in PROPOSAL ($140K). AE: Marcus Johnson. |
| ACC-003 — Global Retail Co | Enterprise | Enterprise | Intentional decay starting week 20; accelerates from week 20–26. EMEA region. At-risk renewal in DISCOVERY ($200K). Prior expansion CLOSED_LOST. AE: Sarah Chen. |
| ACC-004 — MidCo Systems | Commercial | Growth | Stable mid-tier ~420 units/week. Expansion just opened in DISCOVERY ($110K). AE: Marcus Johnson. |
| ACC-005 — StartupXYZ | SMB | Starter | Sporadic: 35% probability of zero units per week. Single user (Kate). Renewal in PROPOSAL ($22K). AE: Emily Rodriguez. |

### Embedded Story Points

- **PLT-001 (2025-05-19):** ACC-003 EMEA API gateway latency — SEV2, 2-day degradation (week ~18). Explains a temporary dip before the structural decay onset.
- **PLT-002 (2025-06-04):** ACC-003 Agentforce runtime outage — SEV2, 2 days (week ~21). Coincides exactly with decay onset; last-week-performance triage should route to `platform_cause`, not `structural_failure`.
- **PLT-003/004/005:** Global maintenance windows (Feb 15, Aug 9, Nov 8) — SEV3, short planned windows, no account-level impact story.

### Benchmark Attribution
Both CCR and ABR benchmarks are attributed to "Sri Vasudevan" — intentional detail for demo realism.

### Running the Scripts
```
cd spoof-data
python generate.py   # Writes all 11 CSVs to spoof-data/
python verify.py     # Validates joins, benchmark coverage, and story point integrity
```

---

## Design Decisions Worth Noting

- **CCR is pace-adjusted**, not raw MTD units. Comparing raw consumption on day 5 vs. day 25 of a billing cycle is meaningless — pace-adjustment is what makes the metric actionable on any day of the month.
- **ERS caps CCR at 100%** for normalization. Overage is a separate commercial intervention; it doesn't make an account a stronger expansion candidate — it makes them a different kind of urgent.
- **ABR uses trailing 30 days**, not the current week. ABR can move without any new consumption events this week — the window rolls daily.
- **UTS requires 2+ weeks of data** to be valid. Accounts with fewer contributing weeks should be flagged as provisional, not classified as at-risk by default.
- **Free-form questions are logged** — this is the metadata loop from the pitch. Recurring questions surface triage candidates and drive the CenSai roadmap.
- **Benchmarks are set by segment + tier** (CCR) or segment alone (ABR). New segments or tiers may not have a benchmark row yet — handle NULLs explicitly rather than defaulting to zero.
