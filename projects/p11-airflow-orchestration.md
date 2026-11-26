> Part of **06 · AIRFLOW (Days 31–36)** · [All projects](../PROJECTS.md) · Conventions: [AI rules](../PROJECTS.md#the-ai-rules) · [Git workflow](../PROJECTS.md#git--how-you-work)

## P11 — Orchestrating the Pipeline · Days 31–33

### 1. Goal
Make your P03 taxi pipeline run automatically every day, retry when it fails, and — the real test — **re-run correctly for any 30-day period in the past.**

### 2. Why this project exists
Everything so far you've run by typing a command. Production doesn't work that way. But the deep lesson isn't scheduling, it's **backfill**: being able to reprocess history correctly. That's why every project since P01 insisted on idempotency and `--date` parameters. **This project is where that discipline pays off** — and if you'd cheated on it, you'd find out here.

### 3. Data source
The TLC Parquet from P03, reorganised into date-partitioned landing folders so a sensor has something real to wait for:
```bash
python -m src.explode_to_daily --input data/raw/yellow --out data/landing/
# → data/landing/date=2024-01-01/ … date=2024-01-31/
```
One folder per day is what makes backfill meaningful: 30 folders, 30 independent runs, each correct in isolation and identical whether run today or in six months.

### 4. What you build

**Step 1 — Airflow running locally.**
```bash
curl -LfO https://airflow.apache.org/docs/apache-airflow/stable/docker-compose.yaml
docker compose up -d        # UI at localhost:8080
```
Mount your project and `data/landing/` into the containers.

**Step 2 — A trivial DAG first.** Two tasks that print. Confirm scheduling works before adding real work. **Debugging Airflow and debugging your pipeline at the same time is miserable** — separate the two.

**Step 3 — Port the real pipeline.** bronze → silver → gold as three tasks with dependencies. Each task **submits** work; it does not do the work. Use `BashOperator` or `SparkSubmitOperator` to invoke your existing scripts. **Airflow is a scheduler, not a compute engine** — logic inside a task is the most common Airflow mistake.

**Step 4 — Understand logical date vs wall-clock.** This is the concept people get wrong for years. A DAG run for `2024-01-15` may execute on `2024-01-16`, or during a backfill, next March. **Your task must use the logical date (`{{ ds }}`) for everything** — never `datetime.now()`. One `now()` makes backfill impossible.

**Step 5 — Retries and timeouts.** `retries=2`, `retry_delay`, and a sensible `execution_timeout`. Then make a task fail deliberately and watch it retry. **A retry policy you've never seen fire is a guess.**

**Step 6 — A sensor.** Wait for the day's landing folder before starting. Use a **deferrable** sensor — a classic sensor occupies a worker slot while sleeping, and enough of them deadlock the cluster.

**Step 7 — Backfill, the main event.**
```bash
airflow dags test taxi_medallion 2024-01-15                      # one run, no scheduler
airflow dags backfill -s 2024-01-01 -e 2024-01-30 taxi_medallion
```
Then verify: **the 30-day backfill must produce exactly the same Gold data as 30 individual daily runs.** Diff them. If they differ, something isn't idempotent, and that's the bug this project exists to surface.

**Step 8 — Alerting and DAG tests.** Email or webhook on failure. Plus `pytest` on DAG structure: imports cleanly, no cycles, every task has an owner and retries.

### 5. The AI part — Idempotency Auditor (~50 lines)

**This one is deliberately set up for the model to fail, and that's the point.**

First, have Claude generate a DAG from your pipeline description. It will almost certainly produce something non-idempotent — `datetime.now()` instead of `{{ ds }}`, an append instead of a partition overwrite, or a task that assumes yesterday succeeded. **Find the bug yourself. Document it.**

Then invert it: write an auditor that takes a DAG file and asks for **specific, evidence-backed** idempotency violations:
```python
PROMPT = """Review this Airflow DAG for idempotency violations.
For each issue:  line | violation | why re-running breaks | fix
Only report issues you can point at a line for. Do not speculate."""
```
Run it over all your DAGs. **Score it in `AI_NOTES.md`.** The sequence — model writes a bug, you catch it, then you use the model to catch that class of bug — is a genuinely strong portfolio story: it shows you understand the failure mode rather than just the tool.

### 6. What you learn
**Airflow:** DAGs and scheduling · **logical date vs execution time** · task dependencies · retries, timeouts, SLAs · sensors and deferrable operators · **backfill** · connections and variables · pools and concurrency · TaskFlow API and XComs
**Data engineering:** orchestration as submission, not execution · idempotency as a hard requirement · DAG tests in CI
**AI:** catching a model's own generated bug · evidence-constrained prompts ("point at a line")

### 7. Done checklist
- [ ] A trivial DAG ran before you added real work
- [ ] All three stages run with dependencies and retries
- [ ] No `datetime.now()` anywhere — `{{ ds }}` everywhere
- [ ] No business logic inside a task; tasks submit only
- [ ] A deliberate failure retried twice and then alerted
- [ ] **30-day backfill output is identical to 30 single runs** (diffed)
- [ ] DAG integrity tests pass in `pytest`
- [ ] `AI_NOTES.md` documents the bug in the generated DAG

### 8. In production this would be
Real work submitted to Spark/Databricks/the warehouse, never executed in the task · **top-level code in a DAG file runs every 30 seconds on the scheduler**, so an API call there takes the whole instance down for everyone · managed Airflow (MWAA, Composer, Astronomer) rather than self-hosted · DAGs generated from config instead of 300 hand-written files · SLA misses alerting separately from failures, since late is a different problem from broken.

### 9. Git

```bash
git switch -c p11-airflow-orchestration
```

**Commit at these points** — each one is a working step, not a save point:
- `chore: airflow docker compose`
- `feat: trivial DAG to prove scheduling`
- `feat: medallion DAG with dependencies and retries`
- `fix: replace datetime.now() with logical date`
- `feat: deferrable sensor for landing folder`
- `test: DAG integrity tests`
- `docs: backfill diff proving 30 runs == 1 backfill`

**PR description should answer:** what backfill proved about your earlier idempotency work.

**Commit the backfill diff output.** It's the evidence that everything since P01 was built correctly.

```bash
gh pr create --fill && gh pr merge --squash --delete-branch
git switch main && git pull && git tag p11 && git push --tags
```

---

[← P10](p10-quality-gates.md) · [Roadmap](../PROJECTS.md) · [P12 →](p12-dynamic-dags.md)
