# Lecture 2: ReAct agents and the growing context

## What this lecture is about

Lecture 1's harness makes a single round trip: the model asks for tools, the harness runs them and sends back the results, and the model answers. That can't handle a task where one step needs another step's result, like "add 12 and 30, then multiply by 4".

The fix is an **agent loop**: keep calling the model and running tools until it gives a final answer. This loop is called **ReAct** (Reason + Act). In this lecture we build it with **LangGraph**, a library that describes an agent as a graph of steps, and we switch to the API's built-in tool calling.

Then we open the box and look at what is sent to the LLM on every turn. The LLM is stateless: it remembers nothing between calls. So on every call, the harness sends the system prompt, the tool definitions and **the whole conversation so far**. Each step appends new messages and nothing is removed:

```
call 1: [system] [user]
call 2: [system] [user] [ai → add] [tool: 42]
call 3: [system] [user] [ai → add] [tool: 42] [ai → multiply] [tool: 168]
```

The context grows linearly with the number of steps, and the total tokens billed grow quadratically. We finish with ways to keep the context under control, and build one of them: a **sub-agent**, an agent that another agent uses as a tool.

## Learning goals

By the end of the lecture, students should be able to:

- Explain why a single round trip isn't enough, and how the ReAct loop fixes it.
- Build a ReAct agent in LangGraph from a state, two nodes and a conditional edge.
- Explain what built-in tool calling does behind the scenes: the API writes a system prompt that describes the tools.
- Explain why the context grows on every turn, and why the total cost grows faster than the context.
- Measure the input tokens of every LLM call.
- Name techniques for managing context, and build a sub-agent as a tool.

## Key concepts

| Concept | Meaning |
|---|---|
| **ReAct** | The loop an agent runs until it can answer: reason (LLM call), act (tool call), observe (tool result). |
| **State** | The data a LangGraph agent passes from step to step. For a ReAct agent, the list of messages. |
| **Reducer** | The rule that merges a step's output into the state. `add_messages` appends new messages to the list. |
| **Node** | A function in the graph. Here: `agent`, which calls the LLM, and `tools`, which runs the tools. |
| **Conditional edge** | An arrow whose destination depends on the state. After `agent`: tool calls? Go to `tools`. Otherwise, stop. |
| **Built-in tool calling** | Tools are passed as JSON schemas, and the reply contains structured `tool_calls`. The API turns the schemas into a system prompt for the model. |
| **Context** | Everything sent to the model in one call: system prompt, tool definitions and messages. |
| **Input tokens** | The size of the context, as counted and billed by the API. |
| **Stateless** | The model keeps no memory between calls, so the harness must re-send everything each time. |
| **Sub-agent** | An agent wrapped in a tool. It works in its own fresh context and returns only its final answer. |
| **Prompt caching** | Reusing the unchanged beginning of a request, so re-reading it costs less. Works because the context is append-only. |

## Lecture outline

The lecture follows the notebook [`react_agent_langgraph.ipynb`](react_agent_langgraph.ipynb):

1. **The big idea:** from a single round trip to the ReAct loop.
2. **Tools:** the same four calculator functions, described by the `@tool` decorator instead of the system prompt.
3. **Built-in tool calling:** `bind_tools`, and replies with structured `tool_calls`. The model still only *asks*.
4. **The state:** a list of messages, and the `add_messages` reducer that appends to it.
5. **The nodes:** `agent` sends the whole conversation to the LLM, and `ToolNode` runs the tools. We also print the complete system prompt the API builds from our prompt and the tool schemas, using Anthropic's documented template.
6. **The graph:** the conditional edge `tools_condition` is the loop's exit. A table maps each piece to lecture 1's harness.
7. **Running it** on "add 12 and 30, multiply by 4, then divide by 7", the question lecture 1's `ask` couldn't answer.
8. **Watching the context grow:**
   - The message list after every step.
   - A log of every LLM call with its input tokens.
   - A chart that splits each call into tokens re-sent from the previous call and new tokens.
9. **Big tool results:** a sales-report tool whose output is re-sent on every later call.
10. **Why it matters:** call *n* sends F + (n − 1)·k tokens, but after *n* calls the total billed is n·F + k·n(n − 1)/2. Cost, speed, context limits and quality, and the techniques that manage context: smaller tool results, trimming, summarizing, clearing old tool results, sub-agents and prompt caching.
11. **Sub-agents as tools:** a "sales analyst" agent wrapped in a `@tool`. It reads the reports in its own fresh context and returns a one-sentence answer, so the main agent's context stays small. Plus the trade-offs: the sub-agent starts from zero, the main agent sees only the summary, and the work still has to be paid for.
12. **`create_agent`:** the same agent in one line with LangChain's prebuilt.
13. **Wrap-up:** key takeaways and exercises.

## Key takeaways

- ReAct turns lecture 1's single round trip into a loop: reason, act, observe, repeat until the model answers.
- LangGraph makes the loop explicit: a state, nodes and edges. The conditional edge is the loop's exit.
- Built-in tool calling is still a system prompt, written by the provider from your tool definitions.
- The LLM is stateless. The agent's memory is the message list, and the whole list is sent on every call.
- The context grows linearly and the bill grows quadratically. Big tool results make it much worse, because they are re-sent on every later call.
- A sub-agent is a tool that runs its own agent. It keeps heavy work out of the main context and returns only a short answer.
