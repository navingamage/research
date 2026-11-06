# oterm, ask before a tool runs: facts for the four write-ups

Facts, numbers and open choices only, as fragments. The write-ups themselves get written separately.

Pin `de8604c6425f5f4777319d7ac228b6d97e47e8a8` (tag `0.14.7`, 2025-12-19); HEAD `77335b6` (2026-11-06) for comparison. Paths are under `src/oterm/` unless they start with `tests/` or `docs/`, and `:line` is at the pin. More detail in `codebase.md`. "(by reading)" = traced, not run.

## 1. The ask

**Domain context facts**
- Tool calling: the model replies with `tool_calls` (a name plus JSON arguments); the client runs them and sends each result back as a `role: tool` message; the model then answers or calls more tools
- One reply can hold several tool calls; a follow-up round can call tools again
- MCP: servers expose tools to a client over stdio or HTTP; the MCP spec's tools page recommends a human in the loop who can deny tool calls, with a confirmation prompt before they run
- Usual shapes of the feature in other tools: ask every time; allow once; allow a tool for the rest of a session; deny, with the model told so

**oterm facts behind the ask**
- Tools reach a chat two ways: `module:object` pairs in the `tools` list of `config.json` (the built-in `think`, `date_time`, `shell` examples included), and tools from `mcpServers` (`app/oterm.py:199-208`)
- Per chat: checkboxes in the chat edit screen (`app/widgets/tool_select.py:68-91`), saved as JSON on the chat (`store/store.py:101-103`)
- `shell`: `subprocess.run(command, shell=True, capture_output=True)`, stdout only, no timeout, in oterm's working directory (`tools/shell.py:23-26`); synchronous, so the UI waits while it runs (by reading)
- Repo docs on the shell tool: "Obviously this can be dangerous" (`docs/tools/index.md:78`)
- MCP tools: call straight through to the server (`tools/mcp/tools.py:14-17`)
- While tools run the chat shows only the loading indicator; tool names, arguments and results appear nowhere in the chat (`ollamaclient.py:117-122`); debug lines in the log viewer only (`ollamaclient.py:145`, `:152`)
- Nothing about tool calls is saved; only the final text (`ollamaclient.py:176-177`, `app/widgets/chat.py:148-173`)

**The gap**
- Every tool call runs as soon as the model asks (`ollamaclient.py:135-161`)
- No confirm, approve or deny anywhere in `src/` at the pin or at HEAD (checked 2026-11-06)
- No way to see the exact command before it runs, refuse it, or allow a tool for the rest of a chat

**Entry paths into the tool loop** (all reach `OllamaLLM.stream`)
- Submit: `asyncio.create_task(self.response_task(...))` (`app/widgets/chat.py:201`)
- Regenerate, from the command palette: a second unstored `asyncio.create_task` (`app/widgets/chat.py:332`)
- MCP prompt, from the command palette: inside a `@work` worker (`app/widgets/chat.py:349-382`)
- MCP sampling: its own client with no tools (`tools/mcp/sampling.py:47-52`), so no tool calls there

## 2. Goal and success checks

**Repo context a domain expert needs**
- Textual 6.11.0 app; each chat tab is a `ChatContainer` with its own `OllamaLLM` client
- The reply runs in a plain asyncio task, not a Textual worker (`app/widgets/chat.py:201`); Escape calls `Task.cancel()` (`app/widgets/chat.py:203-205`); `CancelledError` puts the prompt back in the input (`app/widgets/chat.py:175-179`)
- Waiting on a modal's answer (`push_screen_wait`) works only inside a worker; outside one it raises `NoActiveWorker` (Textual 6.11.0 source)
- While a modal is open, keys go to the modal; the chat's Escape handler isn't reached (by reading)
- Modal screens: `ModalScreen[...]`, `dismiss(value)`; Escape → `dismiss()` with no value is the repo pattern (`app/chat_rename.py:8-25`)
- The tool loop is a recursion, one call per model round (`ollamaclient.py:167-174`); tool results go in as `role: tool` dicts (`ollamaclient.py:155-161`)
- A tool exception ends the turn silently (`ollamaclient.py:162-164`); then the user's own message is blanked in memory (`app/widgets/chat.py:141`)
- The client is rebuilt after editing (`app/widgets/chat.py:242-251`) and after clearing a chat (`app/widgets/chat.py:268-277`)
- Store upgrades run only for steps with version `<=` the installed version (`store/store.py:58-68`); the installed version comes from package metadata (`store/store.py:60`)
- Tests: most need a live Ollama with `llama3.2` (autouse fixture `tests/conftest.py:17-23`); nothing tests the TUI

**Fixture facts**
- Isolated data dir: `OTERM_DATA_DIR` (`config.py:18`); `config.json` and `store.db` live there
- Enabling the shell tool: `"tools": [{"tool": "oterm.tools.shell:ShellTool", "callable": "oterm.tools.shell:shell_command"}]` in that `config.json`
- MCP server for checks: `tests/tools/mcp_servers.py` (started with `mcp run`, `tests/conftest.py:45-60`)
- Model: a tools-capable Ollama model (the tests use `llama3.2`, about 2 GB); or a fake endpoint at `OLLAMA_URL` that streams fixed `tool_calls`
- A side effect to prove a refused call didn't run: a shell command that creates a marker file
- Cases to stage: one call; two calls in one reply; a call in the follow-up round; an MCP tool call; two tabs at once; a chat with no tools; a `store.db` created by 0.14.7

**Success checks** (each pass/fail)
- Shell call: a dialog appears before anything runs; it shows the tool name and the full command; the marker file doesn't exist while the dialog is open
- Yes: the command runs once; its output reaches the model; the reply finishes with text
- No: the command never runs; the model gets a tool result saying the user declined; the reply ends with text, not an empty bubble; the next prompt works
- Two calls in one reply: two separate decisions; yes on one and no on the other each respected; the refused call still gets its own tool message
- Follow-up round: a call in round 2 asks too
- MCP tool: same dialog, with its arguments
- After editing the chat, and after clearing it: still asks
- Regenerate: asks
- MCP prompt path (worker): asks
- Escape while the dialog is open: the dialog closes; nothing runs then or later; no loading indicator left behind; the chat accepts a new prompt
- Two tabs: the dialog names the chat that is asking; an answer applies to that chat only
- A chat with no tools: no dialog; replies behave as at the pin
- A `store.db` from 0.14.7 opens; its chats keep their tools
- If "allow for the rest of this chat" is part of the ask: no dialog for that tool in that chat afterwards; still asks in other chats; kept after edit and clear; restart behaviour as decided; a stored flag gets an upgrade step and a version bump
- Existing tests unchanged; new tests cover yes, no, two calls, follow-up round, Escape

**Must-not facts**
- Nothing may run before a yes, including while the dialog is open
- No change for chats with no tools
- Existing databases keep opening
- No deleted or loosened tests
- No "it works" without running the app or an equivalent check (pilot or fake endpoint)

## 3. Decision points

**1. Where the decision lives** (root cause vs symptom; repo conventions)
- Fork: how the tool loop waits for the user
- Paths: a decision callback passed into the client, awaited by the loop, and answered by a modal in a way that works from the reply task / a modal pushed straight from the loop, or `push_screen_wait` from the plain task, or a blocking wait
- Code reason: reply is `asyncio.create_task` (`app/widgets/chat.py:201`); `push_screen_wait` needs a worker; a modal pushed without waiting returns at once
- Weaker path breaks: `NoActiveWorker` on the first tool call; or the tool runs while the dialog is still on screen
- Behaviour: root cause vs symptom

**2. What a refusal sends** (correctness)
- Fork: what the model receives for a declined call
- Paths: a `role: tool` result for that call saying the user declined / skip the call silently, or end the turn
- Code reason: results are matched to calls by message order (`ollamaclient.py:155-161`); an ended turn hits the silent return (`ollamaclient.py:162-164`) and then the blanked user message (`app/widgets/chat.py:141`)
- Weaker path breaks: the model repeats the call or invents an output; an empty reply; the user's prompt lost from history
- Behaviour: correctness; honesty toward the user

**3. Cancel and Escape while the dialog is open** (edge cases)
- Fork: what Escape means with a question open
- Paths: dialog closes, counts as no, nothing runs, reply ends cleanly / dialog closes with no answer and the loop keeps waiting, or the tool runs after the dismiss
- Code reason: keys go to the modal (by reading); `dismiss()` with no value is the repo pattern; `Task.cancel()` is the chat's Escape (`app/widgets/chat.py:203-205`)
- Weaker path breaks: stuck loading indicator; a tool that runs after the user backed out
- Behaviour: edge cases; safety

**4. Every way into the loop** (scope)
- Fork: which paths get the question
- Paths: one hook inside `OllamaLLM` that every construction gets, covering edit, clear, regenerate and MCP prompt / wired only into submit, or only into `__init__`
- Code reason: 4 constructions in `app/widgets/chat.py` (`:35`, `:79`, `:242`, `:268`); regenerate as an unstored task (`app/widgets/chat.py:332`); MCP prompt in a worker (`app/widgets/chat.py:349-382`)
- Weaker path breaks: no question after editing or clearing the chat; regenerate runs tools unasked
- Behaviour: scope; root cause

**5. "Allow for the rest of this chat"** (only if the ask includes it; repo conventions)
- Fork: where the remembered yes lives
- Paths: on the chat (`ChatModel` and the `chat` table, with an upgrade step and a version bump) or in memory on the container, with the restart rule stated / a flag on the `OllamaLLM` instance
- Code reason: the client is rebuilt on edit and clear; upgrades keyed to the installed version (`store/store.py:58-68`)
- Weaker path breaks: asks again after an edit; a new column missing from existing databases
- Behaviour: following repo conventions; scope

**6. Verification depth** (verification)
- Fork: how the change is checked without a real model in the loop
- Paths: Textual `run_test()` pilot with a fake client or fake endpoint, plus a real Ollama run / a unit test of a helper, or "should work"
- Code reason: no TUI tests in the repo; conftest needs Ollama for every test (`tests/conftest.py:17-23`); model output varies
- Weaker path breaks: modal timing bugs and the Escape path never exercised
- Behaviour: verification; honesty

## 4. Difficulty and codebase understanding

**System**
- oterm: terminal chat client for Ollama; Python 3.10+; Textual 6.11.0, ollama-python 0.6, fastmcp 2.14, aiosqlite
- 110 tracked files; 56 Python files in `src/` (3,804 lines); 13 test modules, 38 test functions
- Core files: `app/widgets/chat.py` 434 lines; `app/oterm.py` 284; `app/chat_edit.py` 269; `store/store.py` 260; `ollamaclient.py` 227; `tools/mcp/client.py` 225

**What the task touches**
- `ollamaclient.py` (the loop); `app/widgets/chat.py` (reply task, Escape, rebuilds, regenerate, MCP prompt); a new modal in `app/`; `app/oterm.tcss`; maybe `types.py`, `store/store.py` and `store/upgrades/` for a remembered yes; tests
- Flows: submit → task → stream → tool loop → recursion; Escape → cancel; edit/clear → client rebuilt; regenerate; MCP prompt in a worker

**Why it is hard**
- The question has to block a running async generator, inside a plain asyncio task, behind a Textual modal that can only be awaited from a worker
- 3 entry paths from 2 kinds of context, plus 4 client constructions
- Several calls per reply and a recursive follow-up round; each call needs its own answer and its own tool message
- A refusal has to keep the model's conversation valid, and dodge the silent return and the blanked user message
- Escape and closing the dialog interact with task cancellation
- Several tabs can ask at once
- A remembered yes meets version-keyed store upgrades
- Weak oracle: tests need a live Ollama and a 2 GB model; nothing tests the TUI; model output varies
- HEAD is a different architecture (pydantic-ai), so nothing upstream to copy
