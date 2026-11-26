# Data Engineering Portfolio — 17 Projects, 60 Days

**What this is:** a project-by-project plan to take you from intermediate SQL + basic Python to a hireable data engineer, by building 16 single-technology projects and then merging them into one capstone.

**Your starting point:** SQL intermediate–advanced. Python basic–intermediate. No Spark.

**Every project has the same eight sections.** Goal, Why it exists, Data source, What you build, The AI part, What you learn, Done checklist, Production note. Follow the numbered steps in order.

---

## The plan at a glance

| # | Project | Tech | Days |
|---|---|---|---|
| [P01](projects/p01-file-ingestor.md) | File Ingestor | Python | 1–3 |
| [P02](projects/p02-api-extractor.md) | API Extractor | Python | 4–6 |
| [P03](projects/p03-batch-pipeline.md) | Batch Pipeline | PySpark | 7–9 |
| [P04](projects/p04-skew-clinic.md) | Performance & Skew Clinic | PySpark | 10–12 |
| [P05](projects/p05-lakehouse-medallion.md) | Lakehouse Medallion | Databricks | 13–15 |
| [P06](projects/p06-bundles-cicd.md) | Asset Bundles + CI/CD | Databricks | 16–18 |
| [P07](projects/p07-delta-deep-dive.md) | Delta Lake Deep Dive | Delta | 19–21 |
| [P08](projects/p08-iceberg-comparison.md) | Iceberg & Format Comparison | Iceberg | 22–24 |
| [P09](projects/p09-dbt-analytics.md) | Analytics Engineering | dbt | 25–27 |
| [P10](projects/p10-quality-gates.md) | Data Quality Gates | dbt | 28–30 |
| [P11](projects/p11-airflow-orchestration.md) | Orchestrating the Pipeline | Airflow | 31–33 |
| [P12](projects/p12-dynamic-dags.md) | Dynamic & Data-Aware DAGs | Airflow | 34–36 |
| [P13](projects/p13-streaming-ingest.md) | Real-Time Ingestion | Kafka + Spark | 37–39 |
| [P14](projects/p14-stateful-streaming.md) | Stateful Streaming | Kafka + Spark | 40–42 |
| [P15](projects/p15-graph-cypher.md) | Graph Modelling & Cypher | Neo4j | 43–45 |
| [P16](projects/p16-graph-pipeline.md) | Spark → Neo4j + GraphRAG | Neo4j | 46–48 |
| **[P17](projects/p17-capstone.md)** | **Open Source Intelligence Platform** | **Everything** | **49–60** |

**Phase 1 rule (P01–P16):** one technology per project. No borrowing. If a PySpark project needs scheduling, you run it by hand — Airflow comes later. You learn each tool's real shape before the next tool hides it.

---

## The AI rules

Every project contains a small AI component **that you write yourself**. It is never more than ~60 lines. These five rules apply to all of them.

**1. It advises, it never decides.**
The AI prints text for you to read. It cannot modify data, change a config, rewrite a contract, or trigger a run. Every project works correctly if you never run the AI step.

**2. One API call per run. Never one per row.**
A per-row call on 500,000 rows is hours of waiting and real money. Batch your evidence into a single call. This single habit separates people who have shipped an LLM pipeline from people who have demoed one.

**3. The pipeline must work without it.**
Unplug the AI and everything still runs. The API being down is not an outage. The AI is a *side-car*, not a dependency — until P13, where it deliberately enters the data path and you learn what that costs.

**4. You verify every answer.**
Each project has an `AI_NOTES.md` recording: what you asked, one thing the model got right, one thing it got wrong, how you caught it. **This file is the most valuable thing in your portfolio.** Anyone can generate code. Almost nobody can show evidence they know when it's wrong.

**5. The honesty test: could deterministic code do this?**
If yes, write the deterministic code. A "AI type checker" is theatre — type checking is `isinstance()`. The AI earns its place only on fuzzy judgement, free text, or ranking hypotheses. Three projects below have a thin AI step, and **I say so in the project** rather than pretending otherwise.

### You write the pipeline code yourself until Day 18

This is not negotiable, and it's the opposite of how most people use AI.

For the first 18 days you are learning to *read* Spark, SQL and Python. If a model writes them, you can't spot a skewed join, can't read an execution plan, and can't answer an interview question. You'd finish with a repo you cannot defend.

From Day 19 (Delta onward), once you can tell good code from bad, you may let Claude draft and you review. That's the earned order — judgement first, speed second.

### How the AI component grows

| Projects | Level | What the AI does |
|---|---|---|
| P01–P06 | **Verifier** | Explains rejects, reviews your config, checks your output looks sane |
| P07–P12 | **Reviewer** | Reads query profiles and execution plans, ranks hypotheses, generates adversarial tests |
| P13–P14 | **In the pipeline** | Classifies records on the data path — batched, cached, budgeted, and *measured* |
| P15–P17 | **Over the data** | Natural language → query, with validation; graph-backed retrieval |

---

## Where the data comes from

Six sources carry all 19 projects. All free, none needs a credit card except the two cloud trials. You learn a dataset once and reuse the familiarity while the *technology* changes.

| Source | Used by | Format & size |
|---|---|---|
| **NYC TLC Trip Records** | P03–P06, P11, P13, P14 | Parquet · ~50 MB/month, ~40M rows/year |
| **GH Archive** | P01, P15, P16, P17 | nested JSONL.gz · ~74 MB/hour |
| **GitHub REST API** | P02, P12, P17 | JSON · 5,000 req/hour with a token |
| **NYC 311 Requests** | P01, P12 | messy CSV · filterable slices |
| **TPC-H** (generated by DuckDB) | P09, P10 | standard benchmark schema, any size |
| **Faker / your own producer** | P07, P08, P13, P14 | controlled, seeded, reproducible |

**Why generated data for some projects:** you need *controlled* change — a known 10% of rows updated, a known 2% arriving late, a known correct answer. No public dataset gives you that, and without it you can only prove your code ran, not that it's right.

**Two rules from Day 1:**
- `data/` is gitignored, always. Raw data never enters the repo. A small `tests/fixtures/` sample does.
- Always `--seed 42` on generated data. Irreproducible test data is untestable test data.

---

## Setup

**Day 1, once:**
```bash
cd /Users/vneal/Projects/DE_AI_Projects
git init
python3.11 -m venv .venv && source .venv/bin/activate
pip install -U pip
pip install anthropic            # the AI side-car, used from P01
export ANTHROPIC_API_KEY=sk-ant-...
```

**Accounts, and when to create them** — timing matters because trials expire:

| Account | Create it on | Why then |
|---|---|---|
| Anthropic API | Day 1 | Used from P01. A few dollars covers the whole 60 days. |
| Databricks Free Edition | Day 12 | Needed for P05. Free Edition doesn't expire, but has a credit budget. |
| GitHub PAT | Day 4 | P02. Classic token, `public_repo` scope. |

**Repository layout** — every project is self-contained:
```
DE_AI_Projects/
├── PROJECTS.md
├── 01-python/p01-file-ingestor/
│   ├── README.md          <- what it does, how to run it
│   ├── AI_NOTES.md        <- what the model got right and wrong
│   ├── CLAUDE.md          <- project context for the coding agent
│   ├── prompts/           <- the prompts that worked
│   ├── src/  tests/  contracts/  data/(gitignored)
└── ...
```


---

## Git — how you work

**One repository, one branch per project.** Not 17 separate repos.

Why: the capstone imports code from earlier projects, so they must live together. More importantly, **branch → commit → PR → merge is the workflow you'll use at work**, and a history showing you worked that way is itself portfolio evidence. A recruiter opening one strong repo with 17 projects and 300 honest commits learns more than one opening 17 thin repos.

### One-time setup

```bash
cd /Users/vneal/Projects/DE_AI_Projects
git init -b main
git add .gitignore PROJECTS.md
git commit -m "chore: project roadmap and gitignore"
gh repo create de-ai-projects --public --source=. --remote=origin --push
```
No `gh`? Create the repo on github.com, then:
```bash
git remote add origin git@github.com:<you>/de-ai-projects.git
git push -u origin main
```

### The cycle, every project

```bash
git switch -c p01-file-ingestor        # 1. branch before you write a line
# ... work, committing as you go ...
git push -u origin p01-file-ingestor   # 2. push early, push often
gh pr create --fill                    # 3. open a PR when the done checklist passes
gh pr merge --squash --delete-branch   # 4. merge it yourself
git switch main && git pull
git tag p01 && git push --tags         # 5. tag the finished project
```

**Open the PR even though you're the only reviewer.** It forces you to read your own diff before merging, which catches more than you'd expect. Write a real PR description: what it does, what you learned, what you'd do differently. **That description is the single best interview preparation in this course** — you're rehearsing the explanation you'll give out loud.

### Commit messages

Use conventional prefixes. They're a 10-second habit that makes a history readable:

| Prefix | For |
|---|---|
| `feat:` | new capability |
| `fix:` | a bug you found |
| `test:` | tests only |
| `perf:` | a measured optimisation — **put the number in the message** |
| `docs:` | README, notes, ADRs |
| `chore:` | config, deps, tidying |

```bash
git commit -m "perf: broadcast zone lookup, 94s -> 31s"
git commit -m "fix: SCD2 mishandled records back-dated before valid_from"
```
A history of `perf:` commits with real numbers is genuinely rare and immediately credible.

### Three rules

1. **Commit when something works, not at the end of the day.** Each commit should be one working step. "WIP" commits teach a reviewer nothing about how you think.
2. **Never commit data or secrets.** `data/` is gitignored. Before every push: `git diff --cached --stat` and look. If a `.parquet` or a token appears, stop.
3. **Commit the honest stuff too.** `AI_NOTES.md` recording where the model was wrong, `BENCHMARKS.md` with the slow version included. **A repo that only shows success looks fabricated.** Showing the failures and what you did about them is what makes it believable.

### What "done" looks like per project

Branch merged, tagged, and `main` containing: working code, tests, `README.md`, `AI_NOTES.md`, and any `BENCHMARKS.md` or ADRs that project produced.

---

## The projects

Each project is its own file in [`projects/`](projects/). Work them in order.


**01 · PYTHON (Days 1–6)**

- [P01 — File Ingestor](projects/p01-file-ingestor.md) · Days 1–3
- [P02 — API Extractor](projects/p02-api-extractor.md) · Days 4–6

**02 · PYSPARK (Days 7–12)**

- [P03 — Batch Pipeline](projects/p03-batch-pipeline.md) · Days 7–9
- [P04 — Performance & Skew Clinic](projects/p04-skew-clinic.md) · Days 10–12

**03 · DATABRICKS (Days 13–18)**

- [P05 — Lakehouse Medallion](projects/p05-lakehouse-medallion.md) · Days 13–15
- [P06 — Asset Bundles + CI/CD](projects/p06-bundles-cicd.md) · Days 16–18

**04 · DELTA LAKE & ICEBERG (Days 19–24)**

- [P07 — Delta Lake Deep Dive](projects/p07-delta-deep-dive.md) · Days 19–21
- [P08 — Iceberg & Format Comparison](projects/p08-iceberg-comparison.md) · Days 22–24

**05 · DBT (Days 25–30)**

- [P09 — Analytics Engineering](projects/p09-dbt-analytics.md) · Days 25–27
- [P10 — Data Quality Gates](projects/p10-quality-gates.md) · Days 28–30

**06 · AIRFLOW (Days 31–36)**

- [P11 — Orchestrating the Pipeline](projects/p11-airflow-orchestration.md) · Days 31–33
- [P12 — Dynamic & Data-Aware DAGs](projects/p12-dynamic-dags.md) · Days 34–36

**07 · STREAMING — KAFKA + SPARK (Days 37–42)**

- [P13 — Real-Time Ingestion](projects/p13-streaming-ingest.md) · Days 37–39
- [P14 — Stateful Streaming + Evaluation](projects/p14-stateful-streaming.md) · Days 40–42

**08 · NEO4J & GRAPH (Days 43–48)**

- [P15 — Graph Modelling & Cypher](projects/p15-graph-cypher.md) · Days 43–45
- [P16 — Spark → Neo4j + GraphRAG](projects/p16-graph-pipeline.md) · Days 46–48

**PHASE 2 — THE CAPSTONE**

- [P17 — Open Source Intelligence Platform](projects/p17-capstone.md) · Days 49–60

---

# How we work

1. **I explain the concept.** Short. Then we build.
2. **You write the code.** I review it like a PR — direct, specific, no participation trophies.
3. **Each project ends with a retro:** what broke, what you'd do differently, what production would add.
4. **Each project ends with a commit, a README and an `AI_NOTES.md`.** The portfolio builds itself.

**Non-negotiables:**
- You type the code. A project I wrote teaches you nothing.
- Every project gets a test and a README.
- **Unverified AI output never gets committed.**
- `AI_NOTES.md` is written honestly — including the times the model was right and you weren't.

**If you fall behind:** don't skip projects, stretch days. Projects P04, P07, P11 and P14 are the ones that teach the hardest ideas (skew, SCD2, backfill, event time). Those four are worth two extra days each if you need them. The ones you could compress if you must are P06 and P08.
