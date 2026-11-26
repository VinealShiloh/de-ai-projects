> Part of **08 · NEO4J & GRAPH (Days 43–48)** · [All projects](../PROJECTS.md) · Conventions: [AI rules](../PROJECTS.md#the-ai-rules) · [Git workflow](../PROJECTS.md#git--how-you-work)

## P15 — Graph Modelling & Cypher · Days 43–45

### 1. Goal
Model the GitHub contributor network as a graph, query it with Cypher, and find one question that's trivial in a graph and horrible in SQL — then prove it by writing both.

### 2. Why this project exists
You're strong in SQL, so the risk here is assuming a graph database is a solution looking for a problem. **Often it is.** The point of this project is to find out exactly where that stops being true. The honest answer is narrow — "how are these two things connected, through any number of hops" — but where it applies (fraud rings, identity resolution, lineage, dependency risk), nothing else comes close.

### 3. Data source
**GH Archive** — the same source from P01, now read as a graph. Every event already contains actor, repo and org, so the edges are sitting there.
```bash
mkdir -p data/gharchive
curl -O --output-dir data/gharchive https://data.gharchive.org/2024-01-15-{0..5}.json.gz   # ~450 MB
python -m src.shred --input data/gharchive --out data/github/     # → nodes.csv, rels.csv
docker run -v $PWD/data/github:/import -p7474:7474 -p7687:7687 \
  -e NEO4J_AUTH=neo4j/password neo4j:5
```
Shred to CSV first, then bulk-load — `neo4j-admin database import` is orders of magnitude faster than `LOAD CSV` for the initial build.

> **Watch for super-nodes.** Bot accounts (`dependabot[bot]`, `github-actions[bot]`) have millions of edges and will destroy your traversals. Whether to filter them is a modelling decision you'll have to make and defend.

### 4. What you build

**Step 1 — Design the model on paper first.** Which things are nodes (User, Repo, Organisation)? Which are relationships (CONTRIBUTED_TO, OWNS, FORKED_FROM)? Which properties belong where? **Spend real time here** — a bad graph model is far harder to fix later than a bad table.

**Step 2 — Constraints and indexes before loading.** Unique constraints on business keys, indexes on anything you'll match on. Loading first and indexing later is slow and lets duplicates in.

**Step 3 — Bulk load.** Shred to CSV, import, verify counts match your source.

**Step 4 — Cypher basics.** Match a user, their repos, their collaborators. Get fluent with `MATCH`, `WHERE`, `RETURN`, `WITH`, aggregation. **Your SQL instincts transfer, except for the traversal syntax** — which is the part worth slowing down for.

**Step 5 — Variable-length paths. This is the real lesson.**
```cypher
MATCH path = shortestPath(
  (a:User {login:'torvalds'})-[:CONTRIBUTED_TO*..6]-(b:User {login:'gvanrossum'}))
RETURN path
```
"Connected through up to 6 hops." **In SQL this is six self-joins, or a recursive CTE that gets slower with each hop.** In Cypher it's one line and the engine walks pointers.

**Step 6 — Write the same query in SQL.** Actually write it. Compare line count and runtime. **This comparison is the deliverable of the project** — it's what lets you argue for or against a graph database with evidence instead of enthusiasm.

**Step 7 — A recommendation query.** "Contributors to repos similar to yours" — a 3-hop traversal. Note how naturally it expresses something that'd be awkward in SQL.

**Step 8 — Find your super-nodes and deal with them.** Count relationships per node, sorted descending. Decide: filter bots, or cap traversal depth? **Document the choice and the reasoning.**

### 5. The AI part — Text-to-Cypher with Validation (~60 lines)

Build a natural-language-to-Cypher translator. The naive version — question plus schema in a prompt — works about 60% of the time. **Getting to 90% is the actual engineering, and it has nothing to do with clever prompting:**

1. **Inject the live schema** (`CALL db.schema.visualization()`), not a hand-written description that drifts.
2. **Few-shot with your own verified queries** from Steps 4–7. Five real examples beat any amount of instruction.
3. **Validate before executing** — parse the generated Cypher, confirm every label and relationship type exists in the schema. Reject anything that doesn't.
4. **Never execute unvalidated generated queries.** A generated query reaching the database unchecked is an injection vector, not a feature. Read-only credentials for this path, always.
5. **Measure.** 50 held-out questions, report accuracy.

**The lesson:** text-to-query is a retrieval and validation problem, not a prompting problem. People who try to solve it with better prompts plateau around 60%.

### 6. What you learn
**Neo4j:** property graph modelling · Cypher · traversal vs join cost · constraints and indexes · variable-length and shortest paths · super-nodes and their consequences · bulk import
**Data engineering:** when a graph genuinely beats relational, and when it's fashion · modelling decisions you have to defend
**AI:** text-to-query with schema injection · few-shot from verified examples · **validation before execution** · read-only credentials for generated queries

### 7. Done checklist
- [ ] Model designed on paper before any loading
- [ ] Constraints and indexes created before the import
- [ ] Node and relationship counts reconcile against source
- [ ] A shortest-path query across ≥4 hops returns sensible results
- [ ] **The same query written in SQL, with both line count and runtime compared**
- [ ] Super-nodes identified and a documented decision made
- [ ] Text-to-Cypher validates against the live schema and refuses invalid output
- [ ] Accuracy measured on 50 held-out questions
- [ ] `AI_NOTES.md` records the most common generation failure

### 8. In production this would be
The graph **projected from the warehouse** on a schedule, not the system of record · modelling reviewed before loading, because fixing a model at scale means a full rebuild · super-node handling designed in from the start · use cases kept narrow — fraud rings, identity resolution, lineage, dependency risk — with "could three SQL joins do this?" asked first every time · generated queries executed under a read-only role with a timeout.

### 9. Git

```bash
git switch -c p15-graph-cypher
```

**Commit at these points** — each one is a working step, not a save point:
- `docs: graph model design and rationale`
- `chore: constraints and indexes before load`
- `feat: shred events to nodes and relationships`
- `feat: bulk import`
- `feat: shortest-path and recommendation queries`
- `docs: same query in SQL, line count and runtime compared`
- `feat: text-to-cypher with schema validation`
- `docs: accuracy on 50 held-out questions`

**PR description should answer:** the one query that justifies a graph database, and the super-node decision you made.

```bash
gh pr create --fill && gh pr merge --squash --delete-branch
git switch main && git pull && git tag p15 && git push --tags
```

---

[← P14](p14-stateful-streaming.md) · [Roadmap](../PROJECTS.md) · [P16 →](p16-graph-pipeline.md)
