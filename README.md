# Learning Agentic AI: LangGraph + MCP

This folder documents a progressive, hands-on exploration of building tool-using agents with LangGraph, ending with integration into the Model Context Protocol (MCP). This is a **learning project**, not a polished product — each file is a deliberate step up in capability from the one before it.

## How I learned this (progression)

**1. `chatbot.py` — Tool calling, synchronous**
Started with the simplest possible version: one LangGraph node (`chat_node`) that calls an LLM bound to a single hardcoded tool (a basic calculator with add/sub/mul/div). Used `add_conditional_edges` with `tools_condition` so the graph itself decides — based on the model's output — whether to route to the tool node or end the conversation, instead of me hardcoding that logic.

**2. `chatbot_async.py` — Same graph, made async**
Converted `chat_node` to an `async def` and switched `.invoke()` calls to `.ainvoke()`. This step was really about understanding *why* async matters for agents specifically: once you're calling external tools or APIs, blocking synchronous calls stall the whole graph, whereas async lets the event loop do other work while waiting on I/O.

**3. `chatbot_mcp.py` — Replacing hardcoded tools with MCP**
This is where it became genuinely agentic infrastructure rather than a toy example. Instead of defining tools directly in Python with `@tool`, I connected to external MCP servers using `MultiServerMCPClient`:
- An arithmetic MCP server over **stdio** transport (a local Python subprocess)
- A remote expense-tracking MCP server over **streamable HTTP** transport (the one I built myself — see the separate Expense Tracker MCP Server project)

The graph structure stayed almost identical to step 1 — same `chat_node` → conditional edge → `tools` node → back to `chat_node` loop. What changed was *where the tools come from*: `await client.get_tools()` pulls the tool definitions dynamically from whichever MCP servers are configured, and `llm.bind_tools(tools)` doesn't care whether those tools are local Python functions or remote MCP tools.

## What clicked for me

- **MCP decouples "what tools exist" from "how the agent is built."** The same `chat_node` / `tools_condition` / `tool_node` pattern works whether tools are hardcoded functions or MCP servers — MCP just changes the tool *source*, not the graph logic.
- **`tools_condition` removes manual routing.** I don't write an if/else to decide "should this go to the tool node" — LangGraph inspects whether the LLM's response includes a tool call and routes automatically.
- **Transport choice matters.** stdio is simple for local processes (spawns a subprocess and talks over stdin/stdout); streamable HTTP is what you need for a server that isn't running on the same machine — like the expense tracker server, which is deployed remotely.

## What I'd build next

- Add persistence (checkpointing) so a conversation using MCP tools survives a restart — I've done this in a separate LangGraph chatbot project and want to combine the two
- Add a second local MCP server of my own (beyond the expense tracker) to practice the stdio transport path end-to-end, not just HTTP
- Handle the case where an MCP server is unreachable — right now there's no fallback if `client.get_tools()` fails

## Tools

Python, LangGraph (`StateGraph`, `ToolNode`, `tools_condition`), LangChain, `langchain-mcp-adapters` (`MultiServerMCPClient`), OpenAI (gpt-5), asyncio

## Files

```
├── chatbot.py         # Step 1: sync tool calling with a hardcoded calculator tool
├── chatbot_async.py   # Step 2: same graph, converted to async
└── chatbot_mcp.py      # Step 3: tools sourced from MCP servers (stdio + streamable HTTP)
```
