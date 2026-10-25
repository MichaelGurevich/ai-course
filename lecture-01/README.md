# Lecture 1: Intro to harness

An LLM only turns text into text. It has no abilities of its own. A **harness** is the ordinary code around the model that reads its output, detects tool requests, runs the real functions and feeds the results back.

See [`intro-to-harness.md`](intro-to-harness.md) for the full lecture overview: goals, key concepts and outline.

## Notebook

[`calculator_harness.ipynb`](calculator_harness.ipynb) builds a calculator agent from scratch, without the API's built-in tool calling:

1. Four tools as plain Python functions: `add`, `subtract`, `multiply`, `divide`.
2. A system prompt that tells the model to reply with a JSON list of tool calls when it wants tools.
3. A raw LLM call, showing that a tool request is just a string.
4. `parse_tool_calls`, which detects tool requests in the model's text.
5. `execute_tool`, which runs the real function and reports errors back.
6. `ask`, one full round trip: the model requests tools, the harness runs them and sends the results back, and the model answers.
7. Examples: a single `add` call, a single `multiply` call, and parallel tool calls (three calculations in one prompt).

## Running it

```bash
pip install anthropic jupyter
export ANTHROPIC_API_KEY=...
jupyter notebook calculator_harness.ipynb
```
