# Part 5: Building a RAG Pipeline

## Overview

In this part you connect retrieval to generation end-to-end:
1. Configure OpenAI API access
2. Initialize and smoke-test the client
3. Build a reusable RAG function that supports all retrieval modes
4. Run an end-to-end query

## 5.1 Configure API Access

We use a `set_env_securely()` helper with `getpass` to set `OPENAI_API_KEY` without exposing it in notebook output.

## 5.2 Initialize the OpenAI Client

A simple smoke test confirms credentials work:
```python
from openai import OpenAI
openai_client = OpenAI(api_key=os.environ.get("OPENAI_API_KEY"))
response = openai_client.responses.create(
    model="gpt-4o",
    input="Hello!",
    instructions="You are a research paper assistant.",
)
```

## TODO: Implement `research_paper_assistant_rag_pipeline`

This is the core RAG function. It must:

1. **Select retrieval strategy** based on `retrieval_mode` parameter (keyword, vector, hybrid, graph)
2. **Call the appropriate retrieval function** from Part 4
3. **Format retrieved rows** into citation-ready context with titles, abstracts, snippets, and scores
4. **Construct a prompt** that includes the user query and formatted context
5. **Call the OpenAI Responses API** with grounding instructions
6. **Return** the generated text

**Key design decisions:**
- The function should work with any retrieval mode by routing to the correct function
- Context formatting should include numbered citations `[1]`, `[2]`, etc.
- The LLM prompt should instruct the model to cite sources and avoid speculation

**Complete solution:**
```python
def research_paper_assistant_rag_pipeline(
    conn, embedding_model, user_query, top_k=10,
    retrieval_mode="hybrid", show_explain=False
):
    # 1. Retrieve
    if retrieval_mode == "keyword":
        rows, columns = keyword_search_research_papers(conn, user_query)
    elif retrieval_mode == "vector":
        rows, columns = vector_search_research_papers(conn, embedding_model, user_query, top_k)
    elif retrieval_mode == "graph":
        rows, columns = graph_search_research_papers(conn, embedding_model, user_query, top_k=top_k)
    else:
        rows, columns, _ = hybrid_search_research_papers_pre_filter(
            conn=conn, embedding_model=embedding_model,
            search_phrase=user_query, top_k=top_k, show_explain=show_explain
        )

    # 2. Format context
    retrieved_count = len(rows) if rows else 0
    formatted_context = ""
    if retrieved_count > 0:
        for i, row in enumerate(rows):
            row_data = dict(zip(columns, row))
            title = row_data.get("TITLE", "Untitled")
            abstract = row_data.get("ABSTRACT", "No abstract.")
            score = (row_data.get("GRAPH_SCORE") or row_data.get("SIMILARITY_SCORE")
                     or row_data.get("RELEVANCE_SCORE") or "N/A")
            formatted_context += f"[{i+1}] {title}\nAbstract: {abstract}\nScore: {score}\n\n"

    # 3. Call LLM
    prompt = f"""User Query: {user_query}
    Retrieved papers: {retrieved_count}
    {formatted_context}
    Summarize findings. Use [X] citations. Highlight consensus and gaps."""

    response = openai_client.responses.create(
        model="gpt-4o", input=prompt,
        instructions="You are a scientific research assistant. Use only provided context. Cite papers [1], [2], etc.",
        temperature=0.3,
    )
    return response.output_text
```

## Troubleshooting

**"Invalid API key"** — Re-run the `set_env_securely` cell with a valid OpenAI key.

**Empty retrieval** — Check Part 4 functions work independently before debugging the pipeline.
