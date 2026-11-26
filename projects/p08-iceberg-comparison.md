> Part of **04 · DELTA LAKE & ICEBERG (Days 19–24)** · [All projects](../PROJECTS.md) · Conventions: [AI rules](../PROJECTS.md#the-ai-rules) · [Git workflow](../PROJECTS.md#git--how-you-work)

## P08 — Iceberg & Format Comparison · Days 22–24

### 1. Goal
Build the same table in Apache Iceberg, change its partitioning **without rewriting the data**, and write an honest comparison of Delta vs Iceberg vs plain Parquet.

### 2. Why this project exists
Table format choice is a multi-year decision argued about by principal engineers. More importantly, this project teaches you to **evaluate a technology** rather than adopt it because it's trending — a senior skill that generalises far beyond Iceberg. And the one experiment at the centre of it (partition evolution) is something Delta genuinely cannot do.

### 3. Data source
The same generated dimension from P07, plus the TLC trips as a large partitioned table — you need real volume for partition evolution to mean anything.
```bash
spark-sql --packages org.apache.iceberg:iceberg-spark-runtime-3.5_2.12:1.5.0 \
  --conf spark.sql.catalog.local=org.apache.iceberg.spark.SparkCatalog \
  --conf spark.sql.catalog.local.type=hadoop \
  --conf spark.sql.catalog.local.warehouse=./warehouse
```

### 4. What you build

**Step 1 — Create an Iceberg table and inspect the metadata.** Write the taxi data partitioned by day. Explore the metadata directory. **Compare the structure to Delta's `_delta_log`** — Iceberg uses a manifest tree, Delta a flat log. That difference drives most of their behavioural differences.

**Step 2 — Hidden partitioning.** In Hive you had to filter on a partition column explicitly, or you'd scan everything. Iceberg derives partitions from the source column:
```sql
CREATE TABLE local.db.trips (...) PARTITIONED BY (days(pickup_datetime));
SELECT * FROM local.db.trips WHERE pickup_datetime > '2024-01-15';  -- prunes automatically
```
No separate partition column, no way for a user to accidentally bypass it. Confirm pruning in the query plan.

**Step 3 — Partition evolution. This is the experiment the project exists for.**
```sql
ALTER TABLE local.db.trips ADD PARTITION FIELD months(pickup_datetime);
```
Change the spec from day to month. **Old data keeps its old layout; new data uses the new one; queries read both correctly.** No rewrite, no downtime. Verify by querying across the boundary. Delta cannot do this — you'd rewrite the table.

**Step 4 — Branching and tagging (write-audit-publish).**
```sql
ALTER TABLE local.db.trips CREATE BRANCH audit_branch;
-- write to the branch, run quality checks, then:
-- fast-forward main to the branch only if checks pass
```
Bad data never becomes visible to readers. **This is Iceberg's standout operational feature** — git-like semantics for a table.

**Step 5 — Snapshot expiry.** Iceberg's `VACUUM` equivalent. Same trade-off as P07: retention versus storage cost.

**Step 6 — Write `COMPARISON.md`.** A real evaluation, not a feature list. Compare on: schema evolution, partition evolution, engine support (who can read it — Spark, Trino, Snowflake, BigQuery, Athena?), catalog requirements, community and governance, operational maturity. End with: **"For a team that needs X, I would choose Y, because Z."** Take a position.

### 5. The AI part — Research Assistant with Citation Discipline (~40 lines)

Have Claude draft the comparison. Then **verify every factual claim against official documentation** and mark each one: `CONFIRMED`, `CORRECTED`, or `UNVERIFIABLE`.

**This project is where you learn the hallucination failure mode on your own terms rather than in front of a staff engineer.** Models are confidently wrong about version-specific feature support — exactly what this document is about. Expect errors on which engines support which Iceberg version, and on recently-changed Delta features.

Keep the marked-up draft in the repo. **A document showing you caught the model being wrong is worth more than a document that's merely correct**, because it's evidence of method.

### 6. What you learn
**Iceberg:** metadata and manifest structure · hidden partitioning · **partition evolution** · branching/tagging and write-audit-publish · catalogs · snapshot expiry
**Data engineering:** how to evaluate technology · writing an ADR · taking a defensible position under uncertainty
**AI:** citation discipline · verifying claims against primary sources · recognising confident wrongness

### 7. Done checklist
- [ ] Partition spec changed on a live table; old and new data both queryable
- [ ] Hidden partitioning confirmed pruning in the query plan
- [ ] A write-audit-publish cycle completed on a branch
- [ ] `COMPARISON.md` covers ≥6 dimensions and takes a position
- [ ] Every AI claim marked CONFIRMED / CORRECTED / UNVERIFIABLE
- [ ] You can argue either side of Delta vs Iceberg for a given scenario
- [ ] `AI_NOTES.md` lists the specific claims the model got wrong

### 8. In production this would be
Format choice driven by which query engines must read the table — that constraint usually decides it before anything else · the catalog (Glue, Polaris, Nessie, Unity) often the real lock-in, not the format · migrations between formats as staffed projects, not weekend work · write-audit-publish standard for anything customer-facing · snapshot expiry scheduled, or metadata grows without bound.

### 9. Git

```bash
git switch -c p08-iceberg-comparison
```

**Commit at these points** — each one is a working step, not a save point:
- `feat: iceberg table with hidden partitioning`
- `feat: partition evolution day -> month without rewrite`
- `feat: write-audit-publish on a branch`
- `docs: COMPARISON.md with claims marked CONFIRMED/CORRECTED`
- `docs: AI_NOTES on hallucinated feature support`

**PR description should answer:** which format you'd choose for which scenario, and why.

```bash
gh pr create --fill && gh pr merge --squash --delete-branch
git switch main && git pull && git tag p08 && git push --tags
```

---

[← P07](p07-delta-deep-dive.md) · [Roadmap](../PROJECTS.md) · [P09 →](p09-dbt-analytics.md)
