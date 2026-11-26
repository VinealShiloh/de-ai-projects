> Part of **06 · AIRFLOW (Days 31–36)** · [All projects](../PROJECTS.md) · Conventions: [AI rules](../PROJECTS.md#the-ai-rules) · [Git workflow](../PROJECTS.md#git--how-you-work)

## P12 — Dynamic & Data-Aware DAGs · Days 34–36

### 1. Goal
Stop scheduling by clock. Make downstream pipelines trigger when their input data **actually arrives**, and generate tasks from config instead of writing them by hand.

### 2. Why this project exists
"Run at 3am and hope the 2am job finished" is the most common fragility in data platforms. It works until a run is slow, then it silently processes yesterday's data. **Data-aware scheduling is the fix**, and dynamic task mapping is how you handle 50 sources without writing 50 DAGs. Both are modern Airflow features that many working engineers haven't adopted.

### 3. Data source
**Three sources on deliberately different cadences** — the whole reason data-aware scheduling exists:

| Source | Cadence | Where from |
|---|---|---|
| TLC trips | daily | `data/landing/` from P11 |
| GitHub events | every 15 min | your P02 extractor |
| NYC 311 | hourly | Socrata API, filtered on `created_date` |

```bash
python -m src.multi_source --config conf/sources.yaml --dry-run
```

### 4. What you build

**Step 1 — Config-driven sources.** `conf/sources.yaml` listing each source: name, type, cadence, target table. **Adding a fourth source must be a YAML edit, not a code change.** That's the deliverable.

**Step 2 — Dynamic task mapping.** Generate one ingestion task per source at runtime with `.expand()`. One DAG, N tasks, driven by the config.

**Step 3 — Define Datasets (Assets).** Declare that your silver task *produces* a dataset and the gold DAG *consumes* it:
```python
silver_trips = Dataset("s3://lake/silver/trips")

@task(outlets=[silver_trips])
def build_silver(): ...

@dag(schedule=[silver_trips])     # runs when silver updates — no clock
def gold_pipeline(): ...
```
**The gold DAG now has no schedule at all.** It runs when its input is ready. Verify by triggering silver manually and watching gold self-trigger.

**Step 4 — Multiple upstreams.** Make a DAG that waits on *two* datasets. It should run only when both have updated — try it with sources on different cadences.

**Step 5 — A custom operator.** Wrap repeated logic (say, "extract from REST API with watermark") as a reusable operator with a hook for the connection. **This is how teams avoid copy-pasting the same 40 lines into every DAG.**

**Step 6 — Pools and concurrency.** Put all API tasks in a pool with limited slots, so 20 parallel tasks don't trip the rate limit you so carefully handled in P02.

**Step 7 — DAG tests in CI.** Extend P11's tests: no import errors, no cycles, every task has an owner and retries, no `datetime.now()` in any DAG, every dataset producer has a consumer. **These run on every PR** — a broken DAG takes down the scheduler for everyone, so this gate is not optional.

### 5. The AI part — PR Reviewer with Your Own Checklist (~60 lines)

Build a GitHub Action that posts a Claude review on every PR, driven by **your own data engineering checklist** rather than a generic "review this code" prompt.

Your checklist, accumulated from P01–P11: idempotency, partition pruning in MERGE conditions, hardcoded dates, secrets, top-level DAG code, `inferSchema`, per-row API calls, missing tests.

**Then measure it.** Run it over your last six projects' diffs and count: real findings, false positives, things it missed that you know are there. Put the numbers in `AI_NOTES.md`.

**This is a portfolio piece on its own**, and the measurement is what makes it one. The lesson is sharp: a generic review prompt produces confident slop; a domain checklist built from your own experience produces a reviewer worth having. **The checklist is the asset, not the model.**

### 6. What you learn
**Airflow:** dynamic task mapping · **Datasets/Assets and data-aware scheduling** · custom operators and hooks · pools and concurrency limits · cross-DAG dependencies · DAG testing
**Data engineering:** config-driven pipelines over hand-written ones · event-driven over time-driven orchestration · reusability as a maintenance strategy
**AI:** domain checklists over generic prompts · measuring a reviewer's precision and recall · automating review in CI

### 7. Done checklist
- [ ] Adding a source is a YAML edit only
- [ ] Tasks generated dynamically, not hand-written
- [ ] **No DAG in the project scheduled by guessing when upstream finishes**
- [ ] A multi-dataset DAG waits correctly for both inputs
- [ ] A custom operator replaces repeated logic in ≥2 DAGs
- [ ] A pool prevents API rate-limit breaches under parallelism
- [ ] CI blocks a PR containing a broken DAG — demonstrated
- [ ] `AI_NOTES.md` has the reviewer's precision/recall numbers

### 8. In production this would be
Data-aware scheduling replacing clock-chaining across the whole platform · DAG factories generating hundreds of DAGs from a registry · pools mapped to real downstream capacity limits · the PR reviewer as a *supplement* to human review, never a replacement, since it has no idea what the change is for · custom operators versioned and released as an internal package.

### 9. Git

```bash
git switch -c p12-dynamic-dags
```

**Commit at these points** — each one is a working step, not a save point:
- `feat: config-driven source registry`
- `feat: dynamic task mapping from config`
- `feat: dataset-triggered downstream DAG`
- `feat: custom operator for API extraction`
- `feat: pool limiting API concurrency`
- `ci: AI PR reviewer with DE checklist`
- `docs: AI_NOTES with reviewer precision and recall`

**PR description should answer:** why clock-based scheduling was fragile, with a concrete example.

```bash
gh pr create --fill && gh pr merge --squash --delete-branch
git switch main && git pull && git tag p12 && git push --tags
```

---

[← P11](p11-airflow-orchestration.md) · [Roadmap](../PROJECTS.md) · [P13 →](p13-streaming-ingest.md)
