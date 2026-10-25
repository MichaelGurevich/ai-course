# Lecture 1: Intro to harness

## What this lecture is about

An LLM by itself has no capabilities. It takes text in and gives text out. It can't run code, do reliable arithmetic, call an API or read a file.

Everything an "AI agent" seems to do comes from the **harness**: the ordinary code around the model. The harness tells the model which tools exist, reads what the model writes, recognizes tool requests, runs the real functions and sends the results back.

In this lecture we build a harness from scratch, at the lowest level. We don't use the API's built-in tool calling. The model writes JSON as plain text, and our own code parses it and calls the functions.

## Learning goals

By the end of the lecture, students should be able to:

- Explain why an LLM can't act on its own, and what "a tool call" really is: text in an agreed format.
- Describe the parts of a harness: tool definitions, system prompt, parser, tool executor.
- Write a system prompt that defines a tool-calling protocol.
- Parse a model's output to detect tool requests, including malformed or wrapped JSON.
- Run the requested tools, handle errors and send results back to the model.
- Explain parallel tool calls: several independent calls in one model reply.

## Key concepts

| Concept | Meaning |
|---|---|
| **LLM** | A function from text to text. It can only *ask* for actions, never perform them. |
| **Tool** | A normal function (here: `add`, `subtract`, `multiply`, `divide`) that the harness can run. |
| **Harness** | The code around the model that turns its text into real actions. |
| **Protocol** | The format the model and harness agree on, defined in the system prompt. Here: `[{"tool": ..., "arguments": {...}}]`. |
| **Tool call** | A reply from the model that matches the protocol. Just a string until the harness acts on it. |
| **Tool result** | The output of the real function, sent back to the model as text. |
| **Parallel tool calls** | Several independent tool calls in one reply, run by the harness and returned together. |

## Lecture outline

The lecture follows the notebook [`calculator_harness.ipynb`](calculator_harness.ipynb):

1. **The big idea:** text in, text out, and the role of the harness.
2. **Tools:** four calculator functions and a registry that maps names to functions.
3. **System prompt:** describing the tools and the JSON protocol to the model.
4. **The LLM without a harness:** the model "asks" for a tool, and nothing happens.
5. **Parsing:** `parse_tool_calls` reads the model's text and finds tool requests.
6. **Executing:** `execute_tool` runs the real function and reports errors as text.
7. **Putting it together:** `ask` runs one full round trip.
8. **Examples:**
   - A single `add` call.
   - A single `multiply` call.
   - Parallel tool calls: three calculations in one prompt.
9. **Wrap-up:** how this maps to built-in tool calling in real APIs, plus exercises.

## Key takeaways

- The LLM only produces text. A tool call is just text in a format we agreed on.
- The harness gives the model its abilities.
- The system prompt is the contract between the model and the harness.
- Built-in tool calling in modern APIs does the describing and parsing for you, but running the tool and sending back the result is still the harness's job.

## What comes next

`ask` does a single round trip, so it can't handle a calculation where one step needs another step's result, like "add 12 and 30, then multiply by 4". Solving that needs an **agent loop**: keep calling the model and running tools until it gives a final answer.
