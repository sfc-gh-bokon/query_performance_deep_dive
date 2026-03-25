# Query Performance Deep-Dive Skill

A Cortex Code skill that investigates slow Snowflake stored procedures starting from a single Query ID. It drills through parent → child → grandchild statements via SNOWHOUSE metadata to find the root cause bottleneck, classifies the anti-pattern, and produces optimization recommendations with SQL implementation and unit tests.

## What It Does

```
Query ID (or parameterized hash)
  → Resolve account (locator → deployment)
  → Profile execution history (runs, errors, variance, 30-day trend)
  → Drill into child statements (ranked by duration)
  → Generate visual execution tree (Mermaid diagram)
  → Check warehouse sizing and spill analysis
  → Recurse into nested SP calls (3-4 levels deep)
  → Classify anti-pattern with confidence scores
  → Assign severity badges and risk ratings
  → Check column datatypes (for cluster key recommendations)
  → Write deliverables:
      - Executive summary (1-page, Slack/email-ready)
      - Full findings report with before/after projections
      - Implementation SQL with rollback instructions
      - Unit tests (8 test types)
      - Monitoring task (automated regression alerts)
      - Re-run validation script
      - Jira/Confluence-ready checklist
      - Glossary of technical terms
```

## Prerequisites

- Cortex Code (CoCo) installed
- SNOWHOUSE access with SALES_ENGINEER role (or equivalent cross-account metadata access)
- A warehouse for SNOWHOUSE queries (e.g., SE_WH)
- A customer account locator and a Query ID or parameterized hash to investigate

## Installation

Extract the zip into one of these locations:

```bash
# Global (available in all projects)
unzip query-performance-deep-dive.zip -d ~/.snowflake/cortex/skills/

# Project-level (available in current project only)
unzip query-performance-deep-dive.zip -d .cortex/skills/
```

Run `/skill` in Cortex Code to verify it appears.

## Usage

Invoke with `$query-performance-deep-dive` followed by your request:

```
$query-performance-deep-dive Investigate query 01c2db65-0002-e6c9-0001-46fa5738b8ce on account QY87826

$query-performance-deep-dive Why is this SP slow? Hash 59b92827681f15b9fa7547a12dfa49b6 on OI3PRODAZURECUS

$query-performance-deep-dive Profile the slowest execution of CALL OMNIABOR.CIGActgH.spSOI_Cnsldtd on account QY87826
```

The skill will pause at key decision points for your input before proceeding.

## Workflow (7 Steps)

| Step | Action | Stopping Point? |
|------|--------|-----------------|
| 1 | Resolve account locator → account_id + deployment | |
| 2 | Profile query execution history + 30-day duration trend with trend classification (DEGRADING/STABLE/INTERMITTENT) | ✋ Confirm which execution to drill into |
| 3 | List child statements ranked by duration, generate execution tree diagram, check warehouse sizing/spill | |
| 4 | Recursively drill into nested SP calls (if children are CALLs), detect cursor loops | |
| 5 | Classify bottleneck with confidence scores (HIGH/MEDIUM/LOW), severity badges ([CRITICAL]/[WARNING]/[INFO]), and risk ratings (SAFE/LOW/MODERATE) | ✋ Present root cause findings |
| 6 | Check column datatypes for cluster key truncation rules | |
| 7 | Write full deliverables package (see Output section) | ✋ Final review |

## Anti-Patterns Detected

| # | Pattern | Severity | Risk to Fix | Example Signal |
|---|---------|----------|-------------|----------------|
| 1 | Row-at-a-time cursor loop | [CRITICAL] P0 | MODERATE | 1000+ children, same hash, 1 row each |
| 2 | Missing temp table statistics | [CRITICAL] P0 | SAFE | >5 broadcast joins after temp table populate |
| 3 | Join explosion / fan-out | [CRITICAL] P0 | MODERATE | Billions inserted → DISTINCT to thousands |
| 4 | Serial UPDATE chain | [WARNING] P1 | LOW | 10-30 UPDATEs on same table, different columns |
| 5 | CDC DELETE on unclustered table | [CRITICAL] P0 | SAFE | DELETE scans full large table |
| 6 | Serial execution of independent work | [WARNING] P1 | MODERATE | Many independent statements run sequentially |
| 7 | Repeated identical join (N x 1 calls) | [WARNING] P1 | LOW | N identical CALLs differing only by filter value |
| 8 | MERGE on unclustered target table | [CRITICAL] P0 | SAFE | MERGE full table scan, partition pruning ratio near 100% |

## Output Files

The skill produces two files per investigation:

| File | Contents |
|------|----------|
| `<SP_NAME>_performance_recommendations.html` | Self-contained styled HTML with dark theme, Mermaid execution tree (rendered via CDN), stat grid cards, colored severity badges, TL;DR (Slack/email-ready), one-page executive summary, 30-day duration trend, before/after comparison, findings with confidence scores, numbered recommendations with priority/risk/confidence, Jira-ready checklist, warehouse sizing assessment, glossary. Print-friendly. |
| `<SP_NAME>_optimization_commands.sql` | Self-contained SQL blocks with USE context + ROLLBACK instructions, unit tests (8 types), monitoring task (automated weekly regression alert), re-run validation script (before/after metric comparison) |

## Features

### Customer Digestibility
- **Mermaid Execution Tree**: Visual diagram showing where time is spent, color-coded by severity (red >30%, orange 10-30%, green <10%)
- **Severity Badges**: [CRITICAL] / [WARNING] / [INFO] for instant triage
- **One-Page Executive Summary**: Top 3 actions with risk ratings and expected improvement
- **Before/After Comparison**: Step-by-step projected timeline showing current vs. estimated post-fix durations
- **Glossary**: Plain-English definitions of spill, broadcast join, fan-out, parameterized hash, etc.

### Actionability
- **Copy-Paste SQL**: Every recommendation is a self-contained block with `USE WAREHOUSE/DATABASE/SCHEMA` and `ROLLBACK` instructions
- **Jira/Confluence Checklist**: Markdown checklist (`- [ ] P0 Rec 1: ...`) ready to paste into project tracking tools
- **Risk Ratings**: SAFE (apply immediately) / LOW (verify with tests) / MODERATE (requires testing cycle)
- **Monitoring Task**: Automated weekly Snowflake Task that emails alerts if performance regresses

### Analysis Quality
- **30-Day Trending**: Duration sparkline with trend classification (DEGRADING / STABLE-SLOW / INTERMITTENT)
- **Warehouse Sizing Check**: Spill-to-scan ratio analysis with size-up recommendations
- **Confidence Scores**: HIGH (confirmed) / MEDIUM (likely) / LOW (needs more data) based on metric strength
- **Pattern Library Links**: Each anti-pattern links to Snowflake docs and community articles for deeper learning

### Validation
- **8-Type Unit Test Suite**: Row count, anti-join (both directions), duplicates, column comparison, diagnostics, performance threshold, multi-parameter
- **Re-Run Validation Script**: Re-runs SP post-fix, captures new query ID, compares before/after metrics automatically
- **Slack/Email Summary**: 3-sentence plain-English summary formatted for pasting into customer communications

## File Structure

```
query-performance-deep-dive/
├── SKILL.md                          # Main workflow (7 steps + all enhancements)
├── README.md                         # This file
└── references/
    ├── snowhouse-queries.md          # SNOWHOUSE view reference, STATS patterns, new query templates
    └── anti-patterns.md              # 8 anti-patterns with severity badges, confidence guidance, risk ratings, related reading
```

Output files per investigation:
```
<output_dir>/
├── <SP_NAME>_performance_recommendations.html   # Styled HTML report (open in browser)
└── <SP_NAME>_optimization_commands.sql           # Implementation SQL + tests + monitoring
```

## Key Technical Details

- **Data source:** `SNOWHOUSE_IMPORT_SHARE_DB.<DEPLOYMENT>.JOB_ETL_V` for query history
- **Drill-down technique:** `PARENT_JOB_ID` links parent → children; `QUERY_PARAMETERIZED_HASH` identifies recurring patterns and enables targeted JOB_ID lookup for nested SPs
- **STATS access:** `j.STATS:stats:scanBytes::NUMBER`, `hashJoinNumberBroadcastDecisions`, `ioLocalTempWriteBytes`, etc.
- **DATA_TYPE_ENCODED:** VARCHAR containing JSON (not variant) — determines cluster key truncation rules (TIMESTAMP_NTZ/LTZ/TZ needs `TO_DATE(col)`)
- **Zero-copy CLONE:** Used for test baselines instead of CREATE TABLE AS SELECT — instant, no extra storage, preserves metadata
