> Part of **05 · DBT (Days 25–30)** · [All projects](../PROJECTS.md) · Conventions: [AI rules](../PROJECTS.md#the-ai-rules) · [Git workflow](../PROJECTS.md#git--how-you-work)

## P09 — Analytics Engineering · Days 25–27

### 1. Goal
Turn a pile of SQL queries into a tested, documented, dependency-aware project where changing one model shows you everything downstream that could break.

### 2. Why this project exists
You already write good SQL. What you're missing is what happens when 40 queries depend on each other and three people edit them. dbt's value isn't the SQL — it's that the dependency graph, the tests and the docs are version-controlled and enforced in CI. **This is the fastest project in the course for you, and the highest ratio of career value to effort.**

### 3. Data source
**TPC-H generated locally by DuckDB** — zero download, standard schema, scalable:
```bash
pip install dbt-core dbt-duckdb duckdb
python - <<'EOF'
import duckdb
con = duckdb.connect("warehouse.duckdb")
con.sql("INSTALL tpch; LOAD tpch; CALL dbgen(sf=1);")   # ~6M lineitem rows, ~1 GB
EOF
```
`sf=1` for development; `sf=10` once incremental models work — a one-character change that reveals whether "incremental" was ever real.

### 4. What you build

**Step 1 — Project skeleton.** `dbt init`, point `profiles.yml` at DuckDB, `dbt debug` until green.

**Step 2 — Declare sources.** Every raw table in `sources.yml` with a freshness threshold. From now on **every reference goes through `{{ source() }}` or `{{ ref() }}`** — a hardcoded table name anywhere is a bug, because it's invisible to the dependency graph.

**Step 3 — Staging layer.** One model per source table. Rename columns to your conventions, cast types, no joins and no business logic. Materialise as views. **Staging is the only place allowed to know what the source looks like** — that's what makes a source change a one-file fix.

**Step 4 — Intermediate layer.** The joins and the hard logic. Not exposed to users. Materialise as ephemeral or view.

**Step 5 — Marts layer.** `fct_orders` plus conformed dimensions. **Declare the grain in the model's description** — one row per what? Materialise as tables.

**Step 6 — Tests, two kinds.** Schema tests (`unique`, `not_null`, `relationships`, `accepted_values`) on every key column. Plus two *singular* tests — a SQL query returning rows only when something is wrong. Good ones: orders whose total doesn't match the sum of their lines; customers in the fact table missing from the dimension.

**Step 7 — One incremental model.** Use `is_incremental()` with a defensible unique key. Then run `--full-refresh` and **diff the results against the incremental version.** If they differ, your incremental logic is wrong — and this silent drift is the most common dbt bug in production.

**Step 8 — Macros and docs.** One macro for repeated logic. Then `dbt docs generate && dbt docs serve` and read the lineage graph. Change a staging model and trace what's downstream.

```bash
dbt build                        # run + test together, always
dbt test --select fct_orders+    # everything downstream of a model
```

### 5. The AI part — Bulk Staging Model Generator (~50 lines)

**This is the single highest-leverage real-world AI use in the whole course**, and what analytics teams genuinely do today. Writing 20 staging models by hand is hours of mechanical typing.

Give the model your source schema (column names and types from `information_schema`) plus **one** staging model you wrote by hand as a style example, and have it generate the rest, plus their `schema.yml`.

**Then review every line**, looking for the two reliable failure modes:
1. **Invented column descriptions.** It will confidently describe `o_orderstatus` as something plausible and wrong. You have the TPC-H spec — check it.
2. **Tests that assert nothing.** `not_null` on a column that's never null is noise that passes forever and catches nothing.

**Note the exception to the Day 18 rule:** this is generation, not advice, and it's allowed here because you've already written one model by hand and can read SQL fluently. You're reviewing a draft in a language you know well — which is exactly the condition under which generation is safe.

### 6. What you learn
**dbt:** project structure and layering · Jinja and `ref()`/`source()` · materialisations (view/table/incremental/ephemeral) · schema vs singular tests · incremental models and `is_incremental()` · macros · snapshots · docs and lineage
**Data engineering:** dimensional modelling in practice · grain as a first-class decision · the DAG as documentation · tests as a merge gate
**AI:** bulk generation from a schema with a style example · reviewing generated SQL for plausible-but-wrong content

### 7. Done checklist
- [ ] `dbt build` green from a clean checkout
- [ ] No hardcoded table names — `git grep` for the raw schema finds nothing outside `sources.yml`
- [ ] Three layers with a clear rule for what belongs in each
- [ ] Every mart model has a declared grain in its description
- [ ] ≥2 singular tests that catch real logic errors
- [ ] Incremental and `--full-refresh` produce identical results
- [ ] Lineage graph explains itself to someone who's never seen the project
- [ ] `AI_NOTES.md` lists every invented description you caught

### 8. In production this would be
`dbt build` on every PR against a sample or a zero-copy clone, blocking merges on failure · incremental models full-refreshed on a schedule because they drift · naming conventions enforced in CI, since an unchecked project becomes 900 models nobody can reason about · model ownership assigned, with each mart having a named owner · `dbt source freshness` alerting before consumers notice.

### 9. Git

```bash
git switch -c p09-dbt-analytics
```

**Commit at these points** — each one is a working step, not a save point:
- `chore: dbt project and duckdb profile`
- `feat: sources with freshness thresholds`
- `feat: staging layer`
- `feat: intermediate and marts with declared grain`
- `test: schema tests plus two singular tests`
- `feat: incremental model`
- `fix: incremental and full-refresh now agree`
- `docs: lineage screenshot and AI_NOTES`

**PR description should answer:** your grain decision for the fact table, and why.

```bash
gh pr create --fill && gh pr merge --squash --delete-branch
git switch main && git pull && git tag p09 && git push --tags
```

---

[← P08](p08-iceberg-comparison.md) · [Roadmap](../PROJECTS.md) · [P10 →](p10-quality-gates.md)
