# Lecture 2 notes: ReAct agents and the growing context

These notes explain what lecture 2 is about and why it matters. They can be read on their own. The notebook, [`react_agent_langgraph.ipynb`](react_agent_langgraph.ipynb), runs every idea here with real code and real token counts.

## What this lecture is about

In lecture 1 we built an agent by hand. We wrote a loop that calls the LLM, checks whether it asked for a tool, runs the tool and sends the result back.

This lecture does three things:

1. **Gives that loop its name, ReAct, and rebuilds it with LangGraph**, a library that describes an agent as a graph of steps.
2. **Opens the box to show what is sent to the LLM on every turn.** The LLM has no memory, so the harness re-sends the whole conversation on every call. The conversation only grows, so every call costs more than the one before.
3. **Shows how to keep the context under control**, with a closer look at one technique: **sub-agents**, agents that are used as tools by another agent.

## Learning goals

By the end of the lecture you should be able to:

- Explain the ReAct pattern and map it onto lecture 1's harness.
- Build a ReAct agent in LangGraph from a state, two nodes and a conditional edge.
- Explain what built-in tool calling does behind the scenes: the API writes a system prompt that describes your tools.
- Explain why the context grows on every turn, and why the total cost grows faster than the context.
- Name techniques for managing context, and build a sub-agent as a tool.

## 1. From a hand-written loop to ReAct

**ReAct** stands for **Rea**son + **Act** (Yao et al., 2022). The model repeats three steps until it can answer:

1. **Reason**: decide what to do next.
2. **Act**: ask for a tool call.
3. **Observe**: read the tool's result.

That is exactly the loop from lecture 1. What changes in this lecture is who does each part:

| Lecture 1 (by hand) | Lecture 2 (LangGraph) |
|---|---|
| Tool descriptions written in the system prompt | `@tool` + `bind_tools` |
| `parse_tool_call` reads JSON out of the text | Built-in tool calling: `response.tool_calls` |
| `execute_tool` | `ToolNode` |
| `if call is None: return reply` | `tools_condition` |
| `messages.append(...)` | The `add_messages` reducer |
| `for step in range(max_steps)` | The graph's cycle, with `recursion_limit` as the safety net |

## 2. LangGraph in four ideas

- **State**: the data passed from step to step. For a ReAct agent it is just the conversation, a list of messages (`MessagesState`).
- **Reducer**: the rule for updating the state. A step returns only its *new* messages, and the `add_messages` reducer **appends** them. Nothing is removed.
- **Nodes**: plain Python functions that read the state and return an update. A ReAct agent has two: `agent` calls the LLM, and `tools` runs the requested tools.
- **Edges**: the arrows between nodes. A **conditional edge** after `agent` checks the reply. If it contains tool calls the graph goes to `tools`, and otherwise it stops.

```
          START
            │
            ▼
       ┌─────────┐  tool calls?  ┌─────────┐
       │  agent  │ ───── yes ──► │  tools  │
       │  (LLM)  │ ◄──────────── │         │
       └─────────┘               └─────────┘
            │ no
            ▼
           END
```

The cycle `agent` → `tools` → `agent` is the ReAct loop.

## 3. What built-in tool calling really is

With built-in tool calling we no longer describe the tools in our system prompt. We pass them in the request's `tools` field, and the reply comes back with structured `tool_calls`.

The model still reads plain text, though. The API **combines** the tool definitions and our system prompt into one system prompt. Anthropic documents its template:

```text
In this environment you have access to a set of tools you can use to answer the user's question.
{{ FORMATTING INSTRUCTIONS }}
String and scalar parameters should be specified as is, while lists and objects should use JSON format. Note that spaces for string values are not stripped. The output is not expected to be valid XML and is parsed with regular expressions.
Here are the functions available in JSONSchema format:
{{ TOOL DEFINITIONS IN JSON SCHEMA }}
{{ USER SYSTEM PROMPT }}
{{ TOOL CONFIGURATION }}
```

So built-in tool calling is lecture 1's trick, done by the provider, in a format the model was trained on. It also explains why even a one-line question uses many input tokens: the whole tool description is part of every request. The notebook fills in this template with the real tool schemas. The formatting instructions and tool configuration are not published, so they stay as placeholders.

## 4. The LLM is stateless, so the context grows

The model remembers nothing between calls. The only way it can know what happened is if the harness sends it again. So the `agent` node sends the system prompt, the tool definitions **and every message so far** on every call.

Here is what each LLM call sends for "Add 12 and 30, multiply the result by 4, then divide by 7":

```
call 1: [system] [user]
call 2: [system] [user] [ai → add] [tool: 42]
call 3: [system] [user] [ai → add] [tool: 42] [ai → multiply] [tool: 168]
call 4: [system] [user] [ai → add] [tool: 42] [ai → multiply] [tool: 168] [ai → divide] [tool: 24]
```

Each call contains the whole previous call, plus the model's last tool request and the tool's result. The question was sent four times, and so were the system prompt and the tool definitions. The notebook logs the real input token count of every call and draws it as a stacked bar chart, split into "already sent in the previous call" and "new in this call".

## 5. Big tool results make it worse

A calculator result is tiny. Real tools often return much more: a web page, a file, the rows of a database query. The notebook adds a tool that returns a whole year's sales report as a text table.

The model needs only six numbers from two reports. But once a report is in the conversation it is **re-sent on every later call**. The tool ran once, and its output is paid for again and again.

## 6. Linear context, quadratic bill

Say every call has a fixed part of **F** tokens (system prompt, tool definitions, question), and every step adds about **k** tokens (a tool request and its result).

- Call *n* sends **F + (n − 1)·k** tokens. The context grows **linearly**.
- After *n* calls the total billed is **n·F + k·n(n − 1)/2** input tokens. The bill grows **quadratically**.

For example, with F = 1,500 and k = 150 (illustrative round numbers):

| Calls | Size of the last call | Total input tokens billed |
|---|---|---|
| 10 | 2,850 | 21,750 |
| 50 | 8,850 | 258,750 |

The context grows about 3 times, but the bill grows about 12 times. As tasks get longer the k·n² part dominates, and doubling the number of steps comes close to quadrupling the bill. The notebook uses the F and k it measured from a real run to draw this curve.

A growing context hurts in four ways:

- **Cost**: every input token of every call is billed.
- **Speed**: the model reads the whole context before it writes its first word.
- **Limits**: every model has a maximum context window.
- **Quality**: the more irrelevant text there is, the easier it is to miss the details that matter.

## 7. Managing the context

Every technique changes **what goes into the message list**:

- **Smaller tool results**: return only what the model needs.
- **Trimming**: drop old messages before calling the LLM.
- **Summarizing**: replace old turns with a short summary.
- **Clearing old tool results**: keep the record that a tool was called, and drop its big output once it has been used.
- **Sub-agents**: hand a heavy subtask to a separate agent with its own context (section 8).
- **Prompt caching**: because the context is append-only, every request *starts with* the previous one. The provider can cache that shared beginning and charge much less to read it again.

These trade off against each other. Trimming and summarizing change the start of the context, which breaks the cache. There is also a model-specific reason to keep the history append-only: the model used in the notebook returns hidden *thinking* blocks that are tied to the exact conversation before them. If you remove or edit earlier messages and then send those blocks back, the API can reject the request. Client-side trimming therefore has to strip the thinking blocks, or you can use the API's server-side compaction and tool-result clearing, which don't count as edits.

## 8. Sub-agents as tools

To an agent, a tool is just a name, a description and some arguments. It doesn't know what runs behind it, which can be **a whole other agent with its own LLM loop**. That is a **sub-agent**.

```
 main agent's context                          sales analyst's context (new for each question)
┌──────────────────────────────┐              ┌───────────────────────────────────────┐
│ [system] [user]              │   question   │ [system] [user: the question]         │
│ [ai → ask_sales_analyst] ────┼─────────────►│ [ai → get_sales_report ×2]            │
│                              │              │ [tool: report] [tool: report]         │
│                              │              │ [ai → add] [tool] ... [ai → subtract] │
│ [tool: one-sentence answer] ◄┼──────────────┤ [tool] [ai: answer]                   │
│ [ai: answer]                 │  answer only └───────────────────────────────────────┘
└──────────────────────────────┘                     thrown away when it's done
```

In the notebook, a "sales analyst" agent is wrapped in an ordinary `@tool` called `ask_sales_analyst`. Every call starts a **fresh conversation** that holds only the question. The analyst reads the reports and does the arithmetic there, and only its final, one-sentence answer goes back to the main agent. The main agent has no report tool at all, so the reports never enter its context.

**What you gain:** the big tool results live in a temporary context that is thrown away. The main conversation stays small no matter how long it continues. The sub-agent can also have its own system prompt and tools, specialized for its job.

**What it costs:**

- The work doesn't disappear. The sub-agent's calls are billed too, and two agents means two system prompts and two sets of tool definitions.
- The sub-agent starts from zero. It sees only the question it is given, so the main agent must ask a complete, self-contained question. The tool's docstring is where you tell it so.
- The main agent sees only the summary. If it later needs a detail the sub-agent left out, it has to ask again.
- The main agent waits while the sub-agent runs its whole loop.

Coding agents use the same pattern: a sub-agent searches a large codebase, reading many files, and the main agent gets back only the few lines that matter.

## 9. In practice: `create_agent`

We build the graph by hand to see how it works. In practice LangChain's `create_agent(model, tools, system_prompt=...)` builds the same ReAct loop in one line, with a `model` node and a `tools` node. It also accepts **middleware**, hooks that run around each model call, which is where context-management techniques such as `SummarizationMiddleware` plug in. An agent made this way can be wrapped in a `@tool` and used as a sub-agent too.

## Key terms

| Term | Meaning |
|---|---|
| ReAct | The reason → act → observe loop an agent runs until it can answer. |
| Harness | The ordinary code around the model that runs tools and manages the conversation (lecture 1). |
| State | The data a LangGraph agent passes between steps. For a ReAct agent, the list of messages. |
| Reducer | The rule that merges a step's output into the state. `add_messages` appends. |
| Node | A function in the graph, such as `agent` or `tools`. |
| Conditional edge | An arrow whose destination depends on the state, such as "tool calls? go to `tools`". |
| Context | Everything sent to the model in one call: system prompt, tool definitions and messages. |
| Input tokens | The size of the context, as counted and billed by the API. |
| Sub-agent | An agent used as a tool by another agent, running in its own fresh context. |
| Prompt caching | Reusing the unchanged beginning of a request so that re-reading it costs less. |

## Key takeaways

- **ReAct is lecture 1's loop with a name:** reason (LLM call), act (tool call), observe (tool result), repeat.
- **LangGraph makes the loop explicit:** a state, nodes and edges. The conditional edge is the loop's exit.
- **Built-in tool calling is still a system prompt**, written by the provider from your tool definitions.
- **The LLM is stateless.** The agent's memory is the message list, and the whole list is sent on every call.
- **Context grows linearly and the bill grows quadratically.** Big tool results make it much worse.
- **A sub-agent is a tool that runs its own agent.** It keeps heavy work out of the main context and returns only a short answer.
