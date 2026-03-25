# SNOWHOUSE Query Templates

## Data Source Reference

| View | Location | Purpose |
|------|----------|---------|
| ACCOUNT_ETL_V | `SNOWHOUSE_IMPORT.PROD` | Account locator → ID + deployment |
| JOB_ETL_V | `SNOWHOUSE_IMPORT_SHARE_DB.<DEPLOYMENT>` | Query history + stats |
| TABLE_ETL_V | `SNOWHOUSE_IMPORT_SHARE_DB.<DEPLOYMENT>` | Table metadata |
| SCHEMA_ETL_V | `SNOWHOUSE_IMPORT_SHARE_DB.<DEPLOYMENT>` | Schema metadata |
| DATABASE_ETL_V | `SNOWHOUSE_IMPORT_SHARE_DB.<DEPLOYMENT>` | Database metadata |
| TABLE_COLUMN_ETL_V | `SNOWHOUSE_IMPORT_SHARE_DB.<DEPLOYMENT>` | Column types |

## JOB_ETL_V Available Columns
UUID, JOB_ID, PARENT_JOB_ID, DESCRIPTION, TOTAL_DURATION (ms), STATS (variant), ERROR_CODE, CREATED_ON, WAREHOUSE_NAME, QUERY_PARAMETERIZED_HASH, ACCOUNT_ID

## JOB_ETL_V Does NOT Have
EXECUTION_STATUS, STATUS, WAREHOUSE_SIZE, BYTES_SCANNED

## STATS Variant Access

```sql
j.STATS:stats:scanBytes::NUMBER
j.STATS:stats:numRowsInserted::NUMBER
j.STATS:stats:numRowsUpdated::NUMBER
j.STATS:stats:numRowsDeleted::NUMBER
j.STATS:stats:ioLocalTempWriteBytes::NUMBER
j.STATS:stats:ioRemoteTempWriteBytes::NUMBER
j.STATS:stats:hashJoinNumberBroadcastDecisions::NUMBER
j.STATS:stats:outputBytes::NUMBER
j.STATS:stats:filesCreated::NUMBER
j.STATS:stats:dictProcessedRows::NUMBER
j.STATS:stats:netReceivedBytes::NUMBER
j.STATS:stats:numPartitionsScanned::NUMBER
j.STATS:stats:numPartitionsTotal::NUMBER
```

### Partition Pruning Ratio

Used in Step 6 to quantify partition pruning effectiveness before/after clustering.

```sql
SELECT
    j.JOB_ID,
    j.STATS:stats:numPartitionsScanned::NUMBER AS PARTITIONS_SCANNED,
    j.STATS:stats:numPartitionsTotal::NUMBER AS PARTITIONS_TOTAL,
    ROUND(DIV0(j.STATS:stats:numPartitionsScanned::NUMBER, j.STATS:stats:numPartitionsTotal::NUMBER) * 100, 1) AS PRUNING_RATIO_PCT,
    ROUND(j.STATS:stats:scanBytes::NUMBER / POWER(1024,3), 2) AS SCAN_GB
FROM SNOWHOUSE_IMPORT_SHARE_DB.<DEPLOYMENT>.JOB_ETL_V j
WHERE j.ACCOUNT_ID = <ACCOUNT_ID>
  AND j.JOB_ID = <BOTTLENECK_JOB_ID>;
```

- **PRUNING_RATIO_PCT near 100%** = no pruning (full scan) — confirms clustering will help
- **PRUNING_RATIO_PCT < 50%** = reasonable pruning already — clustering may not be the fix

Human-readable conversions:
```sql
ROUND(j.STATS:stats:scanBytes::NUMBER / POWER(1024,3), 1) AS scan_gb
ROUND((NVL(j.STATS:stats:ioLocalTempWriteBytes::NUMBER,0) + NVL(j.STATS:stats:ioRemoteTempWriteBytes::NUMBER,0)) / POWER(1024,3), 1) AS spill_gb
ROUND(DIV0(j.STATS:stats:numPartitionsScanned::NUMBER, j.STATS:stats:numPartitionsTotal::NUMBER) * 100, 1) AS pruning_ratio_pct
```

## Column Datatype Check

DATA_TYPE_ENCODED is **VARCHAR containing JSON** (not variant). Example value:
```json
{"type":"TIMESTAMP_NTZ","precision":0,"scale":9,"nullable":true}
```

Cluster key truncation rules:
- `"type":"TIMESTAMP_NTZ"` or `"type":"TIMESTAMP_LTZ"` or `"type":"TIMESTAMP_TZ"` → needs `TO_DATE(col)` in cluster key to reduce cardinality
- `"type":"DATE"` → no truncation needed
- `"type":"FIXED"` → integer, no truncation needed
- `"type":"TEXT"` (high cardinality) → consider `SUBSTRING(col, 1, 5)` since Snowflake only tracks first 5 bytes for VARCHAR clustering

---

## New Query Templates

### 30-Day Duration Trend

Used in Step 2 to assess whether a query is degrading, stable, or intermittent.

```sql
SELECT
    DATE_TRUNC('day', j.CREATED_ON) AS DAY,
    COUNT(*) AS RUNS,
    SUM(CASE WHEN j.ERROR_CODE IS NOT NULL THEN 1 ELSE 0 END) AS ERRORS,
    ROUND(AVG(j.TOTAL_DURATION) / 60000, 1) AS AVG_MIN,
    ROUND(MEDIAN(j.TOTAL_DURATION) / 60000, 1) AS MEDIAN_MIN,
    ROUND(MAX(j.TOTAL_DURATION) / 60000, 1) AS MAX_MIN,
    ROUND(PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY j.TOTAL_DURATION) / 60000, 1) AS P95_MIN
FROM SNOWHOUSE_IMPORT_SHARE_DB.<DEPLOYMENT>.JOB_ETL_V j
WHERE j.ACCOUNT_ID = <ACCOUNT_ID>
  AND j.QUERY_PARAMETERIZED_HASH = '<HASH>'
  AND j.CREATED_ON >= DATEADD(day, -30, CURRENT_TIMESTAMP())
GROUP BY DAY ORDER BY DAY;
```

**Trend classification:**
- DEGRADING: latest-week avg > previous-week avg by >20%
- STABLE-SLOW: variance < 15% across 30 days
- INTERMITTENT: max > 3x median

### Warehouse Sizing / Spill Analysis

Used in Step 3 to assess whether warehouse size contributes to the bottleneck.

```sql
SELECT
    ROUND(SUM(c.STATS:stats:scanBytes::NUMBER) / POWER(1024,3), 1) AS TOTAL_SCAN_GB,
    ROUND(SUM(c.STATS:stats:ioLocalTempWriteBytes::NUMBER) / POWER(1024,3), 1) AS LOCAL_SPILL_GB,
    ROUND(SUM(c.STATS:stats:ioRemoteTempWriteBytes::NUMBER) / POWER(1024,3), 1) AS REMOTE_SPILL_GB,
    ROUND(SUM(c.STATS:stats:outputBytes::NUMBER) / POWER(1024,3), 1) AS TOTAL_OUTPUT_GB,
    c.WAREHOUSE_NAME,
    COUNT(*) AS CHILD_COUNT
FROM SNOWHOUSE_IMPORT_SHARE_DB.<DEPLOYMENT>.JOB_ETL_V c
WHERE c.ACCOUNT_ID = <ACCOUNT_ID>
  AND c.PARENT_JOB_ID = <PARENT_JOB_ID>
GROUP BY c.WAREHOUSE_NAME;
```

**Spill thresholds (local + remote combined):**
- Spill < 10% of scan: Minor — not the bottleneck
- Spill 10-50% of scan: Moderate — consider warehouse size-up as P2
- Spill > 50% of scan: Heavy — recommend size-up as P1
- Remote spill > 0: Any remote spill warrants a size-up recommendation (network I/O penalty)
- No spill: Warehouse sizing adequate

### Loop Detection Aggregation

Used in Step 4 to confirm cursor loop patterns.

```sql
SELECT
    c.QUERY_PARAMETERIZED_HASH,
    COUNT(*) AS ITERATIONS,
    ROUND(SUM(c.TOTAL_DURATION) / 1000.0, 1) AS TOTAL_SEC,
    ROUND(SUM(c.STATS:stats:scanBytes::NUMBER) / POWER(1024,3), 1) AS TOTAL_SCAN_GB,
    SUM(c.STATS:stats:numRowsInserted::NUMBER) AS TOTAL_ROWS
FROM SNOWHOUSE_IMPORT_SHARE_DB.<DEPLOYMENT>.JOB_ETL_V c
WHERE c.ACCOUNT_ID = <ACCOUNT_ID>
  AND c.PARENT_JOB_ID = <PARENT_JOB_ID>
GROUP BY c.QUERY_PARAMETERIZED_HASH
ORDER BY TOTAL_SEC DESC;
```

### Post-Implementation Monitoring (SNOWHOUSE)

Used after optimization to verify improvements persist.

```sql
SELECT
    DATE_TRUNC('day', j.CREATED_ON) AS DAY,
    COUNT(*) AS RUNS,
    SUM(CASE WHEN j.ERROR_CODE IS NOT NULL THEN 1 ELSE 0 END) AS ERRORS,
    ROUND(AVG(j.TOTAL_DURATION) / 60000, 1) AS AVG_MIN,
    ROUND(MAX(j.TOTAL_DURATION) / 60000, 1) AS MAX_MIN,
    ROUND(PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY j.TOTAL_DURATION) / 60000, 1) AS P95_MIN
FROM SNOWHOUSE_IMPORT_SHARE_DB.<DEPLOYMENT>.JOB_ETL_V j
WHERE j.ACCOUNT_ID = <ACCOUNT_ID>
  AND j.QUERY_PARAMETERIZED_HASH = '<HASH>'
  AND j.CREATED_ON >= DATEADD(day, -7, CURRENT_TIMESTAMP())
GROUP BY DAY ORDER BY DAY;
```
