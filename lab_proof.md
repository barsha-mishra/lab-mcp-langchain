# Lab Proof: Agents that Speak MCP Fluently

## 1. What I built

One LangChain agent (`gpt-4o-mini`, created with `create_agent`) that uses tools from **two MCP servers**, each through its own `MultiServerMCPClient`:

| Server | Where it runs | Transport | Secret | Tools |
|---|---|---|---|---|
| Filesystem (`@modelcontextprotocol/server-filesystem`) | Local, on my laptop (started via `npx`) | `stdio` | None, but it can only see `sample_docs/` | 14 |
| GitHub (official server, `https://api.githubcopilot.com/mcp/`) | Remote, on GitHub | `http` | Fine-grained token (read-only, public repos) in the `Authorization` header | 29 |

Both clients use an **evidence interceptor** (`log_tool_calls`) that prints a `[LOG]` line for every tool call (server, tool, inputs) and what the server returned.

I also used a third server for **resources**: the LangChain docs server (`https://docs.langchain.com/mcp`, `http`). Its resource text was put into the system prompt of a tool-less agent.

**Sources:** `sample_docs/project_notes.md` and `sample_docs/faq.md` (fictional client project "Project Atlas"), and the GitHub repository `langchain-ai/langchain-mcp-adapters`.

**Code:** [`Lab_agent_speak_fluently.ipynb`](Lab_agent_speak_fluently.ipynb)

## 2. Query

> "Read our Project Atlas notes (project_notes.md): what constraint do they set about where ticket data may live? Then read the README of the GitHub repository langchain-ai/langchain-mcp-adapters: which transport would let us run an MCP server inside the client's own environment, so that data never leaves it?"

This query needs **both** servers: one half is only in a local file, the other half is only on GitHub.

## 3. Evidence

`[LOG]` lines from the interceptor (both calls ran in parallel, so the "returned" lines appear after both calls):

```
[LOG] filesystem -> read_text_file  {'path': '...\\lab-mcp-langchain\\sample_docs\\project_notes.md'}
[LOG] github -> get_file_contents  {'owner': 'langchain-ai', 'repo': 'langchain-mcp-adapters', 'path': 'README.md'}
[LOG]    returned: # Project Atlas – Project Notes

## Overview
Project Atlas is a support-ticket assistant for our client, Nordlicht Logistics GmbH.
The assistant reads...
[LOG]    returned: successfully downloaded text file (SHA: 0d3e7d0fc7c4facdf437445eaeeb71102de4befe) type='resource' resource=TextResourceContents(uri=AnyUrl('repo://lan...
```

## 4. Final answer

> **Project Atlas Notes (project_notes.md)**
> The constraint set about where ticket data may live is as follows:
> - All ticket data must stay inside the client's own environment. No ticket data may be sent to or stored on external servers.
>
> **GitHub Repository (langchain-ai/langchain-mcp-adapters)**
> In the README of the GitHub repository `langchain-ai/langchain-mcp-adapters`, the transport that would allow running an MCP server inside the client's own environment, ensuring that data never leaves it, is the **`stdio` transport**. This transport is designed primarily to support applications running on a user's machine, making it suitable for environments where data privacy is a concern.

## 5. Trace

| Claim in the answer | Source | Tool call | Verified how |
|---|---|---|---|
| Ticket data must stay in the client's own environment, never on external servers | `sample_docs/project_notes.md`, section "Constraints" | `filesystem -> read_text_file` | Opened the file myself: the sentence matches word for word. Not visible in the `[LOG]` preview (cut at 150 characters). |
| `stdio` is the transport for running a server locally | `README.md` of `langchain-ai/langchain-mcp-adapters` | `github -> get_file_contents` | Searched the README on GitHub for `transport`: found `mcp.run(transport="stdio")` and `"transport": "stdio"` in the local server examples. |
| "Designed primarily to support applications running on a user's machine" | `README.md`, code comment: `# ATTENTION: MCP's stdio transport was designed primarily to support applications running on a user's machine.` | `github -> get_file_contents` | Searched the README on GitHub for `machine`: found the comment. Not visible in the `[LOG]` preview. |
| "Making it suitable for environments where data privacy is a concern" | **Not in the README** | – | Searched the README for `privacy`: no match. This conclusion is the model's own reasoning, not from the source. |

**Other runs that were also grounded:**
- Filesystem agent (Step 4): *engagement lead Maria Keller, go-live 1 March 2027*, after `list_directory` + `read_text_file` on both files.
- GitHub agent (Step 4): transports *stdio, http, streamable_http*, after `get_file_contents` on `README.md`. I checked the README: `stdio` and `http` appear in the examples, and "Streamable HTTP" has its own section.
- My own question (Step 6): *German and English* (from `faq.md`) and latest release *0.3.2* (from `github -> get_latest_release`, `"tag_name":"langchain-mcp-adapters==0.3.2"` visible in the log).
- Resource agent (Step 5): *pass `config={"configurable": {"thread_id": ...}}`*. Evidence check: `'thread_id' in guide_text` → `True`.

## 6. Failure or limitation

**Where the system could produce a plausible but unsupported answer:**

1. **The log doesn't show the evidence.** The interceptor only prints the first 150 characters of each result. For GitHub, those characters are file metadata (SHA, URI), not README text. So the log proves *that* a file was read, but not *what* it says. A reviewer must open the source to check the answer. A fix would be to log the full result (or write it to a `tool_log.jsonl` file).

2. **Grounded but incomplete (Step 5).** The resource agent said I only need a `thread_id` to remember a conversation. The keyword check passed (`thread_id` → `True`), but `"checkpointer" in guide_text` is also `True`: the guide mentions a checkpointer, and the agent left it out. A keyword check proves a word is *there*, not that the answer is *complete*.

3. **Correct reading, wrong interpretation (Step 4).** The GitHub agent listed `http` and `streamable_http` as two different transports. Both are in the README, but they are the same transport (`http` is the short name). Every word was grounded, but the conclusion was slightly misleading.

4. **The agent doesn't always follow the prompt.** The combined agent's system prompt said "list the folder first", but it went straight to `read_text_file`. It only worked because my question named the file. With 43 tools, a small model may guess file names instead.

5. **Write tools are available.** The Filesystem server also offers `write_file`, `edit_file` and `move_file`. With real client data, a wrong tool choice could change or move files. My GitHub token is read-only, but the Filesystem server has no such limit. For client work I would only give the agent read tools.
