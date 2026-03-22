# Part 3: Database Table Setup & Data Ingestion

## Overview

In this part you will:
1. Create the `RESEARCH_PAPERS` table with a `VECTOR` column
2. Create vector (HNSW) and full-text (Oracle Text) indexes
3. Create relational tables for graph retrieval (authors, similarities)
4. Ingest the 1,000 papers into Oracle
5. Build and register a SQL Property Graph

## 3.1 Create the Research Papers Table

The main table has:
- `arxiv_id` — primary key
- `title`, `abstract` — metadata
- `text` — full document text (CLOB)
- `embedding` — vector column with dimension matching your model (768 for nomic-embed)

The DDL safely drops dependent tables first (graph edge tables), then recreates the core table.

## 3.2 Create Indexes

**HNSW Vector Index** — Enables fast approximate nearest-neighbor search:
```sql
CREATE VECTOR INDEX RP_VEC_HNSW
ON research_papers(embedding)
ORGANIZATION INMEMORY NEIGHBOR GRAPH
DISTANCE COSINE
WITH TARGET ACCURACY 90
PARAMETERS (TYPE HNSW, NEIGHBORS 40, EFCONSTRUCTION 500)
```

**Oracle Text Index** — Enables full-text keyword search:
```sql
CREATE INDEX rp_text_idx
ON research_papers(text)
INDEXTYPE IS CTXSYS.CONTEXT
PARAMETERS ('SYNC (ON COMMIT)')
```

> **Knowledge Checkpoint:** HNSW (Hierarchical Navigable Small World) is a graph-based vector index that trades a small amount of accuracy for orders-of-magnitude speedup on large datasets.

## TODO: Implement `create_research_papers_table`

Write a function that:
1. Drops dependent tables safely (paper_similarities, paper_authors, authors, research_papers)
2. Creates the `research_papers` table with the correct VECTOR dimension
3. Commits the transaction

**Hint:** Use `BEGIN ... EXCEPTION WHEN OTHERS THEN IF SQLCODE != -942 THEN RAISE; END IF; END;` for safe drops.

## 3.3 Data Ingestion

Embeddings are converted to `array.array('f', ...)` for proper Oracle VECTOR binding, then inserted row by row with progress tracking.

## 3.4 Graph Tables & Property Graph

For graph-based retrieval, we create:
- `AUTHORS` — normalized author names
- `PAPER_AUTHORS` — author-paper edges (WROTE relationship)
- `PAPER_SIMILARITIES` — top-10 similar papers per paper (SIMILAR_TO relationship)

These are registered as a SQL Property Graph (`RESEARCH_GRAPH`) for use with `GRAPH_TABLE()` queries.

## Troubleshooting

**"ORA-51956: vector memory size"** — The vector memory pool is too small for HNSW. The setup script should have set it to 512M. If not, run: `ALTER SYSTEM SET vector_memory_size=512M SCOPE=SPFILE;` as SYSDBA and restart the container.
