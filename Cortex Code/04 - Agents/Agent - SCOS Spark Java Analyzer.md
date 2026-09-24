---
type: cortex-code-agent
name: scos-spark-java-analyzer
title: "SCOS Spark Java Analyzer"
source_repo: Snowflake-Labs/coco-skills
source_path: skills/spark-migration/snowpark-connect/migrate-spark-java-to-snowpark-connect/agents/analyzer.md
source_url: https://github.com/Snowflake-Labs/coco-skills/blob/28b549f48da9994307081a9d4d2f16379f5a9c16/skills/spark-migration/snowpark-connect/migrate-spark-java-to-snowpark-connect/agents/analyzer.md
category: scos-migration
status: active
last_reviewed: 2026-09-24
tags:
  - snowflake
  - cortex-code
  - agent
  - scos-migration
aliases:
  - "SCOS Spark Java Analyzer"
  - "scos-spark-java-analyzer"
---

# SCOS Spark Java Analyzer

**Type:** Cortex Code Agent
**Agent Name:** `scos-spark-java-analyzer`
**Source:** Snowflake-Labs/coco-skills
**Repository Path:** `skills/spark-migration/snowpark-connect/migrate-spark-java-to-snowpark-connect/agents/analyzer.md`
**Source URL:** [skills/spark-migration/snowpark-connect/migrate-spark-java-to-snowpark-connect/agents/analyzer.md](https://github.com/Snowflake-Labs/coco-skills/blob/28b549f48da9994307081a9d4d2f16379f5a9c16/skills/spark-migration/snowpark-connect/migrate-spark-java-to-snowpark-connect/agents/analyzer.md)
**Status:** Available
**Last Reviewed:** 2026-09-24

---

## Mission

Execute specialized sub-phase operations for SCOS Spark Java Analyzer in the Snowpark Connect (SCOS) migration and validation pipeline.

---

## Responsibilities

- Perform deterministic execution for SCOS Spark Java Analyzer
- Read state from migration_state.json and conversion root
- Follow bounded rules for SCOS compatibility and verification
- Emit structured JSON logs and reports for the coordinator

---

## Inputs

- **Primary Inputs:** migration_state.json, analysis.json, workload files, conversion root

---

## Outputs

- **Primary Outputs:** Updated analysis, test scripts, synthetic datasets, or validation reports

---

## Tools / Capabilities

- `read`
- `write`
- `bash`
- `snowflake_sql_execute`

---

## Constraints

Must preserve PySpark/Spark semantics; adhere strictly to SCOS migration rules.

---

## Invocation

This agent is invoked by parent coordinators or orchestrators using task delegation:

```text
Spawn subagent scos-spark-java-analyzer with role instructions and target parameters.
```

---

## Best Use Cases

- Use when executing scoped, deterministic sub-tasks requiring clear separation of concerns.
- Use when preventing cross-contamination between exploration, generation, and critical auditing.

---

## Bad Use Cases

- Do NOT use for ad-hoc, unbounded interactive chats.
- Do NOT use outside its designated coordinator or plugin lifecycle.

---

## Copy-Ready Agent Definition

`🤖 AGENT` `✅ COPY-READY`

```markdown
# Analyzer Agent — Phase 1 Specialist (Java)

Run the SCOS compatibility analyzer on the Java workload and produce `analysis.json`.

## Inputs

Read `migration_state.json` from the conversion root to get:
- `manifest` — list of `.java` files to analyze
- `migrated_dir` — directory containing the copied source files
- `skill_directory` — path to `snowpark-connect/` for `uv run --project`

## Step 0: Determine RAG Backend

Check if Cortex Search RAG is already initialized:
```bash
uv run --project <SKILL_DIRECTORY> \
  python -c "
from snowflake.snowpark import Session
session = Session.builder.create()
try:
    rows = session.sql(\"SHOW CORTEX SEARCH SERVICES LIKE 'SCOS_COMPAT_ISSUES_SERVICE'\").collect()
    print(f'EXISTS {rows[0][\"database_name\"]}.{rows[0][\"schema_name\"]}' if rows else 'NOT_FOUND')
except Exception as e:
    print(f'ERROR {e}')
"
```

- **If `EXISTS`**: add `--rag-backend cortex` to the Step 1 command.
- **If `NOT_FOUND` or `ERROR`**: omit `--rag-backend`.

## Step 1: Run the Analyzer

```bash
uv run --project <SKILL_DIRECTORY> \
  python <SKILL_DIRECTORY>/scripts/analyze_java.py \
  --path <migrated_dir> \
  --recipe-edits <CONVERSION>/migration_state.json \
  --rag-backend trigger \
  --output <CONVERSION>/analysis.json
```

Wait for completion. Verify `analysis.json` is valid JSON.

## Step 2: Supplement for Known Blind Spots

Scan ALL `.java` files in the manifest for patterns the analyzer may miss:

1. **UDF patterns not in analysis**: `new UDF1`, `new UDF2`, `spark.udf().register(`, `UserDefinedFunction`
2. **`checkpoint()` / `localCheckpoint()`** calls
3. **Map column subscript**: `.apply(col("key"))` pattern on a Column
4. **Catalyst imports**: `org.apache.spark.sql.catalyst.*`
5. **Hadoop/HDFS imports**: `org.apache.hadoop.*`
6. **JavaSparkContext**: `new JavaSparkContext(`, `JavaSparkContext jsc`
7. **Spline imports**: `za.co.absa.spline.*`
8. **RDD aggregate / accumulator / §10 ops** — grep for these tokens (the guide's authoritative Java list; all have `Dataset<Row>` workarounds in `../../references/java/rdd-conversion.md`, so flag them, do NOT punt to a blanket TODO):
   - **Aggregate & reduce**: `aggregate(`, `treeAggregate(`, `treeReduce(`, `.reduce(`, `.fold(`, `foldByKey(`, `aggregateByKey(`, `combineByKey(`, `groupByKey(` (§6.1–6.9).
   - **Accumulators**: `longAccumulator(`, `doubleAccumulator(`, `collectionAccumulator(`, `LongAccumulator`, `DoubleAccumulator`, `CollectionAccumulator`, `AccumulatorV2`, `jsc.accumulator(` (deprecated), and the classic `.forEach(r -> acc.add(` shape → a DataFrame aggregation (§6.10–6.16). Hard gaps only: `foreachPartition` sinks, cache-hit counters across `persist`/`unpersist`, threads polling `acc.value()`, `writeStream().foreachBatch` cross-batch state (§7). Any `JavaSparkContext` or `spark.sparkContext()` hop to reach accumulator/parallelize APIs is blocked under Connect (`SPRKCNTSCL1500`).
   - **UDAF**: `UserDefinedAggregateFunction`, `functions.udaf(`, `registerJavaUDAF` (the last silently becomes a scalar UDF — wrong results, no error) → §6.17.
   - **§10 verified ops**: `groupBy(`, `mapPartitionsWithIndex(`, `partitionBy(`, `repartitionAndSortWithinPartitions(`, `collectAsMap(`, `countApprox(`, `countApproxDistinct(`, `saveAsObjectFile(`, `saveAsSequenceFile(`, `getStorageLevel(`, `context(`, `toDebugString(` (§10). Unambiguous names are auto-detected; the `Dataset` homonyms (`groupBy(`) usually mean a `.toJavaRDD()` hop that should not exist — check whether the receiver is really a `JavaRDD`.

For each pattern NOT already in `analysis.json`, append a supplementary entry:
```json
{
  "file": "<path>",
  "lines": "<line_range>",
  "code": "<snippet>",
  "final_risk": 0.9,
  "root_cause": "<description>",
  "explanation": "<why this is a problem in SCOS>",
  "fix": "<suggested fix>",
  "confidence": "HIGH",
  "source": "supplementary_scan"
}
```

## Step 3: Update Gate File

Update `migration_state.json`:
```json
{
  "phase": 1,
  "phases_completed": {
    "1_analysis": {"status": "passed", "issues_found": N, "supplementary_added": M}
  }
}
```

## Output

- `analysis.json` in the conversion root
- Updated `migration_state.json`
- Report: "Analysis complete: N issues found (M supplementary)"
```

---

## Related Plugin

- None (Standalone Skill Agent)

---

## Related Skills

- [[Skill - Migrate Spark Java to Snowpark Connect]]
- [[Cortex Code Skills Master Index]]
- [[CoCo Quick Access]]

---

## Source

- **Repository File:** `skills/spark-migration/snowpark-connect/migrate-spark-java-to-snowpark-connect/agents/analyzer.md`
- **GitHub URL:** [Snowflake-Labs/coco-skills/skills/spark-migration/snowpark-connect/migrate-spark-java-to-snowpark-connect/agents/analyzer.md](https://github.com/Snowflake-Labs/coco-skills/blob/28b549f48da9994307081a9d4d2f16379f5a9c16/skills/spark-migration/snowpark-connect/migrate-spark-java-to-snowpark-connect/agents/analyzer.md)
- **Commit:** `28b549f48da9994307081a9d4d2f16379f5a9c16`

---

## Official Repository Knowledge

Extracted directly from the official Snowflake Labs Cortex Code repository. Enforces specialized operational bounds, quality gates, and verifiable evidence requirements.

---

## My Operational Notes

- Always supply explicit input and output paths when dispatching this agent.
- Keep agent tasks atomic and review the returned summary before integrating changes into the primary branch.

---

## Client Demonstration Notes

- Highlight to enterprise stakeholders how specialist agents eliminate hallucination by isolating write permissions from discovery tools.
