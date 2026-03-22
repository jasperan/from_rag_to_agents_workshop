# Part 4: Retrieval Mechanisms

## Overview

This is the core retrieval part. You will implement and compare five retrieval strategies, all running inside Oracle:

| Strategy | Technique | Best For |
|---|---|---|
| Keyword | Oracle Text `CONTAINS()` | Exact term matching |
| Vector | `VECTOR_DISTANCE()` with HNSW | Semantic similarity |
| Hybrid Pre-Filter | Text filter first, then vector rank | Known-keyword + semantic |
| Hybrid Post-Filter | Vector candidates first, then text filter | Broad semantic + keyword refinement |
| Hybrid RRF | Reciprocal Rank Fusion of both lists | Balanced fusion of both signals |
| Graph | SQL Property Graph + vector seed | Multi-hop relationship discovery |

## 4.1 Keyword Search (Pre-built)

Uses the Oracle Text index to find documents where `CONTAINS(text, :keyword)` matches. Results are ranked by `SCORE(1)` — Oracle Text's relevance score.

## TODO: Implement `vector_search_research_papers`

Write a function that:
1. Encodes the query using the embedding model with `search_query:` prefix
2. Converts the embedding to `array.array('f', ...)` for Oracle binding
3. Runs a SQL query using `VECTOR_DISTANCE(embedding, :q, COSINE)`
4. Returns `(rows, columns)` tuple

**Key SQL pattern:**
```sql
SELECT arxiv_id, title, abstract,
       ROUND(1 - VECTOR_DISTANCE(embedding, :q, COSINE), 4) AS similarity_score
FROM research_papers
ORDER BY similarity_score DESC
FETCH APPROX FIRST :top_k ROWS ONLY WITH TARGET ACCURACY 90
```

**Complete solution:**
```python
def vector_search_research_papers(conn, embedding_model, search_query, top_k=5):
    query_embedding = embedding_model.encode(
        [f"search_query: {search_query}"],
        convert_to_numpy=True, normalize_embeddings=True
    )[0].astype(np.float32).tolist()
    query_embedding_array = array.array('f', query_embedding)

    query = f"""
        SELECT arxiv_id, title, abstract,
               SUBSTR(text, 1, 200) AS text_snippet,
               ROUND(1 - VECTOR_DISTANCE(embedding, :q, COSINE), 4) AS similarity_score
        FROM research_papers
        ORDER BY similarity_score DESC
        FETCH APPROX FIRST {top_k} ROWS ONLY WITH TARGET ACCURACY 90
    """
    with conn.cursor() as cur:
        cur.execute(query, q=query_embedding_array)
        rows = cur.fetchall()
        columns = [desc[0] for desc in cur.description]
    return rows, columns
```

## TODO: Implement `hybrid_search_research_papers_pre_filter`

Combine keyword filtering with vector ranking:
1. Encode the query (same as vector search)
2. Use `WHERE CONTAINS(text, :kw, 1) > 0` to pre-filter
3. Rank filtered results by `VECTOR_DISTANCE`

This strategy first narrows the candidate set with keywords, then re-ranks by semantic similarity.

**Complete solution:**
```python
def hybrid_search_research_papers_pre_filter(conn, embedding_model, search_phrase, top_k=10, show_explain=False):
    query_embedding = embedding_model.encode(
        [f"search_query: {search_phrase}"],
        convert_to_numpy=True, normalize_embeddings=True
    )[0].astype(np.float32).tolist()
    query_embedding_array = array.array('f', query_embedding)

    with conn.cursor() as cur:
        sql = f"""
            SELECT arxiv_id, title, abstract,
                   SUBSTR(text, 1, 200) AS text_snippet,
                   ROUND(1 - VECTOR_DISTANCE(embedding, :q, COSINE), 4) AS similarity_score
            FROM research_papers
            WHERE CONTAINS(text, :kw, 1) > 0
            ORDER BY similarity_score DESC
            FETCH APPROX FIRST {top_k} ROWS ONLY WITH TARGET ACCURACY 90
        """
        cur.execute(sql, q=query_embedding_array, kw=search_phrase)
        rows = cur.fetchall()
        columns = [desc[0] for desc in cur.description]
    return rows, columns, None
```

## 4.3 Hybrid Post-Filter, RRF, and Graph (Pre-built)

These are provided as complete implementations:
- **Post-filter**: retrieves vector candidates first, then applies text filter
- **RRF**: runs both lists independently and fuses scores with `1/(k + rank)`
- **Graph**: seeds from vector similarity, then expands via SIMILAR_TO and shared-author paths using `GRAPH_TABLE()`

## 4.5 Compare Retrieval Strategies

The final cell runs all strategies on the same query and displays results side-by-side so you can compare ranking behavior.

## Troubleshooting

**"ORA-29902: CONTAINS error"** — The Oracle Text index may not be synced. Run: `EXEC CTX_DDL.SYNC_INDEX('rp_text_idx')`

**Empty vector results** — Verify data was ingested: `SELECT COUNT(*) FROM research_papers`
