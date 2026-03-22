# Part 6: AI Agents — Basics & Tools

## Overview

In this part you move from a single RAG call to an agentic system. You will:
1. Install the OpenAI Agents SDK
2. Create a baseline agent (no tools)
3. Expose Oracle retrieval as a callable tool
4. Add a second tool for past research conversations
5. Strengthen agent instructions for tool routing

## 6.1 Install Agent Runtime

```python
%pip install -Uq --no-cache-dir openai openai-agents
```

This installs the `agents` package which provides `Agent`, `Runner`, `function_tool`, and orchestration primitives.

## 6.2 Baseline Agent (No Tools)

Start simple — define a research assistant and run a direct query:

```python
from agents import Agent, Runner

research_paper_assistant = Agent(
    name="Research Paper Assistant",
    model="gpt-4o",
    instructions="You are a Research Paper Assistant...",
)
result = await Runner.run(research_paper_assistant, input="Summarize recent research on optimization")
```

Without tools, the agent can only answer from its parametric knowledge.

## TODO: Implement `get_research_papers` Tool

Wrap the SQL retrieval functions from Part 4 in an `@function_tool` decorator so the agent can call them.

**Requirements:**
1. Accept `user_query`, `retrieval_mode` (default "hybrid"), and `top_k` (default 5)
2. Route to the correct retrieval function based on mode
3. Format results as a readable string with numbered citations
4. Return the formatted string

**Complete solution:**
```python
from agents.tool import function_tool

@function_tool
def get_research_papers(user_query: str, retrieval_mode: str = "hybrid", top_k: int = 5) -> str:
    """Retrieves academic research papers relevant to the user's query."""
    if retrieval_mode == "keyword":
        rows, columns = keyword_search_research_papers(conn, user_query)
    elif retrieval_mode == "vector":
        rows, columns = vector_search_research_papers(conn, embedding_model, user_query, top_k)
    elif retrieval_mode == "graph":
        rows, columns = graph_search_research_papers(conn, embedding_model, user_query, top_k=top_k)
    else:
        rows, columns, _ = hybrid_search_research_papers_pre_filter(
            conn=conn, embedding_model=embedding_model,
            search_phrase=user_query, top_k=top_k, show_explain=False
        )

    if not rows:
        return f"No papers found for '{user_query}'."

    formatted = [f"{len(rows)} papers retrieved for: '{user_query}'\n"]
    for i, row in enumerate(rows):
        row_data = dict(zip(columns, row))
        title = row_data.get("TITLE", "Untitled")
        abstract = row_data.get("ABSTRACT", "No abstract.")
        score = (row_data.get("GRAPH_SCORE") or row_data.get("SIMILARITY_SCORE")
                 or row_data.get("RELEVANCE_SCORE") or "N/A")
        formatted.append(f"[{i+1}] {title}\nAbstract: {abstract}\nScore: {score}\n")
    return "\n".join(formatted)
```

## 6.4 Second Tool: Past Research Conversations

The `get_past_research_conversations` tool follows the same pattern but searches for prior analyses rather than raw papers. This gives the agent continuity across sessions.

## 6.5 Strengthening Agent Instructions

Tool availability alone is not enough. Clear instructions make routing explicit:
- When to call paper retrieval
- When to call conversation retrieval
- When to combine both

This is a key lesson: **agent instructions are the control plane for tool selection.**

## Troubleshooting

**"ImportError: cannot import name 'Agent'"** — Restart the kernel after installing `openai-agents`.

**Agent doesn't call tools** — Check that tools are attached: `agent.tools.append(get_research_papers)`.
