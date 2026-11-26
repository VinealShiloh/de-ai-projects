> Part of **08 · NEO4J & GRAPH (Days 43–48)** · [All projects](../PROJECTS.md) · Conventions: [AI rules](../PROJECTS.md#the-ai-rules) · [Git workflow](../PROJECTS.md#git--how-you-work)

## P16 — Spark → Neo4j + GraphRAG · Days 46–48

### 1. Goal
Build the graph from Spark instead of CSVs, run graph algorithms on it, write the results **back into your warehouse as a table** — and answer questions that neither the warehouse nor the graph could answer alone.

### 2. Why this project exists
P15 was a toy load. **This is the production pattern:** the graph isn't a separate system people query directly, it's a specialised compute engine inside a wider pipeline. It calculates something relational stores can't — influence, community, connectivity — and hands the answer back as a column everyone else can use. That round trip is the insight.

### 3. Data source
No new download. The graph is now **built by Spark from your Silver layer** — that change *is* the project.
```bash
spark-submit --packages org.neo4j:neo4j-connector-apache-spark_2.12:5.3.0_for_spark_3 \
  src/load_graph.py --source data/silver/events --mode merge --date 2024-01-15
```
**Run it twice with the same date. Node count must not change.** If it doubles, your MERGE keys are wrong — and that's the bug worth finding now, because a full reload won't be an option at capstone scale.

### 4. What you build

**Step 1 — Spark writes the graph.** Shape nodes and relationships as DataFrames, write via the connector. Separate jobs for nodes and relationships — nodes must exist first.

**Step 2 — Make the write idempotent.** Use MERGE semantics keyed on business keys, not `CREATE`. Verify with the double-run test above. **This is P01's idempotency lesson, in its ninth costume.**

**Step 3 — Incremental updates.** Load one day. Then load the next day and confirm only new nodes and relationships appear. Full reloads don't scale past a few million nodes.

**Step 4 — Your first GDS projection.** Graph Data Science runs on an **in-memory projection**, not the stored graph. Project only the nodes and relationships the algorithm needs:
```cypher
CALL gds.graph.project('contributors', ['User','Repo'], 'CONTRIBUTED_TO');
```
**Projection size is a capacity-planning problem** — project everything and you'll run out of memory.

**Step 5 — PageRank.** Who are the influential contributors? Not just the most prolific — influence flows through connections. **Compare the PageRank top 20 against a simple commit-count top 20.** The difference is the whole argument for graph algorithms.

**Step 6 — Louvain community detection.** Find clusters of contributors who work together. Inspect a few — do they correspond to real ecosystems?

**Step 7 — Write results back to the warehouse.** Export `influence_score` and `community_id` per contributor, land them in Delta as a Gold table. **Now every analyst can use a graph-computed feature without knowing Cypher exists.** This is the payoff.

**Step 8 — Orchestrate it.** Your P11/P12 Airflow DAG now refreshes the graph and lands the scores. (Phase 1's no-borrowing rule ends here — this is the first deliberate integration.)

### 5. The AI part — GraphRAG (~60 lines)

Answer questions that need **both** connection-walking and synthesis:

> *"Which critical repos depend on an unmaintained library and have only one active maintainer?"*

No single query answers this. It needs: traverse dependencies, check maintenance signals, count active maintainers, judge "critical". The AI uses your **validated** text-to-Cypher tool from P15 to walk the graph, then synthesises across the hops.

**Then run the comparison that makes this a portfolio piece:** build vector RAG over the same repos' README text and ask both systems the same 20 questions. Report where each wins.

The expected result, which you should verify rather than assume: **vectors find *similar* things; graphs find *connected* things.** Ask "what repos are like this one?" and vectors win. Ask "how is this repo connected to that security advisory?" and vectors fail completely, because similarity isn't connection.

**That comparison, with evidence, is genuinely differentiated** — GraphRAG is current frontier work for enterprise retrieval precisely because real business questions are about relationships.

### 6. What you learn
**Neo4j:** the Spark connector · idempotent graph writes at scale · incremental graph updates · **GDS projections** · PageRank, Louvain, centrality · writing algorithm output back to a warehouse
**Data engineering:** the graph as a feature generator inside a pipeline · specialised compute in a wider architecture · integration across systems
**AI:** GraphRAG · multi-hop retrieval · comparing retrieval strategies with evidence

### 7. Done checklist
- [ ] Graph built by Spark, not a CSV script
- [ ] **Running the same load twice doesn't change node count**
- [ ] Incremental load adds only new data
- [ ] PageRank top 20 compared against commit-count top 20, difference explained
- [ ] Communities inspected and sanity-checked against reality
- [ ] Graph features land in Gold as a queryable table
- [ ] Airflow refreshes the whole chain
- [ ] GraphRAG vs vector RAG compared on 20 questions, with results
- [ ] `AI_NOTES.md` records where each retrieval method failed

### 8. In production this would be
The graph rebuilt incrementally on a schedule, full reload only for disaster recovery · GDS projections sized against available memory, with projection as an explicit capacity decision · algorithm outputs versioned so you can explain why someone's score changed · GraphRAG behind read-only credentials with query timeouts · retrieval quality measured on a fixed question set, the same discipline as P14's eval.

### 9. Git

```bash
git switch -c p16-graph-pipeline
```

**Commit at these points** — each one is a working step, not a save point:
- `feat: spark writes graph via connector`
- `fix: MERGE keys make reloads idempotent`
- `feat: incremental graph updates`
- `feat: GDS projection and PageRank`
- `feat: Louvain communities`
- `feat: graph features written back to Gold`
- `feat: GraphRAG over the validated cypher tool`
- `docs: GraphRAG vs vector RAG on 20 questions`

**PR description should answer:** why the graph is a compute engine, not a system of record.

```bash
gh pr create --fill && gh pr merge --squash --delete-branch
git switch main && git pull && git tag p16 && git push --tags
```

---

[← P15](p15-graph-cypher.md) · [Roadmap](../PROJECTS.md) · [P17 →](p17-capstone.md)
