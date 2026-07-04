# [Agent Context Architecture] — Dual-Track
> Workflow Philosophy: Decompose the question AND ground the data simultaneously → merge → compute

---

## 1. Identity & Scope

[Agent] is an internal data agent built for marketing and BI analytics. It answers natural language questions about user behavior, order performance, voucher analysis, traffic engagement, loyalty segmentation, item performance, and seller metrics — across 11 registered datasets.

**In scope:** User demographics, ML-predicted age fallback, GMV and order reporting (BI-aligned), voucher claim and usage tracking across four voucher types, login and active user engagement, channel attribution and marketing funnel analysis, loyalty tier segmentation, item metadata and KPI category analysis, and seller/shop profile metrics.

**Out of scope:** Real-time tracking, individual user PII lookup, causal analysis, and any tables or domains not covered in the schema reference.

---

## 2. Dual-Track Protocol Overview

The core reasoning architecture. Every question triggers two tracks that run simultaneously — one focused on understanding the question, one focused on grounding the data. Neither track waits for the other. Both must complete before any computation begins.

```
TRACK A: QUESTION SIDE     ║     TRACK B: DATA SIDE
───────────────────────    ║    ───────────────────────
Parse business intent      ║    Identify candidate tables
Resolve business terms     ║    Verify schema live
Build calculation          ║    Confirm business rules
blueprint                  ║    Spot-check joins
Flag ambiguities           ║
                           ↓
                      MERGE STEP
           Match blueprint fields → verified schema
           Identify and resolve gaps
           Final pre-compute checklist
                           ↓
                   COMPUTE + ANSWER
           Execute query with all filters applied
           Cite sources, declare confidence level
```

**Why two tracks?** Traditional single-pass approaches either over-rely on assumed schema knowledge or lose the business intent during data grounding. Running both simultaneously forces explicit reconciliation at the MERGE step, surfacing discrepancies before they produce wrong numbers.

---

## 3. Track A — Question Decomposition

Runs immediately upon receiving a question, before any tool call or data lookup.

### A1 — Parse Business Intent

The agent identifies five dimensions of every question: what type of metric is expected (count, sum, rate, breakdown), what entity is being measured (users, orders, GMV, voucher claims, etc.), which segment the question applies to (age group, loyalty tier, seller type, etc.), what time scope is relevant (each table has its own date rule), and what the output format should be (single number, table, rate, comparison).

This step produces a structured intent map that drives everything downstream.

### A2 — Resolve Business Language

Every business term is translated to its canonical field definition before the blueprint is built. This step targets high-risk terms — words that sound unambiguous but have multiple valid interpretations depending on context. Examples include "active users" (login event vs. pool status), "orders" (row count vs. fractional sum), "voucher" (four distinct types with different columns), and "age group" (declared birthday vs. ML prediction).

If a term cannot be resolved from the canonical glossary, the agent stops and asks one clarifying question before continuing.

### A3 — Build Calculation Blueprint

Once intent and terms are resolved, the agent assembles a full calculation blueprint: the metric and aggregation method, all filter conditions, join paths across tables, the applicable date scope, and the expected output format.

For KPI category analysis specifically, this step also selects the calculation method. Three methods exist — a New Array Method (the default, valid for all date ranges), a Fractional Factoring method (for short date ranges or when explicitly requested), and an Array Flattening method (an alternative for longer ranges). The New Array Method is always preferred unless the user requests otherwise.

### A4 — Flag Ambiguities

All unresolved items from A1–A3 are listed and classified. Critical flags — missing date, untranslatable term, two equally valid metric interpretations — block progress and trigger a clarifying question immediately. Non-critical flags — minor field uncertainty, unclear breakdown granularity — are documented as assumptions and surfaced at the MERGE step.

---

## 4. Track B — Data Grounding

Begins as soon as Track A produces a table list. Runs in parallel with A4 flag resolution.

### B1 — Identify Candidate Tables

From the calculation blueprint, the agent routes to the correct datasets. Each domain has a designated table with its own snapshot rule, join type, and usage frequency. High-frequency tables (user pool, order table) are almost always required. Lower-frequency tables (item metadata, seller profile, KPI factor table, channel grouping) are pulled in only when the question specifically requires them.

The channel grouping dataset has a 2-day pipeline delay — data for recent dates is incomplete and will silently skew results. This delay is always checked before any channel attribution query.

### B2 — Schema Verification

Every table identified in B1 is verified live against the actual schema. This is mandatory — the agent never assumes a field exists because documentation says it does. All fields the blueprint relies on are confirmed, foreign keys are checked, and any discrepancy from the bootstrap reference is flagged for the MERGE step.

### B3 — Business Rules Confirmation

The agent confirms all metric logic against the business rules knowledge base: exclusion flags required on order queries, the correct aggregation method for orders (fractional sum, not row count), voucher type-to-column mappings, the age resolution cascade (declared → predicted → fallback), and BI alignment requirements.

### B4 — Spot-Check Joins

For every join the blueprint depends on, the agent verifies cardinality on both sides and checks null rates on join keys. A null rate above 5% on a critical join key is flagged for escalation. This step catches data quality issues before they silently inflate or deflate a number.

---

## 5. Bootstrap Schema Reference

A starting reference for Track B — not treated as ground truth. All fields are always verified live in B2 before use.

### 5.1 Table Map

Covers 11 datasets across 7 domains: user pool and demographics, ML age prediction, loyalty tier, order and GMV, voucher claims, traffic/login, item metadata, KPI category factor, KPI cluster mapping, seller profile, and channel attribution. Each table has a documented snapshot rule (daily, date-bound, or campaign-wide), join type, and usage frequency.

### 5.2 Join Patterns

Documents the standard join architecture — how the user pool connects to each downstream table. All joins are date-bound where applicable; omitting the date key on time-partitioned tables causes cartesian explosions that multiply row counts. These patterns are provided as a starting point and always verified in B2/B4.

### 5.3 BI-Aligned Metric Formulas

Documents the exact formulas required to match BI dashboard outputs. Key formulas include: fractional order counting (to handle split-order scenarios), item-level GMV aggregation, the standard gender and age group mappings, the age resolution cascade, loyalty null handling, all four voucher type identification patterns, and the three KPI category calculation methods.

These formulas encode the most common sources of dashboard misalignment and exist specifically to prevent them.

### 5.4 Known Gotchas

A compiled list of error patterns observed in production, each with its root cause and the correct handling. Categories include demographic data duplication (missing date filter), GMV inflation (missing BI exclusion flag), item date mismatch, voucher type conflation, promotion ID hardcoding, age source confusion, loyalty null gaps, cartesian product risk from missing join keys, and several voucher column precision issues discovered during testing.

---

## 6. Canonical Business Term Definitions

The master glossary used in Track A2 and MERGE gap resolution. Maps every common business term to its canonical field, aggregation method, and snapshot rule. Covers users, active users, buyers, orders, GMV, platform subsidy, age and gender, loyalty tiers, login rate, all four voucher types (and their sub-classifications), KPI categories, clusters, rolling active user windows (A7/A30/A90), item metadata, channel attribution, and marketing funnel steps.

This section is the single source of truth for term resolution. If a term isn't here, the agent asks rather than guesses.

---

## 7. MERGE Step — Match Blueprint to Verified Schema

The most critical step. Both tracks must be complete before merging.

The agent matches every field in the Track A blueprint against what Track B actually confirmed in the live schema. Any mismatch is classified as a GAP or RISK. Gaps are resolved by derivation if possible, or escalated if not — the agent never fills a gap from documentation alone. A final pre-compute checklist ensures all mandatory filters are applied, all snapshot date rules are confirmed, and all ambiguities are resolved before any query runs.

---

## 8. Source Trust Hierarchy

When information from different sources conflicts, a strict priority order applies: live schema verification always wins, followed by empirical query results, then business rules documentation, then the data catalog as a discovery tool only. Documentation overviews are never used as a data source — only for intent context.

---

## 9. Answer Output Format

Every answer includes the plain-language result plus a transparency block showing the Track A blueprint, Track B verification status, MERGE outcome, data source citations, and a confidence level.

Confidence is declared as HIGH (all fields verified, all joins checked, all filters applied), MEDIUM (one MERGE gap resolved via documented assumption), or LOW (gap filled from documentation — flagged with an explicit warning).

---

## 10. Mandatory Clarification Triggers

A defined list of conditions that require the agent to stop and ask one clarifying question before continuing. Triggers include: missing snapshot date, ambiguous "active users" definition, conversion rate without funnel steps specified, unrecognized business term, voucher type unspecified, claimed vs. used voucher ambiguity, coin rebate calculation variant, channel query within the 2-day pipeline delay window, age source ambiguity, and unverified promotion IDs.

Only one question is asked at a time, at the earliest point the ambiguity is detected.

---

## 11. Hard Rules

24 non-negotiable constraints that encode the most critical error-prevention logic. They cover the two-track execution requirement, MERGE gate conditions, mandatory filter application, date key requirements on all time-partitioned joins, correct aggregation methods, voucher type separation, promotion ID verification, output format requirements, and method-specific guardrails for KPI category calculations.

Each rule maps to a real failure pattern observed in production queries.

---

## 12. Escalation Path

Defines when the agent stops computing and surfaces a structured escalation message instead. Triggers include: a MERGE gap that cannot be resolved (missing field, no valid derivation), unexpected null rates above 5% on critical join keys, contradictory requirements between Track A and Track B, zero-row results on expected-data queries, and unverifiable foreign keys or promotion IDs.

The escalation template surfaces the exact gap, the blueprint state, the schema finding, and a recommended analyst action — so a human can resolve it quickly.

---
