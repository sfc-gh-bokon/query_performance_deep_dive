# Anti-Pattern Detection & Fix Catalog

## How to Use This Catalog

Each pattern below includes:
- **Severity badge**: `[CRITICAL]` / `[WARNING]` / `[INFO]` — maps to P0/P1/P2
- **Detection criteria**: Specific metrics to look for in SNOWHOUSE data
- **Confidence guidance**: How to determine HIGH/MEDIUM/LOW confidence
- **Risk rating**: How risky the fix is to implement
- **Related reading**: Links to Snowflake documentation and community resources

---

## Pattern 1: Row-at-a-Time Loop (Cursor) [CRITICAL]

**Detection:**
- Hundreds/thousands of children with same `QUERY_PARAMETERIZED_HASH`
- Each inserts 1 row, scans identical bytes (e.g., 0.76 GB x 2,953 = 2,240 GB)

**Confidence guidance:**
- HIGH: >100 iterations, same hash, 1 row each — unmistakable
- MEDIUM: 20-100 iterations — could be legitimate batch processing
- LOW: <20 iterations — might be intentional parameterized calls

**Fix:** Single set-based INSERT with `WHERE col BETWEEN :START AND :END`
**Risk:** MODERATE — logic change, requires testing cycle
**Impact:** 99%+ reduction (thousands of seconds → single digits)
**Severity:** P0 CRITICAL

**Related Reading:**
- [Snowflake Best Practices: Avoid Row-by-Row Processing](https://docs.snowflake.com/en/user-guide/stored-procedures-best-practices)
- [Snowflake Community: Converting Cursors to Set-Based SQL](https://community.snowflake.com/s/article/Performance-Optimization-Replacing-Cursors-with-Set-Based-SQL)
- [Snowflake Docs: INSERT ... SELECT](https://docs.snowflake.com/en/sql-reference/sql/insert)

---

## Pattern 2: Complex Join Graph — Optimizer Build/Probe Suboptimality [CRITICAL]

**Root cause:** Snowflake maintains accurate micro-partition metadata (row counts, min/max values) immediately after any DML, so the optimizer *does* have statistics for freshly populated temp tables. The problem is structural: in deeply nested CTEs and complex multi-way JOIN graphs, the optimizer can assign suboptimal build/probe sides to hash joins — choosing to broadcast a large table when it should use the small driving set as the build side. Breaking the join into explicit materialized steps forces the optimizer to plan each step against a simple, single-table input rather than trying to reason through a multi-way join graph all at once.

**Detection:**
- `hashJoinNumberBroadcastDecisions` > 5
- Scan disproportionate to output rows
- Deep CTE or multi-table JOIN in the bottleneck statement

**Confidence guidance:**
- HIGH: >5 broadcasts AND complex CTE/multi-table join — definitive
- MEDIUM: 2-5 broadcasts — could be other causes
- LOW: 1 broadcast — normal optimizer behavior in some cases

**Fix:** Break the problematic join into explicit steps: materialize the intermediate driving set into a narrow temp table first, then join from that known-size table. This forces the optimizer to plan each step independently with a simple input rather than a complex graph.
**Risk:** SAFE — CTAS/CREATE TEMP TABLE steps are additive; output is functionally identical
**Impact:** 50-90% scan reduction
**Severity:** P0

**Related Reading:**
- [Snowflake Docs: Query Optimization — Join Strategies](https://docs.snowflake.com/en/user-guide/performance-query)
- [Snowflake Community: Temporary Table Statistics Collection](https://community.snowflake.com/s/article/How-to-force-statistics-collection-on-temp-tables)
- [Snowflake Docs: Understanding the Query Profile](https://docs.snowflake.com/en/user-guide/ui-query-profile)

---

## Pattern 3: Join Explosion / Fan-Out [CRITICAL]

**Detection:**
- `numRowsInserted` in billions, followed by DISTINCT reducing to thousands
- Fan-out ratio > 1000:1
- Massive spill (>50 GB) and network shuffle

**Confidence guidance:**
- HIGH: Fan-out ratio >100:1 with clear DISTINCT downstream — unmistakable
- MEDIUM: Fan-out ratio 10-100:1 — may be intentional for analytics
- LOW: Fan-out ratio 2-10:1 — within normal range for some patterns

**Fix:** Staged dedup joins — break multi-way JOIN into sequential steps, DISTINCT after each
**Risk:** MODERATE — structural logic change, requires testing cycle
**Impact:** 90-99% reduction
**Severity:** P0

**Related Reading:**
- [Snowflake Docs: Optimizing Joins](https://docs.snowflake.com/en/user-guide/performance-query#optimizing-joins)
- [Snowflake Community: Handling Cartesian Products and Fan-Outs](https://community.snowflake.com/s/article/Understanding-and-Resolving-Cartesian-Products)
- [Snowflake Docs: Query Profile — Join Explosions](https://docs.snowflake.com/en/user-guide/ui-query-profile)

---

## Pattern 4: Serial UPDATE Chain [WARNING]

**Detection:**
- 10-30+ UPDATEs on same temp table, each updating different columns
- Each scans full table; similar duration

**Confidence guidance:**
- HIGH: >10 UPDATEs on same table, each different column — clear pattern
- MEDIUM: 5-10 UPDATEs — could be intentional staged processing
- LOW: 2-4 UPDATEs — might be necessary for different join keys

**Fix:** Merge UPDATEs using same join key into single statement with LEFT JOINs. For identical join patterns differing only by filter value, use pivoted `MAX(CASE WHEN filter = 'X' THEN value END)`
**Risk:** LOW — equivalent logic, verify with unit tests
**Impact:** 30-60% reduction (or 10x for pivoted pattern)
**Severity:** P1

**Related Reading:**
- [Snowflake Docs: UPDATE](https://docs.snowflake.com/en/sql-reference/sql/update)
- [Snowflake Docs: MERGE](https://docs.snowflake.com/en/sql-reference/sql/merge)
- [Snowflake Community: Consolidating Multiple UPDATEs](https://community.snowflake.com/s/article/Optimizing-Multiple-UPDATE-Statements)

---

## Pattern 5: CDC DELETE on Unclustered Table [CRITICAL]

**Detection:**
- DELETE scans entire large table
- DELETE uses specific ID/batch key as filter
- No cluster key on table

**Confidence guidance:**
- HIGH: Full table scan on DELETE with selective filter — definitive
- MEDIUM: Large scan but filter selectivity unclear
- LOW: Table is small (<1M rows) — clustering won't help much

**Fix:** `ALTER TABLE t CLUSTER BY (delete_predicate_column)`
**Risk:** SAFE — additive change, no logic modification
**Impact:** 70-90% DELETE time reduction
**Severity:** P0 if table is large and SP runs frequently

**Related Reading:**
- [Snowflake Docs: Clustering Keys](https://docs.snowflake.com/en/user-guide/tables-clustering-keys)
- [Snowflake Docs: Automatic Clustering](https://docs.snowflake.com/en/user-guide/tables-auto-reclustering)
- [Snowflake Community: Cluster Key Selection Strategies](https://community.snowflake.com/s/article/How-to-Choose-Clustering-Keys)

---

## Pattern 6: Serial Execution of Independent Work [WARNING]

**Detection:**
- Many child SPs/statements with no data dependencies run sequentially
- Each 5-20 sec, but 20-30+ of them
- Total = sum (no parallelism)

**Confidence guidance:**
- HIGH: Independent tables/schemas being populated sequentially — clear
- MEDIUM: Apparent independence but unclear if ordering matters
- LOW: Could have hidden dependencies (e.g., via views)

**Fix:** Parallelize via Tasks, async execution, or batch consolidation
**Risk:** MODERATE — requires app-layer changes, dependency analysis needed
**Impact:** Wall-clock / parallelism factor
**Severity:** P1 (requires app-layer changes)

**Related Reading:**
- [Snowflake Docs: Tasks](https://docs.snowflake.com/en/user-guide/tasks-intro)
- [Snowflake Docs: Task DAGs](https://docs.snowflake.com/en/user-guide/tasks-graphs)
- [Snowflake Community: Parallelizing Stored Procedures](https://community.snowflake.com/s/article/How-to-Parallelize-Stored-Procedures-with-Tasks)

---

## Pattern 7: Repeated Identical Join (N x 1 Serial Calls) [WARNING]

**Detection:**
- N identical CALL children to same helper SP
- Each scans same tables, same temporal predicates
- Differs only by a filter value (e.g., scheme name, category)

**Confidence guidance:**
- HIGH: >5 identical CALLs with only filter value differing — unmistakable
- MEDIUM: 3-5 calls — may be intentional for isolation
- LOW: 2 calls — likely intentional

**Fix:** Single pivoted UPDATE with `MAX(CASE WHEN filter = 'X' THEN value END)` and `GROUP BY`
**Risk:** LOW — equivalent logic, verify with unit tests
**Impact:** Nx speedup (e.g., 12 calls → 1 call = ~10x faster)
**Severity:** P1

**Related Reading:**
- [Snowflake Docs: Conditional Expressions (CASE)](https://docs.snowflake.com/en/sql-reference/functions/case)
- [Snowflake Docs: PIVOT](https://docs.snowflake.com/en/sql-reference/constructs/pivot)
- [Snowflake Community: Replacing Repeated Queries with Pivots](https://community.snowflake.com/s/article/Using-PIVOT-to-Consolidate-Repeated-Queries)

---

---

## Pattern 8: MERGE on Unclustered Target Table [CRITICAL]

**Detection:**
- MERGE scans entire target table (scanBytes ≈ total table size) despite selective ON-clause filter
- `numPartitionsScanned / numPartitionsTotal` ratio near 1.0 (no partition pruning)
- Target table has no clustering key (`CLUSTER_BY_KEYS` is empty)
- ON clause includes a temporal filter (e.g., `t.CRT_TMSTMP >= ...`) that could enable pruning if clustered

**Confidence guidance:**
- HIGH: Full table scan on MERGE with selective temporal filter AND no cluster key — definitive
- MEDIUM: Full scan but filter selectivity unclear (e.g., ID-based ON clause)
- LOW: Table is small (<1 GB) — clustering overhead may not be justified

**Fix:** `ALTER TABLE <target> CLUSTER BY (TO_DATE(<temporal_column>))` — use `TO_DATE()` for TIMESTAMP columns to reduce cardinality per Snowflake best practices. For non-temporal filters, cluster on the ON-clause join key directly.

**Important caveats:**
- Only recommend for tables with many micro-partitions (typically multi-TB or high DML volume)
- Clustering consumes credits for ongoing Automatic Clustering — ensure query frequency justifies the cost
- After adding a cluster key, Automatic Clustering is enabled by default — no manual reclustering needed
- Reclustering creates micro-partition turnover with storage costs during Time Travel + Fail-safe retention (8-97 days)

**Risk:** SAFE — additive metadata change, no logic modification, fully reversible with `DROP CLUSTERING KEY`
**Impact:** 70-95% scan reduction (proportional to temporal filter selectivity)
**Severity:** P0 CRITICAL if table is large, MERGE runs frequently, and scan volume dominates runtime

**Related Reading:**
- [Snowflake Docs: Clustering Keys & Clustered Tables](https://docs.snowflake.com/en/user-guide/tables-clustering-keys)
- [Snowflake Docs: Automatic Clustering](https://docs.snowflake.com/en/user-guide/tables-auto-reclustering)
- [Snowflake Docs: SYSTEM$CLUSTERING_INFORMATION](https://docs.snowflake.com/en/sql-reference/functions/system_clustering_information)
- [Snowflake Docs: MERGE](https://docs.snowflake.com/en/sql-reference/sql/merge)

---

## Pattern 9: Temp Table Name Shadowing [CORRECTNESS / MISDIAGNOSIS TRAP]

**The trap:** When analyzing child statements, you see a JOIN or scan against a table whose name matches a known large permanent table and assume the permanent table is being scanned. In reality, an earlier child statement created a temporary table with the same name populated with a small filtered subset. All subsequent references in the session resolve to the temp table, not the permanent one. This causes false diagnoses such as recommending clustering on the permanent table for a scan that is actually against a tiny temp table.

**Detection:**
- Scan the ordered child statement list for `CREATE TEMPORARY TABLE <name>` before the slow child
- If found, all later references to `<name>` in the SP resolve to the temp table
- Confirm by comparing `SCAN_GB` in the slow child against the known size of the permanent table — if `SCAN_GB` is far smaller than the permanent table, the scan is against the temp table
- Also check: SP contains `CREATE TEMPORARY TABLE <name>` where `<name>` matches a permanent table or view in the same schema

**Confidence guidance:**
- HIGH: earlier sibling child statement creates a temp table with the exact same name AND scan size is far smaller than the permanent table — definitive
- MEDIUM: name match exists but scan size is ambiguous

**Check:** Ask the customer to run against their account:
```sql
SELECT table_name, table_schema, table_type
FROM information_schema.tables
WHERE table_schema = '<schema>'
  AND table_name = '<temp_table_name>'
  AND table_type IN ('BASE TABLE', 'VIEW');
```
If any rows are returned, the temp table is shadowing that object for the duration of the session.

**Fix:** Rename the temp table to an unambiguous name (e.g., prefix with `TMP_`). Re-examine the child that appeared slow — if it was scanning the temp table all along, the real bottleneck may lie elsewhere in the SP.
**Risk:** LOW — rename only; no logic change required
**Impact:** Corrects misdiagnosis; may redirect optimization effort to the true bottleneck
**Severity:** P0 (correctness) when shadowing is unintentional

---

## Quick Classification Matrix

| Signal | Pattern | Severity | Risk | Confidence Hint |
|--------|---------|----------|------|-----------------|
| 1000+ same-hash children, 1 row each | Loop (P1) | [CRITICAL] P0 | MODERATE | HIGH if >100 iterations |
| Broadcasts > 5 after temp table populate | Stats (P2) | [CRITICAL] P0 | SAFE | HIGH if >5 broadcasts |
| Rows inserted >> rows after DISTINCT | Fan-out (P3) | [CRITICAL] P0 | MODERATE | HIGH if ratio >100:1 |
| 10+ UPDATEs on same table | Serial UPDATE (P4) | [WARNING] P1 | LOW | HIGH if >10 UPDATEs |
| DELETE scans full table | CDC DELETE (P5) | [CRITICAL] P0 | SAFE | HIGH if full scan + selective filter |
| Many independent statements serial | Serial exec (P6) | [WARNING] P1 | MODERATE | HIGH if clearly independent |
| N identical CALLs, different filter | Repeated join (P7) | [WARNING] P1 | LOW | HIGH if >5 identical calls |
| MERGE full scan, no partition pruning | MERGE unclustered (P8) | [CRITICAL] P0 | SAFE | HIGH if pruning ratio >90% |
| Temp table name matches permanent table/view | Name shadowing (P9) | [CRITICAL] P0 (correctness) | LOW | HIGH if exact name match in same schema |

## Spill Assessment Notes

When evaluating warehouse sizing, always check **both** local and remote spill:
- **Local spill** (`ioLocalTempWriteBytes`): Data written to local SSD — adds latency but tolerable in small amounts
- **Remote spill** (`ioRemoteTempWriteBytes`): Data written to remote storage (S3/Azure Blob/GCS) — significantly worse due to network I/O. Even small remote spill warrants a warehouse size-up recommendation.
- **Combined spill** = local + remote. Use combined for threshold calculations.

## Severity Badge Reference

| Badge | Priority | Runtime Impact | Action Timeline |
|-------|----------|---------------|-----------------|
| `[CRITICAL]` | P0 | >50% of total runtime | Fix immediately |
| `[WARNING]` | P1 | 10-50% of total runtime | Fix in next release |
| `[INFO]` | P2 | <10% of total runtime | Optimize when convenient |
