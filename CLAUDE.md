# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A CLI chat app that talks to the Anthropic API and exposes documents/tools to the model via MCP (Model Context Protocol). It's a learning project: `mcp_client.py` and `mcp_server.py` are intentionally incomplete scaffolding with `# TODO` markers — the point of the exercise is to implement the MCP protocol calls and server tools/resources/prompts yourself (see "Implementing MCP Features" below).

## Commands

Setup (uv, recommended):
```
uv venv
.venv\Scripts\Activate.ps1        # PowerShell; use .venv/bin/activate on macOS/Linux
uv pip install -e .
uv run main.py
```

Setup without uv:
```
python -m venv .venv
.venv\Scripts\activate
pip install anthropic python-dotenv prompt-toolkit "mcp[cli]==1.8.0"
python main.py
```

`.env` must set both `CLAUDE_MODEL` and `ANTHROPIC_API_KEY` (README only documents the latter) — `main.py` asserts both are non-empty at startup. `USE_UV=1` controls whether the doc MCP server subprocess is launched via `uv run mcp_server.py` or plain `python mcp_server.py`.

There are no lint, type-check, or test commands configured in this repo.

## Architecture

`main.py` loads `.env`, starts `mcp_server.py` as a stdio subprocess wrapped in an `MCPClient` (`doc_client`), and enters an async CLI loop (`CliApp.run`). Any extra CLI args to `main.py` are treated as additional MCP server scripts to launch the same way — don't pass flags like `--help` here, they get forwarded as a server script name and break the MCP stdio handshake.

Request flow: `CliApp` (core/cli.py, prompt_toolkit-based REPL with `@doc` and `/command` completion) → `CliChat` (core/cli_chat.py, extends `Chat`) → `Chat.run` (core/chat.py) loops calling `Claude.chat` (core/claude.py, thin Anthropic SDK wrapper) and, when the model requests a tool, dispatches through `ToolManager` (core/tools.py) to whichever `MCPClient` (mcp_client.py) exposes that tool — looping until the model returns a non-tool-use response.

`CliChat._process_query` handles two special input forms before falling back to a plain chat turn:
- `/command doc_id` → resolved via `doc_client.get_prompt` (an MCP prompt), converted to message params.
- `@doc_id` mentions anywhere in the query → matching doc content is fetched via `doc_client.read_resource` and injected as `<context>` in a wrapper prompt.

`MCPClient` and `mcp_server.py`'s `docs` dict currently back these MCP calls with stub `# TODO` methods (list_tools/call_tool/list_prompts/get_prompt/read_resource all return empty) and a hardcoded `docs` dict with no tools/resources/prompts registered yet — until those TODOs are implemented, tool calls and `@doc`/`/command` features won't actually return real data.

## Known environment gotcha

Do not let this project's `.venv` live inside a cloud-synced folder (OneDrive, Dropbox, etc.) — `uv` can hit `Access is denied` errors deleting/replacing files under `.venv\Lib\site-packages\*.dist-info` because the sync client locks files mid-write, and repeated retries can corrupt the venv. If this happens: `Remove-Item -Recurse -Force .venv` then `uv venv` and `uv pip install -e .` to rebuild clean. Venvs are also not portable between machines/OSes (absolute paths, platform-specific binaries) — always rebuild `.venv` fresh per machine rather than copying it; only the source (tracked in git) should travel.
