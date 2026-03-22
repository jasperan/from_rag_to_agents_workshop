# Part 7: Agent Orchestration & Chat System

## Overview

In this part you compose multiple agents into a production-like system:
1. Define specialized agents with focused scope
2. Register agents as tools on an orchestrator
3. Add a synthesizer for final answer generation
4. Build a thread-aware chat system with Oracle-backed history

## Agent-as-Tools Orchestration

This mirrors a production pattern where narrow agents perform focused work and a coordinator merges outputs:

- **research_paper_agent** — retrieves and summarizes academic papers
- **research_conversation_agent** — retrieves past analytical discussions
- **orchestrator_agent** — delegates to specialists based on query intent
- **synthesizer_agent** — produces a cohesive final response

### Orchestration Flow

1. User query arrives
2. Orchestrator decides which specialist(s) to invoke
3. Specialists run their tools and return structured results
4. Synthesizer combines everything into a grounded response

## TODO: Implement the Orchestrator Agent

Define the orchestrator that uses `agent.as_tool()` to register specialists:

```python
orchestrator_agent = Agent(
    name="research_assistant_orchestrator",
    instructions="...",
    tools=[
        research_paper_agent.as_tool(
            tool_name="translate_to_research_papers",
            tool_description="Retrieve and summarize relevant academic research papers.",
        ),
        research_conversation_agent.as_tool(
            tool_name="translate_to_research_conversations",
            tool_description="Retrieve past research discussions related to the topic.",
        ),
    ],
)
```

**Key design decision:** `agent.as_tool()` wraps an entire agent (with its tools and instructions) as a single callable tool for the orchestrator. The orchestrator doesn't need to know implementation details.

## Agentic Chat System

The chat system adds thread-aware conversation handling:

1. **Store** the user's message in Oracle (`chat_history` table)
2. **Reconstruct** conversation context by `thread_id`
3. **Run** orchestration + synthesis with full context
4. **Save** the assistant response back to Oracle

This gives long-running research sessions continuity.

### Chat History Table

```sql
CREATE TABLE chat_history (
    id VARCHAR2(100) PRIMARY KEY,
    thread_id VARCHAR2(100) NOT NULL,
    role VARCHAR2(20) NOT NULL,
    message CLOB NOT NULL,
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP
)
```

## Interactive Chat Session

The `research_chat_session()` function provides a REPL-style interface where you can have multi-turn conversations with the agent system.

## Troubleshooting

**"RuntimeError: This event loop is already running"** — Make sure `nest_asyncio.apply()` is called before running async functions in the notebook.

**Agent not calling specialist tools** — Review the orchestrator instructions. They must explicitly describe when to use each tool.
