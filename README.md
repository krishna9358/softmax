# AI Agent

A terminal coding agent, built from scratch in plain Python.

No LangChain, no LlamaIndex, no CrewAI, no agent framework of any kind. The
agent loop, the tool protocol, context management, approval policy, hooks,
subagents and session persistence are all written here, in this repo, from the
raw model API up. That is the point of the project: every part of how an agent
actually works is visible and editable rather than hidden behind someone else's
abstraction.

The dependencies that remain are thin, single-purpose libraries — an HTTP
client, an arg parser, a terminal renderer, a tokenizer, a schema validator.
None of them decide how the agent behaves.

| Written from scratch | Delegated to a library |
|---|---|
| Agentic loop, turn management | `openai` — HTTP transport to the model endpoint |
| Tool base class, schemas, dispatch | `pydantic` — parameter schemas and validation |
| Tool registry and discovery | `click` — CLI argument parsing |
| Context accounting, pruning, compaction | `rich` — terminal rendering |
| Loop detection | `tiktoken` — token counting |
| Approval policies and safety checks | `httpx` — `web_fetch` requests |
| Hook system | `ddgs` — search results for `web_search` |
| Subagent orchestration | `fastmcp` — MCP wire protocol |
| Session persistence and checkpoints | `platformdirs`, `tomli` — config paths and TOML |
| Streaming response parsing | |
| System prompt construction | |

The two genuine protocol dependencies are `openai` (used only as an HTTP client
against an OpenAI-compatible endpoint — swapping it for raw `httpx` would change
nothing architecturally) and `fastmcp` (which speaks the MCP stdio/SSE wire
format; the tool adapter and server manager around it are hand-written).

## Setup

Requires Python 3.11+.

```bash
python3 -m venv .venv
source .venv/bin/activate

pip install openai pydantic click rich tiktoken httpx platformdirs tomli ddgs fastmcp
```

Configure the model endpoint with two environment variables:

```bash
export API_KEY="your-openrouter-key"      # https://openrouter.ai/keys
export BASE_URL="https://openrouter.ai/api/v1"
```

Both are read at call time from the environment (`config/config.py`). The agent
exits with an error if `API_KEY` is unset.

## Usage

**Interactive mode** — a REPL that keeps conversation state across turns:

```bash
python main.py
```

**Single-shot mode** — one prompt, one answer, then exit:

```bash
python main.py "explain what tools/registry.py does"
```

**Point it at another directory:**

```bash
python main.py --cwd ~/some/project
```

### Slash commands (interactive mode)

| Command | What it does |
|---|---|
| `/help` | List commands |
| `/config` | Show model, temperature, approval policy, cwd, max turns |
| `/model [name]` | Show or switch the model |
| `/approval [policy]` | Show or set approval policy |
| `/tools` | List registered tools |
| `/mcp` | Show connected MCP servers |
| `/stats` | Token usage and turn count |
| `/clear` | Wipe conversation context and loop-detector state |
| `/save` | Save the current session |
| `/sessions` | List saved sessions |
| `/resume <id>` | Resume a saved session |
| `/checkpoint` | Snapshot the current session |
| `/restore` | Restore from a checkpoint |
| `/exit`, `/quit` | Leave |

### Approval policies

Every mutating tool call (write, edit, shell) passes through the approval
manager in `safety/approval.py`:

- `on-request` — ask before mutating operations (default)
- `auto` — approve reads and writes inside cwd, ask for anything outside
- `never` — reject everything requiring approval
- `yolo` — approve everything, including flagged-dangerous shell commands

## Configuration

Config is merged from three layers, later overriding earlier:

1. System config — `config.toml` in the platform config dir
2. Project config — `.ai-agent/config.toml` in the working directory
3. CLI flags

```toml
# .ai-agent/config.toml
approval = "on-request"
max_turns = 100
hooks_enabled = true

[model]
name = "mistralai/devstral-2512:free"
temperature = 1.0
context_window = 256000

[[hooks]]
name = "run tests after edits"
trigger = "after_tool"
command = "python3 -m pytest -q"

[mcp_servers.filesystem]
command = "npx"
args = ["-y", "@modelcontextprotocol/server-filesystem", "."]
```

An `AGENT.MD` file in the working directory is loaded as project instructions
and appended to the system prompt.

## Project structure

```
main.py               CLI entrypoint (Click) — arg parsing, REPL, slash commands
apply_patch.py        Multi-file patch tool (see Known issues)

agent/
  agent.py            The agentic loop: call model, run tools, feed results back
  session.py          Owns per-conversation state and wires every subsystem
  events.py           Typed events the loop emits for the UI to render
  persistence.py      Session save/resume and checkpoint/restore on disk

client/
  llm_client.py       OpenAI-SDK wrapper: streaming, retries, backoff
  response.py         Stream event, tool call, and token-usage types

tools/
  base.py             Tool ABC, ToolResult, ToolKind, confirmation types
  registry.py         Registers tools, exposes JSON schemas to the model
  discovery.py        Loads user tools from .ai-agent/tools/*.py
  subagents.py        Spawns sub-agents with their own tools and turn limits
  builtin/            read_file, write_file, edit_file, list_dir, grep, glob,
                      shell, web_search, web_fetch, memory, todo
  mcp/                MCP client, server manager, tool adapter

context/
  manager.py          Message history, token accounting, tool-output pruning
  compaction.py       Summarizes old turns when nearing the context window
  loop_detector.py    Detects repeated tool calls and breaks cycles

config/
  config.py           Pydantic schema — model, hooks, approval, MCP servers
  loader.py           Layered TOML loading, AGENT.MD discovery

safety/approval.py    Approval policies, dangerous-command and path checks
hooks/hook_system.py  Shell hooks on agent/tool lifecycle events
prompts/system.py     System prompt construction
ui/tui.py             Rich rendering — output, tool calls, diffs, help
utils/                Errors, path resolution, text formatting
```

## How it works

**The loop.** `agent/agent.py` is the core. It sends the conversation plus tool
schemas to the model, streams the reply, and if the model asks for tools it runs
them, appends the results as new messages, and calls the model again. That
repeats until the model answers without requesting a tool, or `max_turns` is
hit. Each step emits an event (`agent/events.py`) that `ui/tui.py` renders, so
the loop never touches the terminal directly.

**Session as the wiring point.** `agent/session.py` builds one object holding
the tool registry, context manager, approval manager, hook system, MCP manager
and loop detector. The agent talks only to the session, which is why subsystems
can be added without touching the loop.

**Tools.** Everything the agent can do is a `Tool` subclass (`tools/base.py`)
exposing a JSON schema. `tools/registry.py` collects them and hands the schemas
to the model. Three sources feed it: the built-ins, Python files dropped in
`.ai-agent/tools/`, and tools proxied from MCP servers.

**Staying inside the context window.** `context/manager.py` counts tokens with
tiktoken and prunes stale tool output first, since that is the bulkiest and
least reusable content. If pruning isn't enough, `compaction.py` summarizes
older turns into a single message and keeps recent ones verbatim.

**Guardrails.** Mutating calls go through `safety/approval.py` before running.
`context/loop_detector.py` watches for the agent repeating identical calls and
injects a loop-breaker prompt to knock it out of the cycle.

## Known issues

- `apply_patch.py` imports from `unified_agent.tools.base`, a package name that
  no longer exists, so the module fails to import. It is also not registered in
  `tools/registry.py` — yet `prompts/system.py` instructs the model to use
  `apply_patch` for multi-file edits. Fix the import and register it, or drop
  the mention from the prompt.
- No test suite.
- No `requirements.txt` — dependencies are listed in Setup above.
