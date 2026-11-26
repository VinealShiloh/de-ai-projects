> Part of **03 · DATABRICKS (Days 13–18)** · [All projects](../PROJECTS.md) · Conventions: [AI rules](../PROJECTS.md#the-ai-rules) · [Git workflow](../PROJECTS.md#git--how-you-work)

## P06 — Asset Bundles + CI/CD · Days 16–18

### 1. Goal
Make `git push` the only way code reaches Databricks. You should never again click "Run" in the UI to ship something.

### 2. Why this project exists
This is the difference between *"I used Databricks"* and *"I ran Databricks."* Most candidates cannot do this, and it's what separates a data engineer from an analyst who writes notebooks. Everything here — tests in CI, environment promotion, deploys from a merge — is standard software engineering that the data world adopted late.

### 3. Data source
Nothing new. The data already sits in Unity Catalog. What you add is **test fixtures**: a 1,000-row sample committed to the repo so CI can run tests without a cluster or cloud access.
```bash
python -c "import pandas as pd; pd.read_parquet('data/raw/yellow/yellow_tripdata_2024-01.parquet').head(1000).to_parquet('tests/fixtures/trips_sample.parquet')"
```
Fixtures are committed; real data never is. That split *is* the lesson.

### 4. What you build

**Step 1 — Get the logic out of notebooks.** Move transformations into `src/taxi/transforms.py` as plain functions. The notebook (or job entry point) becomes a thin caller. **Notebooks can't be unit-tested, diffed properly, or imported** — that's why this matters.

**Step 2 — Tests that run with no cluster.** `pytest` against local Spark using the committed fixture. Must finish in under a minute.

**Step 3 — Write the bundle.** `databricks.yml` defining resources (your job) and two targets, `dev` and `prod`, differing in catalog, schedule and cluster size.
```bash
databricks bundle validate
databricks bundle deploy --target dev
databricks bundle run medallion_job --target dev
```

**Step 4 — Secrets in a scope.** No tokens in code or in `databricks.yml`. Create a secret scope; reference secrets by name.

**Step 5 — GitHub Actions.** On PR: install deps, run `pytest`, run `databricks bundle validate`. On merge to `main`: deploy to prod. Branch protection so `main` can't be pushed to directly.

**Step 6 — Break it on purpose.** Push a commit with a failing test. Watch CI block the merge. Fix it. **This is the proof the gate is real** — an untested CI gate is decoration.

**Step 7 — Local development loop.** Set up `databricks-connect` so you can run against a remote cluster from your editor without deploying.

### 5. The AI part — CI Failure Triage (~40 lines)

**Honest note: this is one of the thinner AI steps in the course.** CI logs are usually readable by a human in 30 seconds, and a model adds little. I'm including it for one specific case where it genuinely helps — Spark stack traces, which are 200 lines of JVM noise wrapping one real Python error.

Pipe a failed run's log into a call that extracts: the actual error, the file and line, and the most likely cause. Useful mainly as a *log compressor*.

If you find it doesn't earn its place after a week, **delete it and write that decision in `AI_NOTES.md`.** Knowing when not to use AI is a real result, and saying so is more credible than pretending every step was valuable.

### 6. What you learn
**Databricks:** Asset Bundles · environment promotion · `databricks-connect` · secret scopes · testing Spark without a cluster
**Engineering:** CI/CD for data · branch protection · fixtures vs real data · separating logic from execution context
**AI:** recognising when a model isn't adding value, and removing it

### 7. Done checklist
- [ ] No business logic left in a notebook
- [ ] `pytest` runs in CI in under a minute, no cloud access
- [ ] `databricks bundle deploy --target dev` works from a clean checkout
- [ ] A failing test blocks a merge — demonstrated, not assumed
- [ ] Merging to `main` deploys to prod with no manual step
- [ ] No secret in the repo
- [ ] `AI_NOTES.md` has a verdict on whether the triage step was worth keeping

### 8. In production this would be
A deploy is a merged PR, always · dev/staging/prod as separate workspaces or at minimum separate catalogs · prod data never read from dev · bundle configs reviewed like code because they control cluster spend · a rollback path, usually redeploying the previous tag · deploy notifications to the team channel.

### 9. Git

```bash
git switch -c p06-bundles-cicd
```

**Commit at these points** — each one is a working step, not a save point:
- `refactor: move logic out of notebooks into modules`
- `test: fixtures committed, tests run without a cluster`
- `feat: databricks.yml with dev and prod targets`
- `ci: GitHub Actions for test and deploy`
- `test: prove a failing test blocks the merge`
- `docs: AI_NOTES verdict on the triage step`

**PR description should answer:** why notebooks were the problem, and what replaced them.

**This is the project where the PR matters most** — the CI you build here runs on every later PR.

```bash
gh pr create --fill && gh pr merge --squash --delete-branch
git switch main && git pull && git tag p06 && git push --tags
```

---

[← P05](p05-lakehouse-medallion.md) · [Roadmap](../PROJECTS.md) · [P07 →](p07-delta-deep-dive.md)
