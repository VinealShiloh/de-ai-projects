> Part of **04 · DELTA LAKE & ICEBERG (Days 19–24)** · [All projects](../PROJECTS.md) · Conventions: [AI rules](../PROJECTS.md#the-ai-rules) · [Git workflow](../PROJECTS.md#git--how-you-work)

## P07 — Delta Lake Deep Dive · Days 19–21

### 1. Goal
Build a customer dimension that tracks history: when a customer's details change, you keep the old version and record exactly when each was valid. Then break the table on purpose and recover it.

### 2. Why this project exists
"What was this customer's address *at the time they placed that order?*" is a question your Gold tables from P03 cannot answer, because they only hold the current state. **Slowly Changing Dimension Type 2** is how the industry answers it, and `MERGE` is how you implement it. This is also where you learn what a transaction log *is*, by reading one.

### 3. Data source
**Generated, not downloaded** — you need *controlled* change, which no public dataset gives you.
```bash
pip install delta-spark faker
python -m src.seed --rows 1000000 --out data/batch1.parquet --seed 42
python -m src.seed --rows 1000000 --out data/batch2.parquet --seed 42 \
       --mutate-pct 10 --new-pct 5 --late-arriving-pct 2
```
Batch 2 deliberately contains: updates to existing customers (10%), brand new customers (5%), and records back-dated to *before* batch 1 (2%). **That last 2% is what breaks naive SCD2 implementations**, and finding it is the project. Fix the seed so runs are reproducible.

### 4. What you build

**Step 1 — Write a Delta table and read the log.** Write batch 1 as Delta. Then open `data/dim_customer/_delta_log/00000000000000000000.json` and read it. You'll see the files added and the schema. **Understand that a Delta table is Parquet files plus this log** — nothing more. Everything else follows from that.

**Step 2 — Simple upsert with MERGE.** Merge batch 2, updating matched rows and inserting new ones. This is SCD Type 1 — history is overwritten. Note what you just lost.

**Step 3 — SCD Type 2 by hand.** Now keep history. Add `valid_from`, `valid_to`, `is_current`. When a customer changes: close the old row (`valid_to = now`, `is_current = false`) and insert a new one. The two-step MERGE pattern is fiddly — **write it yourself, because this is a standard interview question.**

**Step 4 — Handle the late arrivals.** A record dated *before* your current row's `valid_from` must be inserted in the middle of the history, not at the end. Your Step 3 code almost certainly gets this wrong. **Find out, then fix it.**

**Step 5 — Time travel.**
```sql
SELECT * FROM delta.`data/dim_customer` VERSION AS OF 2;
SELECT * FROM delta.`data/dim_customer` TIMESTAMP AS OF '2026-11-20';
```

**Step 6 — Break it, then restore it.** Run a `DELETE` with a wrong `WHERE`. Observe the damage. Then `RESTORE TABLE ... TO VERSION AS OF <n>`. **This is the real incident-response tool** — "what did this table look like before the bad deploy" becomes a query instead of a disaster.

**Step 7 — Maintenance, measured.** Count files before and after `OPTIMIZE`. Time a filtered query before and after `ZORDER BY customer_id`. Run `VACUUM` and understand the retention window: **it deletes files time travel depends on.** Short retention plus a long-running reader is a classic production outage.

**Step 8 — Constraints and schema evolution.** Add a `CHECK` constraint; watch a violating write fail. Add a new column with `mergeSchema`; watch it succeed. Know which one protects you and which one lets things through.

### 5. The AI part — Adversarial Test Generator (~45 lines)

**Flip the usual roles: the model is the attacker, not the author.**

Send your SCD2 MERGE statement in one call and ask: *list 15 inputs that would corrupt this dimension.* Then turn every plausible one into a `pytest` case.

Models are markedly better at adversarial thinking than at correct implementation — that asymmetry is the lesson. Expect it to surface: duplicate business keys in one batch, a record whose dates overlap an existing version, a delete followed by a re-insert, a null in the business key, two updates for the same key in one batch. **The late-arriving case from Step 4 is usually in its list** — which is why you run this *before* you think you're finished.

### 6. What you learn
**Delta:** ACID on object storage · the transaction log · optimistic concurrency · `MERGE` semantics · **SCD Type 2** · time travel · `RESTORE` · `OPTIMIZE`/`ZORDER` · `VACUUM` retention · constraints vs schema evolution · change data feed
**Data engineering:** temporal modelling · late-arriving data · table maintenance as scheduled work
**AI:** using a model adversarially · generating test cases rather than implementations

### 7. Done checklist
- [ ] You can explain what's inside `_delta_log/` without looking
- [ ] SCD2 handles updates, inserts **and** late arrivals correctly
- [ ] For any row you can state when it was valid, proven by a time-travel query
- [ ] You recovered from a deliberate bad `DELETE` using `RESTORE`
- [ ] `OPTIMIZE` and `ZORDER` effects measured, not assumed
- [ ] You can explain why `VACUUM` with short retention is dangerous
- [ ] ≥10 tests, at least 5 from the adversarial list
- [ ] `AI_NOTES.md` records which attacks were real and which were noise

### 8. In production this would be
`MERGE` without partition pruning in the match condition is a classic outage — it rewrites the whole table · `VACUUM` retention set well above your longest-running query · concurrent writers to one table will conflict, making partitioning a concurrency decision as much as a performance one · `OPTIMIZE` scheduled off-peak · SCD2 dimensions reconciled against source counts nightly, because silent drift is the norm.

### 9. Git

```bash
git switch -c p07-delta-deep-dive
```

**Commit at these points** — each one is a working step, not a save point:
- `feat: seeded dimension generator`
- `feat: SCD Type 1 merge`
- `feat: SCD Type 2 with validity windows`
- `fix: handle records back-dated before valid_from`
- `feat: restore from a bad delete`
- `perf: OPTIMIZE and ZORDER, file count and query time recorded`
- `test: adversarial cases from the AI attack list`

**PR description should answer:** the late-arriving bug, how you found it, and how you fixed it.

**The `fix:` commit for late arrivals is the one to write carefully** — it's a standard interview question and your own commit is the best revision note.

```bash
gh pr create --fill && gh pr merge --squash --delete-branch
git switch main && git pull && git tag p07 && git push --tags
```

---

[← P06](p06-bundles-cicd.md) · [Roadmap](../PROJECTS.md) · [P08 →](p08-iceberg-comparison.md)
