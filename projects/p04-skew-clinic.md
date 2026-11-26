> Part of **02 · PYSPARK (Days 7–12)** · [All projects](../PROJECTS.md) · Conventions: [AI rules](../PROJECTS.md#the-ai-rules) · [Git workflow](../PROJECTS.md#git--how-you-work)

## P04 — Performance & Skew Clinic · Days 10–12

### 1. Goal
Take a slow Spark job and make it 5× faster — and be able to say exactly which change bought which speedup, with numbers.

### 2. Why this project exists
**This is the senior data engineer's actual job.** Cost is runtime × cluster size, so a 4× speedup is a 75% bill cut. Most Spark tuning advice online is cargo-cult ("just add `.cache()`"). This project teaches you to measure instead of guess. Nearly every PySpark interview question comes from this material.

### 3. Data source
The same TLC Parquet from P03 — don't re-download. The skew is **manufactured**, because you need to control it to study it:
```bash
python -m src.skew --input data/raw/yellow --zone 132 --target-pct 40 --out data/skewed
```
Zone 132 is JFK Airport, already the heaviest real zone, so you're exaggerating a genuine pattern rather than inventing one. Keep both datasets: benchmark on `data/skewed`, then re-run the winner on `data/raw` to confirm your fix didn't just overfit the skew.

### 4. What you build

**Step 1 — Write the naive version and measure it.** Join trips to zones, group by zone, aggregate. No tuning. Record wall-clock time, shuffle read/write bytes, and spill — all from the Spark UI. **This number is your baseline. Everything is measured against it.**

**Step 2 — Find the skew.** Count rows per join key and sort descending. You'll see one key dominating. In the Spark UI, look at the task duration distribution within the slowest stage: most tasks finish in seconds, one runs for minutes. **That one slow task is the entire problem** — 199 cores idle while 1 does the work.

**Step 3 — Broadcast the small side.** The zone lookup is 12 KB; it should never be shuffled. `broadcast(zones)`. Measure.

**Step 4 — Prune columns and push down predicates.** Select only needed columns *before* the join; filter dates before reading. Confirm in `explain()` that `PushedFilters` is non-empty. Measure.

**Step 5 — Salt the skewed key.** Add a random suffix (0–N) to the hot key, join on the salted key, then aggregate twice — once per salt bucket, then combine. This is the classic skew fix and the one interviewers ask about. Measure.

**Step 6 — Turn on AQE.** `spark.sql.adaptive.enabled=true`, `spark.sql.adaptive.skewJoin.enabled=true`. Measure. **Then compare against your manual salting** — AQE often matches it. Knowing when the engine handles it for you is as valuable as knowing how to do it by hand.

**Step 7 — Fix partition counts.** Default `shuffle.partitions` is 200 regardless of data size. Aim for ~128 MB per partition. Compute the right number from your input size. Measure.

**Step 8 — The UDF experiment.** Write a Python UDF for something simple (categorise fare into bands). Measure. Rewrite it with native `when().otherwise()`. Measure. Then try a Pandas UDF. **Expect the Python UDF to be several times slower** — it serialises every row to a Python process and back. Write the numbers down; this is a guaranteed interview question.

**Step 9 — Produce `BENCHMARKS.md`.** A table: variant, runtime, shuffle bytes, spill, speedup vs baseline. One row per change.

### 5. The AI part — Execution Plan Reviewer (~55 lines)

**This is the most useful AI workflow in the entire course, and almost nobody has it on a portfolio.**

Capture `df.explain(mode="formatted")` and the stage metrics, feed both in one call, and ask for **ranked hypotheses** about the bottleneck — not a fix.

```python
PROMPT = """Here is a Spark physical plan and stage metrics.
List the top 3 likely bottlenecks, ranked, each as:
  hypothesis | evidence in the plan | the one change to test it
Do not suggest fixes you cannot justify from the plan below.

PLAN:\n{plan}\n\nSTAGE METRICS:\n{metrics}"""
```

Then **test each hypothesis and score the model.** In `AI_NOTES.md` keep a running tally: hypotheses proposed, confirmed, wrong. By the end you'll have a measured accuracy figure for Claude-as-performance-reviewer on your own workload. That table is a genuinely differentiated portfolio artefact — it shows you use AI *with* evidence rather than on faith.

### 6. What you learn
**PySpark:** data skew and salting · AQE · broadcast joins and thresholds · predicate pushdown · column pruning · partition sizing · spill diagnosis · reading physical plans · why Python UDFs are slow and what Pandas UDFs change · `cache()` and when it's a mistake
**Data engineering:** benchmarking method — one change at a time, always measured · cost as a function of runtime · performance as a tracked metric
**AI:** hypothesis ranking rather than answer generation · scoring model output against reality

### 7. Done checklist
- [ ] `BENCHMARKS.md` has a row per change with real numbers
- [ ] Final version ≥5× faster than baseline
- [ ] Every speedup attributed to one named change
- [ ] You can explain salting on a whiteboard without notes
- [ ] You have the UDF vs native vs Pandas UDF numbers
- [ ] The winning config re-run on unskewed data still wins
- [ ] `AI_NOTES.md` has the hypothesis scoreboard

### 8. In production this would be
Job runtime tracked as a metric over time with alerts on regression · skew usually discovered at 3am when a job that ran 20 minutes for a year suddenly takes 6 hours because one customer got big · cluster sizing revisited quarterly against actual data growth · the salting threshold driven by config, not a hardcoded constant · a cost dashboard per job, because finance eventually asks.

### 9. Git

```bash
git switch -c p04-skew-clinic
```

**Commit at these points** — each one is a working step, not a save point:
- `feat: skew injection script`
- `perf: baseline measurement recorded`
- `perf: broadcast zone lookup, <before> -> <after>`
- `perf: column pruning and predicate pushdown, <before> -> <after>`
- `perf: salt skewed key, <before> -> <after>`
- `perf: AQE enabled, compared against manual salting`
- `docs: BENCHMARKS.md and AI_NOTES scoreboard`

**PR description should answer:** which change bought which speedup, with numbers.

**Put the real numbers in the `perf:` messages.** A history of measured optimisations is rare and immediately credible.

```bash
gh pr create --fill && gh pr merge --squash --delete-branch
git switch main && git pull && git tag p04 && git push --tags
```

---

[← P03](p03-batch-pipeline.md) · [Roadmap](../PROJECTS.md) · [P05 →](p05-lakehouse-medallion.md)
