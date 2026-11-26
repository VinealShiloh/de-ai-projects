> Part of **PHASE 2 — THE CAPSTONE** · [All projects](../PROJECTS.md) · Conventions: [AI rules](../PROJECTS.md#the-ai-rules) · [Git workflow](../PROJECTS.md#git--how-you-work)

## P17 — Open Source Intelligence Platform · Days 49–60

### 1. Goal
Build one platform that uses every technology from the previous 16 projects to answer a real question: **which open source projects are at risk, and who holds them up?**

### 2. Why this project exists
Sixteen separate projects prove you can use sixteen tools. **One integrated platform proves you can build a system** — and that's the difference between a portfolio that gets a first interview and one that gets an offer. It also forces the integration problems that single-tech projects hide: schema changes rippling across systems, orchestrating work across four engines, and cost at a scale where sloppiness is expensive.

The question is genuinely useful. Bus factor, maintainer burnout signals, dependency concentration, community health — nobody can simply look these up, and the answer matters to anyone depending on open source.

### 3. Data source
Everything converges on **GH Archive**, which is why it was chosen on Day 1 — it's simultaneously a batch archive, a replayable stream and a graph.

| Layer | Source | Volume |
|---|---|---|
| Batch backfill | GH Archive hourly files, 30 days | ~74 MB/hr → **~53 GB/month** |
| Live stream | GitHub Events API → Kafka | 5,000 req/hour |
| Dependency edges | `package.json` / `requirements.txt` via contents API | rate-limited; cache hard |
| Enrichment | Anthropic API over issue and PR text | batched, cached, budget-capped |

```bash
make seed DAYS=3      # start here — prove the pipeline end to end
make seed DAYS=30     # scale the backfill once it's correct
```

> **Be honest about scale on a laptop.** 53 GB is real. Build and prove the whole platform on **3 days (~5 GB)**, then scale. A pipeline that works on 3 days and is *designed* for 30 is a far stronger story than one that drowned trying to download everything first.

### 4. What you build

```
GH Archive (hourly) ──► Python extractor (P02) ──┐
                                                  ├──► Raw zone
Live events ──► Kafka ──► Structured Streaming ───┘
                           └─ LLM enrichment (batched, cached, budgeted)
                                                  │
                        ┌─────────────────────────▼──────────────────────┐
                        │  Bronze → Silver → Gold  (Delta, Databricks)   │
                        │  PySpark, tuned and benchmarked                │
                        └──────┬──────────────────────┬──────────────────┘
                               │                      │
                  ┌────────────▼─────────┐   ┌────────▼───────────┐
                  │ dbt marts + QA gates │   │ Neo4j graph + GDS  │
                  └────────────┬─────────┘   └────────┬───────────┘
                               └──────────┬───────────┘
                                          ▼
                        Agentic Q&A (text-to-SQL + GraphRAG, validated)
                                   + dashboard + README

           Airflow orchestrates everything. CI/CD deploys everything.
```

**Days 49–50 — Design.** Architecture diagram. ADRs for the five decisions you'd be challenged on: table format, batch vs stream boundary, graph scope, where the LLM sits. **Write these before coding.** A reviewer will ask "why?" about each.

**Days 51–53 — Ingestion.** Batch extractor with watermarks (P02). Kafka producer and streaming consumer with checkpoints (P13). LLM enrichment on the stream, cached and budgeted.

**Days 54–56 — Transform and model.** Medallion on Delta, tuned using P04's method, deployed via Asset Bundles and CI (P06). dbt marts with quality gates that block publication (P09, P10).

**Days 57–58 — Graph.** Spark → Neo4j incrementally (P16). PageRank and Louvain. Scores written back to Gold.

**Day 59 — AI layer.** Validated text-to-SQL and GraphRAG over the platform, with the eval harness from P14 reporting accuracy and cost per query.

**Day 60 — Ship it.** Airflow orchestrating everything with data-aware scheduling. `make up` works from a clean clone. README, benchmarks, dashboard, write-up.

### 5. The AI part — The Whole Stack, Measured

Every level from the course appears, and each carries its number:

| Level | What it does | What you report |
|---|---|---|
| Verifier | Reject explainer, permission auditor, idempotency auditor | issues caught vs missed |
| Reviewer | PR reviewer on your own checklist | precision and recall |
| In-pipeline | Issue/PR classification on the stream | accuracy, cost per 1,000, cache hit rate |
| Over-data | Text-to-SQL + GraphRAG | accuracy on held-out questions, cost per query |

```bash
make ask Q="which critical repos have a bus factor of one?"
make eval        # regression eval across the AI layer
```

**The eval harness is what makes this credible.** When someone asks *"how do you know the AI layer is any good?"*, you answer with a number and a method. Almost nobody can.

### 6. What you learn
**Integration:** making four engines cooperate · schema changes rippling across systems · end-to-end orchestration · one deploy pipeline for heterogeneous components
**Architecture:** defending design decisions in writing · cost at scale · knowing what to cut when the scope exceeds the time
**Communication:** a README that explains a system to a stranger in five minutes — **the most undervalued skill in this entire course**

### 7. Done checklist
- [ ] A stranger clones the repo, runs `make up`, and the platform comes up
- [ ] 5 ADRs covering the decisions you'd be challenged on
- [ ] Batch and streaming both land in the same Gold tables consistently
- [ ] Quality gates block a deliberately injected breach
- [ ] Graph features available as warehouse columns
- [ ] `make ask` answers a question needing both SQL and graph traversal
- [ ] `make eval` reports accuracy and cost for every AI component
- [ ] `BENCHMARKS.md` has real performance numbers
- [ ] **You can whiteboard the architecture from memory and defend every choice**
- [ ] `AI_NOTES.md` consolidates what AI did well and badly across 60 days

### 8. In production this would be
Owned by a team with on-call · SLAs per table, with consumers notified on breach · cost attributed per component and reviewed monthly · a disaster recovery plan tested, not just written · schema changes going through a contract process rather than a Slack message · **the AI layer's accuracy tracked over time**, because model and data drift are both real and both silent.

### 9. Git

```bash
git switch -c p17-capstone
```

**Commit at these points** — each one is a working step, not a save point:
- `docs: architecture diagram and 5 ADRs`
- `feat: batch extractor with watermarks`
- `feat: kafka producer and streaming consumer`
- `feat: LLM enrichment on the stream`
- `feat: medallion on Delta, tuned`
- `feat: dbt marts with quality gates`
- `feat: graph build and GDS feature export`
- `feat: agentic Q&A over SQL and graph`
- `feat: airflow orchestration end to end`
- `ci: one-command deploy`
- `docs: README, BENCHMARKS, consolidated AI_NOTES`

**PR description should answer:** the whole architecture, and the five decisions you'd be challenged on.

**Work on sub-branches off the capstone branch** (`p17-capstone-ingestion`, `p17-capstone-graph`) and PR each into it. Twelve days is too long for one branch.

**At the end:** consider `git subtree split` to extract the capstone into its own repo to pin on your profile, keeping this one as the full learning history.

```bash
gh pr create --fill && gh pr merge --squash --delete-branch
git switch main && git pull && git tag v1.0 && git push --tags
```

---

[← P16](p16-graph-pipeline.md) · [Roadmap](../PROJECTS.md) · —
