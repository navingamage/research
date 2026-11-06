# oterm: codebase context for asking before a tool runs

Pin `de8604c6425f5f4777319d7ac228b6d97e47e8a8` (tag `0.14.7`, 2025-12-19). Paths are under `src/oterm/` unless they start with `src/`, `tests/`, `docs/` or `.github/`, or name a root file such as `pyproject.toml`. A bare `chat.py`, `tool_select.py` or `store.py` means `app/widgets/chat.py`, `app/widgets/tool_select.py` or `store/store.py`. `:line` is at the pin. "(by reading)" means traced, not run.

## 1. What it is and who runs it
- "The terminal client for Ollama" (`pyproject.toml:4`); app subtitle "the TUI Ollama client." (`src/oterm/app/oterm.py:25`).
- Installed with `uvx oterm`, `pip install oterm` or `brew install oterm` (in `homebrew/core` since 0.13.1, `docs/installation.md:6-20`). MIT, one maintainer, no hosted product.
- People use it to chat with local models through an Ollama server. Each chat is a tab with its own model, system prompt, parameters and tools. Chats and messages are kept in a local SQLite file.
- Tool calling: the model can ask for tools that the chat has enabled. Tools are either Python callables named in the config file or tools from MCP servers. The built-in example tools are `think`, `date_time` and `shell` (`docs/tools/index.md:72-90`).

## 2. Repo layout
- 110 tracked files: 57 under `src/` (56 `.py` plus `app/oterm.tcss`), 17 under `tests/`, the rest docs and packaging.
- `src/oterm/`
  - `cli/oterm.py`: typer entry point (`oterm`, `--upgrade`, `--db`, `--data-dir`, `--config`, `--version`)
  - `config.py`: env config and `config.json`
  - `ollamaclient.py`: `OllamaLLM`, the streaming chat client and the tool loop (227 lines)
  - `app/oterm.py`: the Textual `App`, tabs, command palette, startup (284 lines)
  - `app/widgets/chat.py`: `ChatContainer` (one per tab) and `ChatItem` (434 lines)
  - `app/chat_edit.py`, `chat_rename.py`, `chat_export.py`, `prompt_history.py`, `pull_model.py`, `mcp_prompt.py`, `image_browser.py`, `log_viewer.py`: modal screens
  - `app/widgets/tool_select.py`: the tool checkboxes in the chat edit screen
  - `tools/`: registry (`__init__.py`), `external.py` loader, example tools `shell.py`, `date_time.py`, `think.py`
  - `tools/mcp/`: `client.py` (fastmcp client), `setup.py`, `tools.py`, `prompts.py`, `sampling.py`, `logging.py`
  - `store/store.py`: SQLite access; `store/upgrades/`: 13 version-keyed upgrade modules
  - `types.py`: `ToolCall`, `ChatModel`, `MessageModel`
  - `utils.py`: `parse_response` (`<think>` handling), data dir, version helpers, `check_ollama`
- Total Python in `src/`: 3,804 lines.

## 3. Startup and configuration
- `config.py:10` loads `.env`. `EnvConfig` (`:13-19`): `OLLAMA_HOST` (default `127.0.0.1` port `11434`), `OLLAMA_URL`, `OTERM_VERIFY_SSL`, `OTERM_DATA_DIR`.
- Data dir default: `~/.local/share/oterm` on Linux (or `$XDG_DATA_HOME`), `~/Library/Application Support/oterm` on macOS (`utils.py:110-124`).
- `AppConfig` reads `config.json` from the data dir and writes it back if missing, at import time (`config.py:27-42`, `:57`).
- `OTerm.on_mount` (`app/oterm.py:220-266`): store, theme, saved chats, keymap remaps (`:236-238`), then `load_mcp` (`:240`).
- `load_mcp` (`:199-208`):
  - Tools from the config `tools` list go into `available_tool_defs["external"]` (`:203-205`).
  - MCP servers from `mcpServers` add one group per server (`:206-208`, `tools/mcp/setup.py:13-53`).
- Every saved chat becomes a `ChatContainer` in a `TabPane` with id `chat-<id>` (`app/oterm.py:249-260`). Messages render when the tab is activated, in a worker (`:272-276`).

## 4. Tools: where they come from and how a chat picks them
- Registry: `available_tool_defs: dict[str, list[ToolCall]]`, keyed by group (`"external"` or the MCP server name) (`tools/__init__.py:5`); `available_tool_calls()` flattens it (`:8-9`).
- `ToolCall` is a TypedDict with `tool` (an `ollama.Tool`) and `callable` (`types.py:15-17`).
- Config-defined tools: `load_external_tools` imports `module:object` pairs (`tools/external.py:10-40`). The built-in example tools are only available once listed there (`docs/tools/index.md:80`; the doc's example under "enable the shell tool" actually shows the `think` tool, `:82-89`).
- `shell` tool: `ShellTool` (`tools/shell.py:5`); `shell_command` runs `subprocess.run(command, shell=True, capture_output=True)` and returns stdout only (`:23-26`). No timeout, no working-directory choice, no stderr. It's synchronous, so it runs on the event loop and the UI waits for it (by reading).
- MCP tools: `MCPToolCallable.call` forwards kwargs to `client.call_tool` and joins the text parts (`tools/mcp/tools.py:14-17`). `MCPClient.call_tool` catches any exception and returns `[]` (`tools/mcp/client.py:182-196`), so a failing MCP tool reaches the model as an empty result.
- Per-chat choice: the chat edit screen has a `ToolSelector` (`app/chat_edit.py:226-227`) with one "select all" checkbox per group and one checkbox per tool (`app/widgets/tool_select.py:68-91`). It's disabled when the model doesn't report the `tools` capability (`app/chat_edit.py:181-183`).
- Saved as JSON of full `Tool` dumps in `chat.tools` (`store/store.py:101-103`, `:141-143`). When a chat loads, it keeps only stored tools that are equal to a currently loaded tool (`app/widgets/chat.py:73-77`, `tool_select.py:25-27`). A tool whose description or schema changed on the server silently drops out of the chat.

## 5. The reply path in a chat tab (`app/widgets/chat.py`)
- `ChatContainer` keeps `self.ollama`, an `OllamaLLM` built in 4 places:
  - class attribute default (`chat.py:35`)
  - `__init__` on load (`chat.py:79-88`)
  - after editing the chat (`chat.py:242-251`, inside the `@work` method `action_edit_chat`, `:207-251`)
  - after clearing the chat (`chat.py:268-277`)
  - plus one more with no tools for MCP sampling (`tools/mcp/sampling.py:47-52`)
- Submit: `on_submit` starts `response_task` with `asyncio.create_task` (`chat.py:191-201`). It's a plain asyncio task, not a Textual worker.
- `response_task` (`chat.py:115-189`):
  - mounts the user item, an empty assistant item and a `LoadingIndicator` (`chat.py:118-127`)
  - streams `self.ollama.stream(...)` and re-renders the assistant item on every chunk (`chat.py:132-136`)
  - then sets `self.ollama.history[-1].content` to the parsed reply (`chat.py:141`) and saves the user and assistant messages (`:148-173`)
  - `CancelledError`: removes both items and puts the prompt back in the input (`chat.py:175-179`)
  - `ResponseError`: removes both items and shows a notification (`chat.py:180-186`)
  - always removes the loading indicator (`chat.py:188-189`)
- Escape: `key_escape` cancels `self.inference_task` (`chat.py:203-205`). The class binding names a `cancel_inference` action (`:40-42`) that has no `action_` method. In Textual 6.11.0 a binding without a target returns False, so the key falls through to `key_escape` (by reading).
- Regenerate (command palette "Regenerate last Ollama message", `app/oterm.py:58-62`): `action_regenerate_llm_message` (`chat.py:284-332`) drops the last 2 history entries and calls `self.ollama.stream(...)` again with a random seed, in a second `asyncio.create_task` (`:332`) that isn't stored. Escape can't cancel it. It saves the raw text including `<think>` (`:319-326`), unlike `response_task`.
- MCP prompt (command palette): `action_mcp_prompt` is a `@work` worker that awaits `response_task` directly (`:349-382`). Here the same `response_task` runs inside a worker.
- Clear chat rebuilds the client with the old tool defs (`:265-282`). Edit chat rebuilds it from the new tool list (`:234-251`) and resends assistant history with `<think>` text included (`:226-232`; on load it is stripped, `:62-69`).

## 6. The tool loop (`ollamaclient.py`)
- `OllamaLLM.__init__` (`ollamaclient.py:37-59`): `tool_defs` kept as given, `self.tools` = the `Tool` objects sent to Ollama (`:54-55`).
- `stream()` (`ollamaclient.py:61-177`), one call per model round:
  - New `AsyncClient` per call (`ollamaclient.py:73-75`). The user prompt is appended to `self.history` (`:78-82`).
  - `client.chat(..., messages=self.history + tool_call_messages, tools=self.tools, stream=True)` (`ollamaclient.py:90-99`).
  - The first chunk that carries `tool_calls` sets `pending_tool_call` and keeps that message (`ollamaclient.py:112-114`). Later chunks with more tool calls aren't merged.
  - Text is yielded only while no tool call is pending (`ollamaclient.py:117-122`). So a reply that calls tools shows nothing until the follow-up round.
  - After the stream: tool messages start with the assistant message (`ollamaclient.py:127`), plus the thinking if any (`:132-133`).
  - For each tool call (`ollamaclient.py:135`), linear search by name in `tool_defs` (`:140-142`). An unknown name gets no tool message at all.
  - Sync or async callable (`ollamaclient.py:147-150`); the result goes in as `{"role": "tool", "content": str(...), "name": ...}` (`:155-161`).
  - Any exception: `log.error` and `return` (`ollamaclient.py:162-164`). The turn ends with no text and no history entry.
  - Then it recurses with `tool_call_messages=tool_messages` and no prompt (`ollamaclient.py:167-174`). The recursion is the follow-up round, and it can call tools again.
  - Only the final text is appended to history (`ollamaclient.py:176-177`). Tool calls and results are never kept in `history`, never saved, and never shown.
- Round context (`ollamaclient.py:90-99`, `:167-170`): round 2 sends history + round-1 tool messages. Round 3 sends history + round-2 tool messages only, so round-1 calls drop out of the request (by reading).
- Tool arguments are passed as `**kwargs` straight from the model (`ollamaclient.py:138`, `:148`, `:150`). Unexpected or missing argument names raise `TypeError`, which takes the silent `return` path.

## 7. What the user sees today
- While a tool runs, the chat shows the loading indicator only. No tool name, arguments or result appear in the chat or the store.
- Tool calls go to the log viewer at debug level only (`ollamaclient.py:145`, `:152`). Ctrl+L opens it (`app/oterm.py:32`).
- The empty-reply case: when the loop returns early, `response` stays `""`, and `history[-1]` (the user's own message, appended at `ollamaclient.py:82`) is set to `""` (`chat.py:141`). An empty assistant message is saved too (`:163-172`) (by reading).

## 8. Store and upgrades (`store/store.py`)
- One file, `store.db` in the data dir (`store.py:24-26`). `Store.get_store()` is a class-level singleton (`:17-22`, `:69`).
- New DB: tables `chat` (`tools TEXT DEFAULT "[]"`, `thinking BOOLEAN`) and `message` (`author`, `text`, `images`) (`store.py:28-56`), then `PRAGMA user_version` set from the installed package version (`:57`).
- Existing DB: runs every upgrade step whose version is `<=` the installed version and `>` the DB's `user_version` (`store.py:58-68`). The installed version comes from `importlib.metadata.version("oterm")` (`:60`).
- Upgrade steps: tuples of `(version, [async steps])` (`store/upgrades/__init__.py:15-29`), e.g. `0.13.1` drops a column by rebuilding the table and adds `thinking` (`store/upgrades/v0_13_1.py:8-49`).
- So a new column needs an upgrade step and a version bump to at least that step's version. With an editable install, the metadata version only changes after reinstalling (by reading).
- Each method opens its own `aiosqlite` connection (`store.py:84-260`). `save_message` uses `INSERT OR REPLACE` (`store.py:215-232`).

## 9. Textual specifics that decide the design (Textual 6.11.0 from `uv.lock`)
- `push_screen(..., wait_for_dismiss=True)` (what `push_screen_wait` does) raises `NoActiveWorker` outside a worker (Textual 6.11.0 wheel, `textual/app.py` lines 2825-2830).
- Workers in this repo: `@work` methods (`chat.py:207`, `:253`, `:349`; `app/oterm.py:97`, `:210`, `:272`).
- A plain task can still wait on a modal through `push_screen(screen, callback)` plus a future, or the reply can move into a worker. A worker is cancelled through the worker, not `Task.cancel()`.
- Modal pattern in the repo: `ModalScreen[str]` with an `escape` binding that calls `dismiss()` with no value (`app/chat_rename.py:8-25`). CSS classes `screen-container`, `short`, `title`, `button-container` (`app/oterm.tcss:108-126`).
- Key routing: while a modal is open, keys go to the modal's own bindings. The chat's `key_escape` isn't reached, so Escape closes the dialog and doesn't cancel the reply (by reading).
- Several tabs can stream at once. Each tab has its own task and client.

## 10. Build, run and tests
- Dev setup: `uv sync` (`docs/development/index.md`). Python `>=3.10` (`pyproject.toml:21`). Pinned deps: textual 6.11.0, ollama 0.6.x, fastmcp 2.14.x, aiosqlite 0.22.x, pydantic 2.12.x (`pyproject.toml:22-36`, `uv.lock`).
- Run: `uv run oterm`. It needs an Ollama server at `OLLAMA_URL`. `check_ollama` notifies and exits when it can't connect (`utils.py:184-213`).
- Tests: 13 test modules, 38 test functions.
  - `tests/conftest.py:17-23` is an autouse fixture that calls `ollama.show("llama3.2")` and pulls it if missing. With no Ollama server running, every test errors, including `test_utils.py` and `test_store.py`.
  - `test_shell_tool.py`, `test_llm_client.py::test_tool_streaming` and the MCP tool tests check tool calling end to end through a real model.
  - MCP test servers: `tests/tools/mcp_servers.py`, started with `mcp run` (`tests/conftest.py:45-60`).
  - Nothing tests the TUI (no `run_test()` pilot anywhere) and nothing tests a refusal path.
- CI: `.github/workflows/` only builds the docs and publishes to PyPI. No workflow runs the tests.
- Lint and types: ruff with E, F, UP, I (`pyproject.toml:68-78`); pyright config (`:86-88`); `.pre-commit-config.yaml`.

## 11. Conventions a change should follow
- Modal screens live in `src/oterm/app/<name>.py` as `ModalScreen[...]` subclasses with `action_cancel` → `dismiss()`.
- User notices go through `self.app.notify(...)`. Logging goes through `oterm.log.log` (`log.py:54`), which the log viewer shows.
- Per-chat settings live on `ChatModel` (`types.py:40-51`) and the `chat` table. New persisted fields go through a store upgrade step (§8).
- Command palette entries are `SystemCommand`s in `OTerm.get_system_commands` (`app/oterm.py:36-75`). Key bindings can be remapped through `keymap` in `config.json` (`:236-238`), which uses binding ids.

## 12. Known bugs and traps at the pin
- A modal awaited from the reply task raises, because the task isn't a worker (§9). A modal that isn't awaited lets the tool run anyway.
- The client is rebuilt on edit and clear (§5). A hook passed only in `__init__` stops applying after either.
- Several tool calls in one reply are one loop (`ollamaclient.py:135`). Each needs its own decision. A refused call without a tool message leaves an assistant tool call with no result.
- Follow-up rounds go through the same `stream()` recursion (`ollamaclient.py:167`). Tools called there need the same decision.
- Escape while a dialog is open goes to the dialog (§9). The reply task keeps waiting unless the dialog's answer ends it.
- Regenerate and MCP prompts reach the same loop from different contexts: an unstored task, and a worker (§5).
- The silent `return` on a tool exception, and the blanked user message after an empty reply (§6-7).
- The shell tool blocks the event loop while it runs (§4), so nothing can be clicked or cancelled meanwhile.
- MCP tool failures come back as an empty result, not an error (§4).
- Tool calls aren't kept in history, so the next turn doesn't know a tool ran or was refused (§6).
- A stored tool that no longer equals the server's tool silently drops out of the chat (§4).
- Store upgrades are keyed to the installed version (§8).

## 13. Pin vs HEAD
- HEAD `77335b6` (2026-11-06) is a rewrite on pydantic-ai. The first commit after the pin is "Migrate from ollama-python to pydantic-ai" (2026-02-25).
- Nothing in `src/` at HEAD asks before a tool call, approves or confirms one (checked 2026-11-06). HEAD still cancels with `key_escape`.

## 14. Unverified items
- Textual behaviour (binding fall-through, key routing while a modal is open) is from reading the 6.11.0 source, not run.
- What a given Ollama model does after a refused call (stops, retries, or makes up an answer) depends on the model. Not run.
- Whether the `llama3.2` default reliably emits two tool calls in one reply. Not run.
