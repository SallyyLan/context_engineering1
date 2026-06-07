# LLM Agent Instruction Architecture — Accurate Metric Retrieval via Natural Language

> Context engineering for an internal LLM-powered data agent. The goal: let non-technical marketing analysts get accurate numbers from complex data warehouses — just by asking questions in plain language.

---
## Project Context

Built as part of an internal AI tooling initiative to reduce analyst dependency on ad-hoc SQL requests. The instruction file was iteratively refined through failure analysis — each hard rule maps to a real error pattern observed in production queries.

This repository contains the **sanitized version** of the instruction file with company-specific identifiers removed.

---


## The Problem

The marketing analytics team needed answers from an 11-table data warehouse spanning user behavior, orders, vouchers, traffic, and seller data. The pain points:

- **Metric ambiguity** — terms like "active users", "orders", and "conversion rate" had multiple valid interpretations depending on context
- **Silent errors** — wrong joins, missing filters, and incorrect aggregation methods produced plausible-looking but wrong numbers with no warning
- **High cognitive load** — getting the right number required knowing which table, which filter flags, which date rules, and which edge cases applied
- **BI misalignment** — ad-hoc queries frequently diverged from dashboard figures due to missing exclusion flags or wrong metric formulas

The existing approach (ad-hoc SQL requests via Jira tickets) was slow, error-prone, and didn't scale.

---

## The Solution

A structured instruction file that governs how an LLM agent reasons through any data question — before touching any data.

The core mechanism is a **Dual-Track Protocol**:

```
TRACK A (Question Side)          TRACK B (Data Side)
─────────────────────────        ─────────────────────────
Parse business intent            Identify candidate tables
Resolve ambiguous terms          Verify schema live
Build calculation blueprint      Check business rules
Flag blockers early              Spot-check joins

              ↓ Both must complete ↓

                    MERGE STEP
         Match blueprint fields → verified schema
         Resolve any gaps
         Final pre-compute checklist

                    COMPUTE + ANSWER
         Execute with all filters applied
         Cite sources, declare confidence level
```

The two tracks run simultaneously. Neither can be skipped. No number is produced until both tracks complete and the merge passes.

---

## Key Design Decisions

**Ambiguity resolution before computation.** The agent is required to stop and ask one clarifying question if a business term can't be resolved — rather than guessing and producing a wrong number silently.

Getting the question right matters as much as getting the data right. The remaining decisions follow the same principle — every design choice exists to prevent a specific class of silent error:

- **BI-alignment as a hard constraint.** Specific filters and aggregation methods are mandatory on every relevant query. These are the most common sources of dashboard misalignment.
- **Voucher types are always distinct.** Four voucher types use different columns and different identification logic. The agent is never allowed to treat them as interchangeable.
- **Snapshot date rules are table-specific.** Each table has its own date rule, and the agent must apply the right one — omitting a date key on a time-partitioned table silently multiplies row counts.
- **Confidence is declared, not assumed.** Every answer is labeled HIGH, MEDIUM, or LOW based on whether all fields were schema-verified, all joins were spot-checked, and all filters were applied.

---

## Project Stages

**Stage 1 — Accurate Number Extraction** 

Ensuring the agent produces reliable, BI-aligned numbers from a single dataset in response to natural language questions. This means resolving ambiguous business terms, verifying schema live, applying the correct filters and aggregation methods, and declaring confidence explicitly on every answer.

**Stage 2 — Cross-Dataset Analysis** 

Extends the same dual-track approach to allow analysts to join above 5 datasets in a single natural language question. The goal is to support full campaign funnel analysis. For example: traffic → user segment → voucher engagement → order conversion.

---


## What's in Stage 1 File

| Section | What It Does |
|---|---|
| Dual-Track Protocol | The core reasoning loop — question decomposition + data grounding |
| Business Term Definitions | Canonical glossary translating natural language to exact fields and formulas |
| Bootstrap Schema Reference | Table map, join patterns, and BI-aligned metric formulas |
| Hard Rules | 24 explicit guardrails covering the most common error patterns |
| Confidence System | HIGH / MEDIUM / LOW output labels based on verification completeness |
| Escalation Path | When to stop and surface a gap rather than guess |


