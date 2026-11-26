> Part of **02 · PYSPARK (Days 7–12)** · [All projects](../PROJECTS.md) · Conventions: [AI rules](../PROJECTS.md#the-ai-rules) · [Git workflow](../PROJECTS.md#git--how-you-work)

## P03 — Batch Pipeline · Days 7–9

### 1. Goal
Take a year of raw taxi trips and turn them into clean daily summaries, in three stages — Bronze (raw), Silver (cleaned), Gold (aggregated) — where re-running any day gives exactly the same result.

### 2. Why this project exists
This is the shape of ~80% of production data pipelines: the **medallion architecture**. Raw is kept forever and never edited. Silver is cleaned and typed. Gold answers business questions. Separating them means a bug in your cleaning logic doesn't cost you the source data. Everything else in this course is a variation on this pattern.

### 3. Data source
**NYC TLC Trip Records** — the canonical DE dataset. Parquet, no signup. The TLC web page blocks `curl`, but the CDN behind it doesn't:
```bash
mkdir -p data/raw/yellow data/ref
for m in 01 02 03 04 05 06 07 08 09 10 11 12; do
  curl -o data/raw/yellow/yellow_tripdata_2024-$m.parquet \
    https://d37ci6vzurychx.cloudfront.net/trip-data/yellow_tripdata_2024-$m.parquet
done
curl -o data/ref/taxi_zone_lookup.csv \
  https://d37ci6vzurychx.cloudfront.net/misc/taxi_zone_lookup.csv
```
~50 MB/month, ~600 MB and ~40M rows for the year. `taxi_zone_lookup.csv` (12 KB) maps zone IDs to names — your join dimension.

> **Download January only first.** Get it correct on one month, then scale to twelve.

### 4. What you build

**Step 1 — Install and open a session.**
```bash
pip install 'pyspark==3.5.*'
```
Read one Parquet file, `.show(5)`, `.printSchema()`. Keep the Spark UI at `http://localhost:4040` open from now on.

**Step 2 — Bronze: land it as-is.** Read raw Parquet, add `ingested_at` and `source_file`, write to `data/bronze/` partitioned by date. No cleaning. Bronze is a faithful copy with provenance.

**Step 3 — Define the schema explicitly.** Write the `StructType` by hand. **Never use `inferSchema` on data you don't control** — it reads the file twice and silently changes types between months, which breaks downstream code at 3am.

**Step 4 — Silver: clean and type.**
- Drop trips with zero or negative distance, fare, or duration
- Drop trips longer than 24 hours or faster than 100 mph (GPS errors)
- Deduplicate — decide your grain and defend it
- Derive `trip_duration_min`, `avg_speed_mph`, `fare_per_mile`
- Join `taxi_zone_lookup` for pickup and dropoff zone names
- Count what you dropped at each step and log it. **A cleaning step that silently removes 30% of rows is a bug, not a feature.**

**Step 5 — Gold: aggregate.** Daily totals by pickup zone, dropoff zone and hour: trip count, average fare, average duration, total revenue. Partition by date.

**Step 6 — Make it idempotent and parameterised.** Every stage takes `--date`. Before writing a partition, delete it. Run the same date five times, get identical output.
```bash
python -m src.bronze --date 2024-01-15
python -m src.silver --date 2024-01-15
python -m src.gold   --date 2024-01-15
```

**Step 7 — Test the logic without Spark overhead.** Write transformations as pure functions `(DataFrame) -> DataFrame`, then test them on 10-row DataFrames. Business logic must be testable without a cluster.

**Step 8 — Read the Spark UI.** For each run, find: how many stages, where the shuffle happened, how much data moved, whether anything spilled to disk. Write what you found in the README. If you can't explain a shuffle, you don't yet understand your own job.

### 5. The AI part — Cleaning Rule Critic (~45 lines)

Your Step 4 cleaning rules are *guesses*. Is dropping trips over 100 mph right? What about a 0-mile trip with a $50 fare — fraud, error, or a cancelled ride?

Profile Silver (row counts dropped per rule, plus 5 examples of each dropped row), send that in one call, and ask: **which of these rules is removing legitimate data, and what am I not checking for?**

This is a fair use of a model: it's judgement about domain edge cases, not arithmetic. Expect it to suggest real ones you missed — negative tips, trips ending before they start, fares that don't match distance bands — and also to suggest nonsense. Log both in `AI_NOTES.md`.

### 6. What you learn
**PySpark:** DataFrame API · lazy evaluation and actions · narrow vs wide transformations · **the shuffle** · explicit schemas · joins (and broadcast) · window functions · partitioned writes · `explain()` · reading the Spark UI
**Data engineering:** medallion architecture · provenance columns · idempotent partition overwrite · parameterised runs as a precondition for backfill · testing transformation logic in isolation
**AI:** using profiling output as evidence · asking for criticism rather than generation

### 7. Done checklist
- [ ] Bronze, Silver, Gold all run for a given date
- [ ] Running the same date 5 times gives byte-identical Gold
- [ ] Schema is explicit; `inferSchema` appears nowhere
- [ ] Each cleaning step logs how many rows it dropped and why
- [ ] `pytest` tests transformation functions on small DataFrames, no cluster
- [ ] You can point at the Spark UI and explain every shuffle
- [ ] Scales from one month to twelve without a code change

### 8. In production this would be
`spark.sql.shuffle.partitions` sized against input volume or left to AQE, never guessed · **small-file compaction as a scheduled job** — small-file explosion is the #1 real Spark problem · backfill treated as a requirement, not a favour · cleaning thresholds owned by the business and version-controlled, not hardcoded by an engineer · row-count deltas between stages monitored, with an alert when the drop rate changes.

### 9. Git

```bash
git switch -c p03-batch-pipeline
```

**Commit at these points** — each one is a working step, not a save point:
- `feat: bronze layer with provenance columns`
- `feat: explicit StructType schema`
- `feat: silver cleaning with per-rule drop counts`
- `feat: gold daily aggregates`
- `fix: idempotent partition overwrite`
- `test: transformation functions on small DataFrames`
- `docs: README with Spark UI findings`

**PR description should answer:** your cleaning rules, and how many rows each one dropped.

```bash
gh pr create --fill && gh pr merge --squash --delete-branch
git switch main && git pull && git tag p03 && git push --tags
```

---

[← P02](p02-api-extractor.md) · [Roadmap](../PROJECTS.md) · [P04 →](p04-skew-clinic.md)
