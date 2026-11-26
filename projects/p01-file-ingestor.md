> Part of **01 · PYTHON (Days 1–6)** · [All projects](../PROJECTS.md) · Conventions: [AI rules](../PROJECTS.md#the-ai-rules) · [Git workflow](../PROJECTS.md#git--how-you-work)

## P01 — File Ingestor · Days 1–3

### 1. Goal
Build a program that reads messy data files and sorts every row into one of two piles: **good** and **bad**. Good rows are written out clean. Bad rows are written out with a reason attached. Nothing is ever silently dropped.

### 2. Why this project exists
This is the most common job in data engineering. Before anything is transformed, joined or charted, something must read a file and decide which rows are trustworthy. Spark, dbt and Databricks are all fancier versions of this same decision. You also learn the habit that matters most: **bad data is quarantined, never deleted.**

### 3. Data source

**File 1 — NYC 311 complaints (messy CSV).** Real complaints filed by New Yorkers. Free text, inconsistent capitalisation, missing values, mixed date formats.
```bash
mkdir -p data/raw
curl -G "https://data.cityofnewyork.us/resource/erm2-nwe9.csv" \
  --data-urlencode '$limit=500000' \
  --data-urlencode "\$where=created_date between '2024-01-01T00:00:00' and '2024-02-01T00:00:00'" \
  -o data/raw/nyc311_2024-01.csv
```
~500,000 rows, ~150 MB.

**File 2 — GH Archive (nested JSON).** Every public GitHub action, one JSON object per line, heavily nested.
```bash
curl -O --output-dir data/raw https://data.gharchive.org/2024-01-15-0.json.gz
```
~74 MB compressed, 1 hour of activity.

> Start with File 1 only. Add File 2 once File 1 works.

### 4. What you build

**Step 1 — Read without loading into memory.** A function that reads the CSV one row at a time using a generator. Print the row count. *It must finish on 150 MB without memory climbing.*

**Step 2 — Write the schema contract.** Create `contracts/nyc311.yaml` listing each column: name, type, nullable, valid range or pattern.
```yaml
columns:
  - {name: unique_key,    type: integer,   nullable: false}
  - {name: created_date,  type: timestamp, nullable: false}
  - {name: complaint_type,type: string,    nullable: false}
  - {name: incident_zip,  type: string,    nullable: true, pattern: '^[0-9]{5}$'}
```

**Step 3 — Validate each row.** A function taking one row, returning `Ok(cleaned)` or `Reject(row, reason)`. The reason must be specific: `"incident_zip: '1OO23' does not match ^[0-9]{5}$"`.

**Step 4 — Write three outputs.** `data/clean/` (passed), `data/rejected/` (failed, with reason), `data/runs/<run_id>.json` (rows read, passed, rejected, top 5 reasons, duration).

**Step 5 — Make it idempotent.** Run twice on the same file; output must be identical, not doubled. Delete-and-rewrite the target partition before writing.

**Step 6 — CLI and logging.**
```bash
python -m src.ingest --input data/raw/nyc311_2024-01.csv --schema contracts/nyc311.yaml --out data/
```
Use `logging`, not `print`.

**Step 7 — Tests.** `pytest` covering: valid row passes; null in non-nullable rejects; bad ZIP rejects; empty file doesn't crash; all-bad file produces zero clean output and doesn't crash.

### 5. The AI part — Reject Explainer (~40 lines)

After a run you have a pile of rejected rows and one question: **is my contract wrong, or is the data actually bad?** That's slow for a human and there's no deterministic rule for it — so it's a fair use of a model.

Send the top 20 rejection reasons plus 3 example rows each, in **one** call, and ask for a verdict per reason.

```python
# src/ai/explain_rejects.py
import anthropic, json

PROMPT = """You are reviewing rejected rows from a data ingestion run.
For each rejection reason below, answer in one line:
  reason | LIKELY_BAD_DATA or LIKELY_BAD_CONTRACT | why
If a rule rejects more than 20% of rows, the rule is probably wrong.

{evidence}"""

def explain(summary: dict) -> str:
    client = anthropic.Anthropic()
    msg = client.messages.create(
        model="claude-haiku-5-5",          # cheap; small job
        max_tokens=1000,
        messages=[{"role": "user", "content": PROMPT.format(evidence=json.dumps(summary, indent=2))}],
    )
    return msg.content[0].text
```
Run it separately, never inside the ingestion loop:
```bash
python -m src.ingest --input ... --out data/
python -m src.ai.explain_rejects --run-id <run_id>
```

### 6. What you learn
**Python:** generators and flat memory · `csv`/`json` stdlib · dataclasses · exception handling that doesn't swallow errors · `argparse`, `logging`, `pathlib` · `pytest`
**Data engineering:** schema contracts · quarantine over deletion · idempotency · run metadata
**AI:** calling the API · batching evidence into one call · prompting for a parseable format · treating output as advice

### 7. Done checklist
- [ ] Runs on 150 MB without memory climbing
- [ ] Running twice gives identical output, not doubled
- [ ] A corrupted row at line 300,000 is quarantined with a clear reason; the run completes
- [ ] `pytest` passes, ≥5 tests
- [ ] Summary JSON adds up: read = passed + rejected
- [ ] `AI_NOTES.md` records one thing the explainer got right and one it got wrong

### 8. In production this would be
Event-triggered (S3 `ObjectCreated`), not a manual command · **failing** the run above a reject-rate threshold rather than quarantining 40% silently · contract in a shared registry, not beside the code · source files never deleted, marked processed in a manifest so you can replay · alerting when *new* reject reasons appear, because that usually means an upstream team changed something without telling you.

### 9. Git

```bash
git switch -c p01-file-ingestor
```

**Commit at these points** — each one is a working step, not a save point:
- `chore: project skeleton and schema contract`
- `feat: streaming CSV reader`
- `feat: row validation against contract`
- `feat: clean/rejected/summary outputs`
- `fix: make re-runs idempotent`
- `test: validation edge cases`
- `feat: reject explainer (AI)`
- `docs: README and AI_NOTES`

**PR description should answer:** what you quarantined and why you chose quarantine over dropping.

```bash
gh pr create --fill && gh pr merge --squash --delete-branch
git switch main && git pull && git tag p01 && git push --tags
```

---

— · [Roadmap](../PROJECTS.md) · [P02 →](p02-api-extractor.md)
