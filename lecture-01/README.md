# Lecture 1: What is a harness?

An LLM only turns text into text. It has no abilities of its own. A **harness** is the ordinary code around the model that reads its output, detects tool requests, runs the real functions and feeds the results back.

## Notebook

[`calculator_harness.ipynb`](calculator_harness.ipynb) builds a calculator agent from scratch, without the API's built-in tool calling:

1. Four tools as plain Python functions: `add`, `subtract`, `multiply`, `divide`.
2. A system prompt that tells the model to reply with JSON when it wants a tool.
3. A raw LLM call, showing that a tool request is just a string.
4. `parse_tool_call`, which detects a tool request in the model's text.
5. `execute_tool`, which runs the real function and reports errors back.
6. `run_agent`, the agent loop that ties it all together.

## Running it

```bash
pip install anthropic jupyter
export ANTHROPIC_API_KEY=...
jupyter notebook calculator_harness.ipynb
```
