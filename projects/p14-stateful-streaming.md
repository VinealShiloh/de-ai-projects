> Part of **07 · STREAMING — KAFKA + SPARK (Days 37–42)** · [All projects](../PROJECTS.md) · Conventions: [AI rules](../PROJECTS.md#the-ai-rules) · [Git workflow](../PROJECTS.md#git--how-you-work)

## P14 — Stateful Streaming + Evaluation · Days 40–42

### 1. Goal
Handle events that arrive **late or out of order**, aggregate them into time windows correctly, and produce a measured accuracy number for the AI classifier you built in P13.

### 2. Why this project exists
Late data is normal — a phone goes through a tunnel and delivers its events 90 minutes later. Deciding what to do with it is **event-time processing**, the hardest idea in streaming and the one that separates people who've run a stream from people who've read about one. And the evaluation half gives you something almost no portfolio has: **evidence that your AI feature actually works.**

### 3. Data source
The same replayed stream, now with lateness injected:
```bash
python -m src.producer --source data/raw/yellow/yellow_tripdata_2024-01.parquet \
  --topic rides --rate 500 --late-pct 15 --max-delay-min 90 --seed 42
```
The producer writes a **sidecar manifest** of exactly which events it delayed and by how much — so "which late events should my watermark have dropped?" has a ground-truth answer you can diff against instead of a guess.

For the evaluation, hand-label a sample from P13's feedback file:
```bash
python -m src.sample_for_labelling --n 200 --seed 42 --out data/to_label.jsonl
# label by hand → data/labelled_sample.jsonl
```
**Label them before you look at the model's output.** Yes, 200 rows by hand is tedious. It is also the only reason anyone will believe your accuracy number.

### 4. What you build

**Step 1 — Event time vs processing time.** Aggregate trips per 10-minute window, first by processing time, then by event time. Compare. With late data they diverge badly. **Processing time is easy and wrong; event time is harder and correct.**

**Step 2 — Add a watermark.**
```python
df.withWatermark("pickup_time", "30 minutes") \
  .groupBy(window("pickup_time", "10 minutes"), "zone").count()
```
The watermark says: *"I'll wait 30 minutes for stragglers, then close the window."* **It's the trade-off between completeness and latency, and it is a business decision, not an engineering one.**

**Step 3 — Measure what you dropped.** Diff your output against the producer's manifest. How many late events made it? How many were dropped? **Then tune the watermark to 10 minutes and to 2 hours and re-measure.** You should be able to predict the result before running it.

**Step 4 — Tumbling vs sliding windows.** Build both. Understand that sliding windows multiply state — a 1-hour window sliding every 5 minutes holds 12× the state.

**Step 5 — Watch state grow.** In the Spark UI's Streaming tab, track state rows and memory. Remove the watermark and watch state grow without bound. **Unbounded state is the #1 streaming outage** — the job runs fine for a week, then OOMs. The watermark is what bounds it.

**Step 6 — Stream-static join.** Join the event stream against the static zone dimension. Note how the static side is re-read, and what happens when it changes mid-stream.

**Step 7 — Deduplicate within the watermark.** `dropDuplicates` with a watermark bound — without the bound, the dedup state grows forever, which is Step 5's lesson again in a different costume.

**Step 8 — Output modes.** Try `append`, `update`, `complete`. Learn which your sink supports — **the sink constrains the choice, not your preference.**

### 5. The AI part — The Evaluation Harness (~60 lines)

**This is the most credible AI artefact in the entire course.** Nearly every "AI-powered" portfolio project has zero measurement behind it. Yours will have a number.

```bash
python -m src.eval_llm --gold data/labelled_sample.jsonl
```
Report, from your 200 hand-labelled records:
- **Overall accuracy**, and **per-class precision and recall** — aggregate accuracy hides a class the model fails at entirely
- **Cost per 1,000 records**
- **p95 latency**
- **Cache hit rate**

Then run the same evaluation against a cheaper model and chart the trade-off. If Haiku gets 91% at a fifth the cost of Sonnet's 94%, **that's a real engineering decision with real numbers** — exactly the conversation you'd have at work.

Finally, make it a **regression test**: change your prompt, re-run the eval, confirm you didn't make it worse. **A prompt edit is a deploy, and deploys need tests.** Most teams learn this the hard way.

### 6. What you learn
**Streaming:** **event time vs processing time** · watermarks as a completeness/latency trade-off · tumbling and sliding windows · state stores and state growth · late and out-of-order handling · stream-static joins · bounded deduplication · output modes
**AI engineering:** building an eval set · precision and recall per class · **cost/quality trade-off curves** · prompt changes as deploys requiring regression tests
**Data engineering:** ground truth as infrastructure · measurement over assertion

### 7. Done checklist
- [ ] Event-time and processing-time results compared, divergence explained
- [ ] **You can predict which late events are dropped before running** — and the manifest diff proves you right
- [ ] Watermark tested at three values with measured effects
- [ ] State growth observed, and bounded with a watermark
- [ ] Dedup bounded — no unbounded state anywhere
- [ ] 200 records hand-labelled before seeing model output
- [ ] Accuracy, per-class precision/recall, cost per 1,000, p95 latency all reported
- [ ] Two models compared on a quality/cost curve
- [ ] Eval runs as a regression test on prompt changes

### 8. In production this would be
The watermark set with product input, since "how late is too late" is their call · a late-arriving-data path that corrects already-published aggregates rather than dropping them · state store size monitored and alerted · a batch reconciliation job as the source of truth · **LLM evals run in CI on every prompt change**, with a quality floor blocking the deploy · per-class accuracy tracked over time, because model and data drift are both real.

### 9. Git

```bash
git switch -c p14-stateful-streaming
```

**Commit at these points** — each one is a working step, not a save point:
- `feat: event-time windowed aggregation`
- `feat: watermark with late-data handling`
- `docs: watermark tuning at 10m/30m/2h with measured effects`
- `feat: stream-static join`
- `fix: bound dedup state with watermark`
- `feat: evaluation harness`
- `docs: accuracy, per-class precision/recall, cost curve`

**PR description should answer:** your watermark choice as a business trade-off, and your classifier's measured accuracy.

**Commit `data/labelled_sample.jsonl`.** 200 hand-labelled rows is the asset that makes your accuracy number believable.

```bash
gh pr create --fill && gh pr merge --squash --delete-branch
git switch main && git pull && git tag p14 && git push --tags
```

---

[← P13](p13-streaming-ingest.md) · [Roadmap](../PROJECTS.md) · [P15 →](p15-graph-cypher.md)
