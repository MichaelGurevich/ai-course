# Lecture 2: ReAct agents with LangGraph

The loop from lecture 1 has a name: **ReAct** (Reason + Act). The model reasons, asks for a tool, observes the result and repeats until it can answer. This lecture rebuilds that loop with **LangGraph**, then opens it up to show what is sent to the LLM on every turn.

The LLM is stateless. On every call the harness sends the system prompt, the tool definitions and the **whole conversation so far**. Each step appends new messages and nothing is removed, so the context grows on every turn and the total tokens billed grow even faster.

For a written explanation of the lecture's ideas, read the [lecture notes](lecture_notes.md).

## Notebook

[`react_agent_langgraph.ipynb`](react_agent_langgraph.ipynb):

**Part 1: Building a ReAct agent with LangGraph**

1. The calculator tools again, now described by `@tool` instead of the system prompt.
2. Built-in tool calling with `bind_tools`: the reply comes back with structured `tool_calls`.
3. The state: a list of messages, and the `add_messages` reducer that appends to it.
4. The two nodes: `agent` (calls the LLM) and `ToolNode` (runs the tools), plus the complete system prompt the API builds from our prompt and the tool schemas.
5. Wiring the graph: the conditional edge `tools_condition` is the loop's exit.
6. Running it, plus a table that maps each piece to lecture 1's hand-written harness.

**Part 2: The context grows on every turn**

7. Streaming the state to watch the message list grow, one step at a time.
8. Logging every LLM call: which messages were sent and how many input tokens they cost.
9. A chart showing that each call re-sends everything the previous call sent, plus a bit more.

**Part 3: Big tool results make it worse**

10. A sales-report tool that returns a long table. Its output is re-sent on every later call.

**Part 4: Why this matters**

11. Linear context, quadratic bill: extrapolating the measured numbers to 50 steps, and the techniques that manage context (smaller tool results, trimming, summarizing, clearing tool results, sub-agents, prompt caching).

**Part 5: Sub-agents as tools**

12. A "sales analyst" agent wrapped in an ordinary `@tool`. Each call starts a fresh conversation, does the heavy reading there and returns only a short answer.
13. Part 3's question again: the main agent's context stays small, because the reports live and die in the sub-agent's context.

**Part 6: The one-liner**

14. The same agent in one line with LangChain's `create_agent`.

## Running it

```bash
pip install langgraph langchain langchain-anthropic matplotlib jupyter
export ANTHROPIC_API_KEY=...
jupyter notebook react_agent_langgraph.ipynb
```

The token counts you see depend on the model's choices in each run, but the shape is always the same: every call contains the previous one.
