> Part of **05 · DBT (Days 25–30)** · [All projects](../PROJECTS.md) · Conventions: [AI rules](../PROJECTS.md#the-ai-rules) · [Git workflow](../PROJECTS.md#git--how-you-work)

## P10 — Data Quality Gates · Days 28–30

### 1. Goal
Add quality checks that **stop bad data from ever reaching a dashboard**, and run the same dbt project against a second warehouse (Databricks) so you learn what is dbt and what is the warehouse.

### 2. Why this project exists
**A silently wrong number is worse than a late one.** A late dashboard annoys people; a wrong one gets a decision made on it. Learning to block publication rather than serve bad data is the line between a data engineer and someone who moves data around.

Running the same project on a second warehouse is the secondary lesson: it shows you exactly which parts of your SQL were portable and which were warehouse-specific — a distinction most people never see.

### 3. Data source
The TPC-H models from P07 (DuckDB) and the Gold tables from P05 (Databricks). No new data; dbt reads both over their native connections.
```bash
pip install dbt-databricks
dbt build --target duckdb
dbt build --target databricks
```
Bad data is **generated**, because you need a breach you control:
```bash
python -m src.inject_bad_data --kind volume_drop  --pct 60
python -m src.inject_bad_data --kind null_spike   --col fare_amount
python -m src.inject_bad_data --kind distribution --col trip_distance --shift 3x
```
Three different failures — **a null check catches exactly one of them.** Finding out which is the exercise.

### 4. What you build

**Step 1 — Two targets, one project.** Add Databricks alongside DuckDB in `profiles.yml`. Run the same models against each. **Write down every difference you hit** — type names, date functions, incremental strategy, merge behaviour. That list is the deliverable of Step 1.

**Step 2 — Isolate the differences.** Where SQL diverges, use `{{ adapter.dispatch() }}` or `{{ target.type }}` rather than forking the model. **Keep one codebase.** This is the real technique multi-warehouse teams use.

**Step 3 — Install dbt-expectations.** Tests beyond null and unique: distribution, row-count ranges, value bounds, freshness.

**Step 4 — Volume anomaly test.** Fail if today's row count is more than 40% below the 7-day average. **This catches the failure class null checks never will** — a partial upstream load, which looks perfectly valid row by row.

**Step 5 — Distribution test.** Fail if the mean of `trip_distance` moves more than 3 standard deviations from its trailing average. Catches unit changes, a broken join, sensor drift.

**Step 6 — Set severities deliberately.** `warn` for things you want to know about; `error` for things that must stop the pipeline. **Getting this wrong in either direction is costly:** everything `error` and you're paged nightly for nothing; everything `warn` and nothing ever stops.

**Step 7 — Prove the gate.** Inject each of the three breaches in turn. Confirm the pipeline **fails** and names what broke. A gate you haven't seen fire is not a gate.

**Step 8 — Exposures.** Declare which dashboards depend on which models. Now `dbt ls --select +exposure:revenue_dashboard` answers "if I change this, what breaks?" **before** you change it.

### 5. The AI part — Threshold Proposer (~50 lines)

Profile each table (row counts over 30 days, null rates, cardinality, mean and stddev per numeric column), send the profile in one call, and ask for proposed test thresholds **with reasoning**.

Then pressure-test every suggestion: **a threshold too tight pages you every Monday; too loose and it never fires.** Backtest against your actual history before accepting one.

**The honest limitation, and the lesson:** the model cannot know your seasonality. It doesn't know taxi volume drops on public holidays or that month-end spikes orders. It will propose a tight volume threshold that fires every December 25th. **Knowing where the handoff from model to human judgement sits is the actual skill here** — the model proposes, your domain knowledge disposes.

### 6. What you learn
**dbt:** multiple adapters · `adapter.dispatch()` for cross-warehouse SQL · dbt-expectations · test severity · exposures and impact analysis · source freshness
**Data engineering:** **data quality as a contract** · anomaly detection vs constraint checking · blocking publication over serving bad data · alert fatigue as a real failure mode
**AI:** proposing thresholds from profiles · recognising where domain knowledge beats the model

### 7. Done checklist
- [ ] One project builds green on both DuckDB and Databricks
- [ ] Warehouse differences documented in a list
- [ ] No forked models — differences handled with dispatch
- [ ] All three injected breaches caught, each by a different test
- [ ] Severities assigned deliberately, with a written rationale
- [ ] Thresholds backtested against real history, not accepted as proposed
- [ ] Exposures let you answer "what breaks if I change this?"
- [ ] `AI_NOTES.md` records at least one threshold you overrode, and why

### 8. In production this would be
Quality gates tied to consumer SLAs, with a contract per critical table · a quarantine table rather than a hard stop for some pipelines, so partial data still flows while flagged · thresholds reviewed quarterly as volume grows · an on-call rotation for data quality alerts, which makes alert fatigue a genuine operational risk · exposures auto-generated from BI metadata rather than hand-maintained.

### 9. Git

```bash
git switch -c p10-quality-gates
```

**Commit at these points** — each one is a working step, not a save point:
- `feat: databricks target alongside duckdb`
- `refactor: dispatch instead of forked models`
- `feat: volume anomaly test`
- `feat: distribution drift test`
- `test: prove each injected breach is caught`
- `feat: exposures for impact analysis`
- `docs: AI_NOTES on thresholds you overrode`

**PR description should answer:** which test caught which breach, and which breach a null check would have missed.

```bash
gh pr create --fill && gh pr merge --squash --delete-branch
git switch main && git pull && git tag p10 && git push --tags
```

---

[← P09](p09-dbt-analytics.md) · [Roadmap](../PROJECTS.md) · [P11 →](p11-airflow-orchestration.md)
