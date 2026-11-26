> Part of **03 · DATABRICKS (Days 13–18)** · [All projects](../PROJECTS.md) · Conventions: [AI rules](../PROJECTS.md#the-ai-rules) · [Git workflow](../PROJECTS.md#git--how-you-work)

## P05 — Lakehouse Medallion · Days 13–15

### 1. Goal
Rebuild your P03 pipeline on Databricks, where Bronze/Silver/Gold are **real governed tables** rather than folders, new files are picked up automatically, and the whole thing runs on a schedule without you.

### 2. Why this project exists
P03 taught you Spark. This teaches you what a company actually buys when it buys Databricks: a catalog so people can find tables, permissions so the wrong people can't read them, automatic incremental ingestion, and a scheduler. The engine is the same; the operating model is the product.

### 3. Data source
The TLC Parquet from P03, uploaded to a Unity Catalog Volume so Auto Loader has a landing zone.
```sql
CREATE CATALOG IF NOT EXISTS main;
CREATE SCHEMA  IF NOT EXISTS main.taxi;
CREATE VOLUME  IF NOT EXISTS main.taxi.landing;
```
```bash
pip install databricks-cli
databricks configure --token
databricks fs cp -r data/raw/yellow dbfs:/Volumes/main/taxi/landing/yellow/
```
**Upload one month first, then drip-feed the rest one file at a time.** That's how you see Auto Loader doing incremental discovery rather than one bulk read.

> **Watch the spend.** Free Edition has a credit budget. Job clusters only, autotermination at 10 minutes, and never leave an all-purpose cluster running overnight. This is the most common way people burn their free credits in a week.

### 4. What you build

**Step 1 — Set up Unity Catalog.** Create catalog → schema → volume as above. Understand the three-level name `catalog.schema.table` — it replaces the two-level `database.table` you know from SQL.

**Step 2 — Bronze with Auto Loader.** Instead of reading a fixed path, use `cloudFiles` to watch the volume:
```python
(spark.readStream.format("cloudFiles")
   .option("cloudFiles.format", "parquet")
   .option("cloudFiles.schemaLocation", "/Volumes/main/taxi/_schema/bronze")
   .load("/Volumes/main/taxi/landing/yellow/")
   .writeStream
   .option("checkpointLocation", "/Volumes/main/taxi/_ckpt/bronze")
   .trigger(availableNow=True)
   .toTable("main.taxi.bronze_trips"))
```
Auto Loader tracks which files it has already seen. Add a new file, run again — only the new file is read. **That's the feature.**

**Step 3 — Silver and Gold as managed tables.** Port your P03 logic, writing with `.saveAsTable("main.taxi.silver_trips")` instead of a path. Managed means Databricks owns the storage and lifecycle.

**Step 4 — Set permissions that actually restrict.**
```sql
GRANT SELECT ON TABLE main.taxi.gold_daily TO `analysts`;
GRANT MODIFY ON TABLE main.taxi.silver_trips TO `engineers`;
```
Then verify with `SHOW GRANTS`. Permissions you didn't verify are permissions you don't have.

**Step 5 — Build the Workflow.** A multi-task job: bronze → silver → gold, each depending on the previous, with retries (2), a timeout, and email on failure. Parameterise by date. Schedule it daily.

**Step 6 — Prove the automation.** Drop a new file into the volume. Run the job. Confirm only the new data flowed through and Gold updated. **No human in the loop.**

**Step 7 — One SQL dashboard.** Top 10 routes by revenue, trips per hour. This is the first time your work is visible to a non-engineer, which is the point of all of it.

### 5. The AI part — Permission Auditor (~50 lines)

Write your access policy in plain English in `ACCESS_POLICY.md`:
> Analysts read Gold only. Engineers read and write Silver and Bronze. Nobody but the service principal writes Gold. No one outside the data team reads raw.

Then dump actual grants (`SHOW GRANTS` across your tables) and ask the model, in one call: **does the real state match the written policy, and what is over-granted?**

This is a genuinely good use: comparing a fuzzy English policy against structured state is exactly the kind of translation a model is good at, and over-granting is the most common real-world governance failure. Expect it to miss inherited permissions — catalog-level grants cascade — which is precisely the lesson.

**Rule 1 still applies:** it reports; you run the `REVOKE` yourself after reading.

### 6. What you learn
**Databricks:** Unity Catalog and the three-level namespace · Volumes · **Auto Loader** and schema evolution · managed vs external tables · Workflows with task dependencies · job clusters vs all-purpose and why that choice is money · SQL warehouses · notebooks vs repos
**Data engineering:** governance as code · incremental file discovery · scheduled orchestration · cost awareness as a daily habit
**AI:** comparing stated policy against actual system state

### 7. Done checklist
- [ ] Three-level namespace in use; no `/dbfs/` paths in your table code
- [ ] Auto Loader picks up a new file without reprocessing old ones
- [ ] `SHOW GRANTS` matches `ACCESS_POLICY.md`
- [ ] The Workflow runs end-to-end on a schedule with retries configured
- [ ] Dropping a file triggers a correct refresh with no manual step
- [ ] No all-purpose cluster running when you stop for the day
- [ ] `AI_NOTES.md` records what the auditor caught and what it missed

### 8. In production this would be
Notebooks for exploration only — production is Python modules in Git, deployed via Asset Bundles, reviewed in PRs · **all-purpose clusters left running are the single biggest source of wasted spend at most companies** · Unity Catalog permissions owned by a platform team and audited quarterly · job clusters sized per task · serverless/Photon chosen with a cost comparison, not by default · separate catalogs per environment, with prod data never readable from dev.

### 9. Git

```bash
git switch -c p05-lakehouse-medallion
```

**Commit at these points** — each one is a working step, not a save point:
- `chore: unity catalog, schema and volume`
- `feat: bronze via Auto Loader`
- `feat: silver and gold as managed Delta tables`
- `feat: access policy and grants`
- `feat: multi-task Workflow with retries`
- `docs: ACCESS_POLICY.md and AI_NOTES`

**PR description should answer:** what Auto Loader gave you that your P01 ingestor did not.

```bash
gh pr create --fill && gh pr merge --squash --delete-branch
git switch main && git pull && git tag p05 && git push --tags
```

---

[← P04](p04-skew-clinic.md) · [Roadmap](../PROJECTS.md) · [P06 →](p06-bundles-cicd.md)
