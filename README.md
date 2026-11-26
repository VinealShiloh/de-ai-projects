# Data Engineering Portfolio

17 projects building a modern data engineering stack from scratch, one technology at a time, finishing with a platform that uses all of them together.

**Stack:** Python · PySpark · Databricks · Delta Lake · Apache Iceberg · dbt · Airflow · Kafka · Neo4j

Every project includes a small AI component built and measured honestly — see [the AI rules](PROJECTS.md#the-ai-rules).

## Projects

| # | Project | Tech | Status |
|---|---|---|---|
| [P01](projects/p01-file-ingestor.md) | File Ingestor | Python | ⬜ |
| [P02](projects/p02-api-extractor.md) | API Extractor | Python | ⬜ |
| [P03](projects/p03-batch-pipeline.md) | Batch Pipeline | PySpark | ⬜ |
| [P04](projects/p04-skew-clinic.md) | Performance & Skew Clinic | PySpark | ⬜ |
| [P05](projects/p05-lakehouse-medallion.md) | Lakehouse Medallion | Databricks | ⬜ |
| [P06](projects/p06-bundles-cicd.md) | Asset Bundles + CI/CD | Databricks | ⬜ |
| [P07](projects/p07-delta-deep-dive.md) | Delta Lake Deep Dive | Delta | ⬜ |
| [P08](projects/p08-iceberg-comparison.md) | Iceberg & Format Comparison | Iceberg | ⬜ |
| [P09](projects/p09-dbt-analytics.md) | Analytics Engineering | dbt | ⬜ |
| [P10](projects/p10-quality-gates.md) | Data Quality Gates | dbt | ⬜ |
| [P11](projects/p11-airflow-orchestration.md) | Orchestrating the Pipeline | Airflow | ⬜ |
| [P12](projects/p12-dynamic-dags.md) | Dynamic & Data-Aware DAGs | Airflow | ⬜ |
| [P13](projects/p13-streaming-ingest.md) | Real-Time Ingestion | Kafka + Spark | ⬜ |
| [P14](projects/p14-stateful-streaming.md) | Stateful Streaming | Kafka + Spark | ⬜ |
| [P15](projects/p15-graph-cypher.md) | Graph Modelling & Cypher | Neo4j | ⬜ |
| [P16](projects/p16-graph-pipeline.md) | Spark → Neo4j + GraphRAG | Neo4j | ⬜ |
| [P17](projects/p17-capstone.md) | **Open Source Intelligence Platform** | Everything | ⬜ |

Mark each ✅ as it merges.

## How this repo works

One branch per project, merged by PR, tagged on completion — see [the Git workflow](PROJECTS.md#git--how-you-work).

Each project ships with:
- **`README.md`** — what it does and how to run it
- **`AI_NOTES.md`** — what the model got right, what it got wrong, how it was caught
- **tests** — every project has them
- **`BENCHMARKS.md`** where performance was the point

Full roadmap: **[PROJECTS.md](PROJECTS.md)**
