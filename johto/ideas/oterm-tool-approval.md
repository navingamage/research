# oterm: ask before a tool runs

- **Repo:** https://github.com/ggozad/oterm/tree/de8604c6425f5f4777319d7ac228b6d97e47e8a8 (tag `0.14.7`, 2025-12-19)
- **Stack:** Python 3.10+, Textual 6.11 TUI, ollama-python 0.6, fastmcp 2.14 (MCP client), aiosqlite. Built with uv from PyPI.
- **Domain:** a local LLM chat client with tool calling. Built-in tools (shell, date/time, think), custom Python tools, and MCP server tools, plus MCP prompts and sampling.
- **Status:** ready. Full codebase notes and write-up facts are in [next3/oterm-tool-approval/](../next3/oterm-tool-approval/). The backup is shell_gpt (below).

## Pre-flight (checked 2026-11-06)
- **What it is:** a real application, "the terminal client for Ollama". It's installed from PyPI or Homebrew and keeps chats in a local SQLite store. Single maintainer, MIT, no pricing.
- **Size:** 56 Python files in `src/` (about 130 KB). 13 test modules (17 files under `tests/`).
- **Tests:**
  - `tests/conftest.py` pulls `llama3.2` into a running Ollama before every test, so most of the suite needs a live Ollama server.
  - The MCP tests start their own small servers (`tests/tools/mcp_servers.py`).
- **Clean history:**
  - No `CLAUDE.md`, `AGENTS.md`, `.claude/`, `.cursor` or copilot files in any commit.
  - AI co-authored commits start 2026-02-25 with "Migrate from ollama-python to pydantic-ai".
  - `0.14.7` is the commit right before that, so the pin is the last release on the old architecture.
- **Builds from public sources:** PyPI only (`uv.lock`).
- **Gap at HEAD:** checked on 2026-11-06 at `77335b6`. Nothing in `src/` asks the user, approves or confirms before a tool call.

## How a tool call runs at the pin
- **Choosing tools:** each chat picks its tools in the chat edit screen (`app/chat_edit.py`, `ToolSelector`). The choice is stored as JSON in `chat.tools`.
- **Where calls run:** `OllamaLLM.stream` (`ollamaclient.py:61`) streams the reply. When the model returns `tool_calls`, it stops showing text (`:113`).
  - After the stream ends, it runs every call in order (`:135`), awaiting async tools (`:148`).
  - It then recurses with the tool messages, so a follow-up round can call tools again.
- **Nothing is shown:** while tools run, the chat shows only the loading indicator. Tool calls and results never appear in the chat. They aren't saved to the store or kept in `history` either; only the final assistant text is (`:177`).
- **A failing tool ends the turn quietly** (`:162`–`164`). Then `response_task` sets `history[-1].content` to the empty reply (`chat.py:141`). At that point `history[-1]` is the user's own message, so the user's prompt is blanked in memory.
- **Not a Textual worker:** the reply runs in a plain `asyncio.create_task` (`chat.py:201`, and `:332` for regenerate). `push_screen_wait` only works inside a Textual worker (the `@work` methods). Escape cancels the task (`chat.py:203`, `:175`).
- **The client is rebuilt in 3 places** in `chat.py`: on load (`:79`), after editing the chat (`:242`) and after clearing it (`:268`). MCP sampling builds another one with no tools (`tools/mcp/sampling.py`).
- **Store upgrades are keyed to the package version** (`store/store.py:57`–`63`, `store/upgrades/`). A new column only reaches existing databases if an upgrade step exists and the installed version is at least that step's version.
- **The shell tool** is `subprocess.run(command, shell=True)` with no limits (`tools/shell.py`). MCP tools call straight through to the server (`tools/mcp/tools.py`).

## Gap
- Any tool a chat has enabled runs as soon as the model asks for it. The shell tool runs any command the model writes. There's no way to see the call before it runs, refuse it, or allow a tool for the rest of a chat.

## Where a solution can go wrong
- **Which await works.** A modal awaited from the reply task fails, because that task isn't a Textual worker. A modal that isn't awaited lets the tool run anyway.
- **All the places the client is built.** An approval hook wired only into the first `OllamaLLM(...)` stops asking after the chat is edited or cleared.
- **Several calls in one reply.** Each one needs its own decision. A refused call still needs a tool message, or the model just repeats the call or makes up an answer.
- **Follow-up rounds.** Tools called in the recursive round have to be asked about too.
- **Cancelling while the question is open.** Escape cancels the task. The dialog has to close, and nothing may run afterwards.
- **What the user sees.** The dialog has to show the tool and its arguments, like the exact shell command. With several chat tabs, it has to say which chat is asking. Afterwards the chat still shows nothing about what ran.
- **"Allow for this chat".** Kept in memory, it's lost on restart or when the client is rebuilt. Saved in the store, it needs an upgrade step and a version bump, or existing databases never get the column.
- **The refusal path meets the empty-reply bug.** If refusing ends the turn without text, the user's prompt gets blanked in history (see above).
- **Chats with no tools must behave exactly as before,** and so must existing databases.
- **No easy test oracle.** Most tests need a live Ollama with a model pulled, and nothing tests the TUI. Checking it means a fake client, Textual's `run_test` pilot, or installing Ollama.

## Backup: shell_gpt
- TheR1D/shell_gpt @ `4ea2f834cff5dc020dabb25bb02030c630cd2062` (tag `1.5.0`, 2026-01-28). Python CLI, 42 files, 8 test files.
- No AI files. The first AI co-authored commit is 2026-05-06, after the pin.
- Function calls run without asking (`sgpt/handlers/handler.py`, `handle_function_call`). The same idea fits, with fewer traps:
  - piped stdin with no terminal
  - REPL and chat modes
  - the response cache replaying a call
- It's much smaller, so a model reads all of it quickly.
