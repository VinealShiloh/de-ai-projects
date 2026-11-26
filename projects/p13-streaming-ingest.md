> Part of **07 · STREAMING — KAFKA + SPARK (Days 37–42)** · [All projects](../PROJECTS.md) · Conventions: [AI rules](../PROJECTS.md#the-ai-rules) · [Git workflow](../PROJECTS.md#git--how-you-work)

## P13 — Real-Time Ingestion · Days 37–39

### 1. Goal
Build a pipeline that processes events as they arrive, never stops, and — when you kill it halfway — restarts with **every record processed exactly once**. Neither lost nor duplicated.

### 2. Why this project exists
Batch jobs have a start and an end. Streams don't, which breaks every assumption you've built so far: you can't "re-run yesterday", you can't see all the data before deciding, and a crash mid-processing is normal rather than exceptional. **Exactly-once semantics** is how the industry answers that, and understanding checkpointing is what makes it real rather than a slogan.

### 3. Data source
**The TLC trips replayed as a stream**, so you already know the data and only the paradigm is new.
```bash
docker compose up -d kafka
python -m src.producer --source data/raw/yellow/yellow_tripdata_2024-01.parquet \
  --topic rides --rate 500 --key-by PULocationID
```
Plus generated free text for the AI step — **with a hidden ground-truth label**:
```bash
python -m src.gen_feedback --rows 50000 --seed 42 --out data/feedback.jsonl
```
Each row carries its true category in a `_label` field **the pipeline never reads**. 

> **Do not skip this.** That hidden column is what makes P14's evaluation possible. Set it up now or you'll have nothing to measure against in three days.

### 4. What you build

**Step 1 — Kafka running, topic created.** Create `rides` with 6 partitions. **Partition count caps your parallelism and is painful to change later**, so think about it now rather than accepting the default.

**Step 2 — A producer with meaningful keys.** Key by `PULocationID`. Kafka guarantees ordering **within a partition only**, so the key decides what stays ordered. Produce with no key and watch events for the same zone scatter across partitions, out of order. **That experiment is worth the five minutes.**

**Step 3 — Your first streaming read.** `readStream` from Kafka, parse the JSON value, write to console. Watch it run forever. Notice there is no "end".

**Step 4 — Write to Delta with a checkpoint.**
```python
(df.writeStream
   .format("delta")
   .option("checkpointLocation", "./ckpt/bronze_rides")
   .trigger(processingTime="30 seconds")
   .start("data/bronze_rides"))
```
**The checkpoint is the whole game** — it records which Kafka offsets have been committed. Open it and look.

**Step 5 — Prove exactly-once.** Produce 100,000 events with sequential IDs. Kill the consumer mid-stream with `kill -9`. Restart it. Then check: every ID present, each exactly once.
```bash
python -m src.verify_exactly_once --expect 100000
```
**If this fails, nothing else in this project matters.**

**Step 6 — Compare trigger modes.** `processingTime="30 seconds"` versus `availableNow=True`. The second processes everything available and stops — a batch job wearing streaming clothes, and often the right production answer. Compare cost and latency.

**Step 7 — Watch consumer lag.**
```bash
kafka-consumer-groups --bootstrap-server localhost:9092 --describe --group rides
```
Produce faster than you consume and watch lag grow. **Lag is *the* streaming metric** — it's what you'd alert on in production.

### 5. The AI part — LLM on the Data Path (~60 lines)

**This breaks Rule 3 deliberately, and that's the lesson.** Until now the AI has been a side-car. Now it's *in* the pipeline, classifying ride feedback inside `foreachBatch`. Everything that makes production LLM work hard shows up at once:

1. **Batch.** One call per micro-batch with 100 records, never one per record. Per-record on 50,000 rows is hours and real money.
2. **Cache by content hash.** Identical text must never be billed twice. On a replay, the entire batch should come from cache — which means a restart costs nothing.
3. **Degrade, don't die.** API down or rate-limited? Write the record with `category = NULL` and a flag, and **keep the stream running.** A stream that dies because a third-party API hiccuped is a worse outage than missing enrichment.
4. **Cap the spend.** A per-batch token budget, enforced in code. Without it a traffic spike becomes an unbounded bill.
5. **Log cost.** Tokens and dollars per 1,000 records, in your run metrics. **Put that number in the README.**

**The honest framing:** most "real-time AI" requirements are satisfied by a 5-minute micro-batch at a fraction of the cost. Asking "how real-time, really?" before putting a model on the hot path is the senior move here.

### 6. What you learn
**Streaming:** Kafka topics, partitions, offsets, consumer groups · keys and ordering guarantees · Structured Streaming's micro-batch model · **checkpoints and exactly-once** · trigger modes · backpressure · consumer lag · streaming sinks
**AI engineering:** batching on the data path · **caching by content hash** · graceful degradation of a non-deterministic dependency · budget caps · cost per 1,000 records as a tracked metric

### 7. Done checklist
- [ ] Topic partitioned deliberately, with a reason you can state
- [ ] You've seen ordering break with no key, and fixed it with one
- [ ] `kill -9` then restart leaves every record exactly once — verified
- [ ] Trigger modes compared on latency and cost
- [ ] Consumer lag observed growing and recovering
- [ ] **API failure leaves the stream running**, with records flagged not lost
- [ ] Replay serves from cache — no second bill
- [ ] Cost per 1,000 records measured and in the README

### 8. In production this would be
**Consumer lag alerted on, always** — it's the first symptom of nearly every streaming problem · schemas in a registry (Avro/Protobuf), because a producer adding a field silently breaks consumers · checkpoint directories treated as sacred; delete one and you reprocess or lose data · a batch reconciliation job behind every stream, because streams drift and batch is the source of truth · LLM steps behind a circuit breaker with a documented fallback.

### 9. Git

```bash
git switch -c p13-streaming-ingest
```

**Commit at these points** — each one is a working step, not a save point:
- `chore: kafka compose and topic setup`
- `feat: producer with partition keys`
- `feat: structured streaming consumer to Delta`
- `feat: checkpointing`
- `test: exactly-once verified after kill -9`
- `feat: LLM enrichment, batched and cached`
- `feat: budget cap and graceful degradation`
- `docs: cost per 1000 records`

**PR description should answer:** what breaks without a checkpoint, and what the LLM step costs.

```bash
gh pr create --fill && gh pr merge --squash --delete-branch
git switch main && git pull && git tag p13 && git push --tags
```

---

[← P12](p12-dynamic-dags.md) · [Roadmap](../PROJECTS.md) · [P14 →](p14-stateful-streaming.md)
