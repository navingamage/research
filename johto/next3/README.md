# Next 3 picks

Three more ideas, each in a different kind of work: a feature in an AI chat client that runs tools, operating a self-hosted app through restarts and upgrades, and a feature in a large web frontend. All three were vetted on 2026-11-06 ([repo-vetting.md](../repo-vetting.md)).

| # | Task | Repo @ pin | Type | Why it's hard |
|---|---|---|---|---|
| 1 | [Ask before a tool runs](oterm-tool-approval/) | [ggozad/oterm @ `de8604c6`](https://github.com/ggozad/oterm/tree/de8604c6425f5f4777319d7ac228b6d97e47e8a8) (0.14.7) | Feature | The question has to pause the tool loop inside a plain asyncio task, but a Textual modal can only be awaited from a worker. One reply can hold several calls plus a recursive follow-up round, each needing its own answer, and a refusal has to keep the conversation valid instead of ending the turn through the silent return, which also blanks the user's prompt in memory. |
| 2 | [Restarts and upgrades during a send](listmonk-safe-restarts/) | [knadh/listmonk @ `30846f84`](https://github.com/knadh/listmonk/tree/30846f84154659ad0b365a3d7c9fe2231dd0d3c2) (v5.1.0) | Operations | A restart mid-send skips subscribers for good, because the batch query moves the checkpoint before anything is sent and only SIGHUP is handled. With the app unchanged, the safety has to come from the deployment and the procedure: pause isn't exact, only a log line says a drain is done, and a shell can sit between Docker and the app as PID 1. |
| 3 | [Sleep timer for the player](jellyfin-web-sleep-timer/) | [jellyfin/jellyfin-web @ `0a580ce8`](https://github.com/jellyfin/jellyfin-web/tree/0a580ce8df89282dfd120b6d5efb8e106da23bd5) (v12.4) | Feature | The player is event-wired legacy JavaScript, and new files have to be TypeScript. "Stop after this episode" races the next item, which starts after an await, and every player kind behaves differently: video saves each volume change, audio fades on its own, remote and cast play on another device, and SyncPlay turns pause into a group request. There are 12 test files, and none cover the playback manager, the players or the controls. |

## What's in each folder
- `codebase.md` covers how the codebase works around the change, with `file:line` references at the pin. It also has build and test commands, repo conventions, known traps, and a list of what couldn't be confirmed by reading.
- `sections.md` has facts, numbers and open choices for the four write-ups: the ask, the goal and success checks, the decision points, and difficulty and codebase understanding. It's fragments only, and the write-ups themselves get written separately.

## How the notes were checked
- Everything comes from reading the code at the pin. Nothing was built or run for these notes.
- Every path and line reference was checked against the pinned tree: 165 for oterm, 171 for listmonk and 168 for jellyfin-web.
- Anything traced by reading but not run is marked "by reading" or "(unverified)".
- Re-checked on 2026-11-06:
  - oterm HEAD `77335b6` still has no approval step. HEAD has moved to pydantic-ai, so there's nothing upstream to copy.
  - listmonk fixed the checkpoint in `a729a4a` (2026-10-16), which is only in `v7.0.0-beta.1`. Every stable release up to `v6.2.0` still has the skip.
  - jellyfin-web master at `c2679743` (2026-11-03) still has no sleep timer.

## Before a run
- **oterm**
  - Most tests need a running Ollama with `llama3.2` (about 2 GB), which the autouse fixture pulls before every test (`tests/conftest.py:17-23`).
  - The built-in shell tool only shows up once it's listed under `tools` in `config.json`. Setting `OTERM_DATA_DIR` keeps that config and `store.db` apart from any real install.
  - A fake endpoint at `OLLAMA_URL` that streams fixed `tool_calls` can stand in for the model (not tried).
- **listmonk**
  - Run the released image `listmonk/listmonk:v5.1.0` with Postgres and a mail sink, so every message can be counted per subscriber. Docker has to work wherever the run happens.
  - Any check should name `v5.1.0`. The shipped `docker-compose.yml` pulls `latest`, which is a newer release now.
  - There are no Go tests, and CI only builds (`codebase.md` §11).
- **jellyfin-web**
  - It needs a Jellyfin 12.4 server (`jellyfin/jellyfin:12.4`, amd64 and arm64) with a user, a few short generated episodes and some music. Episode autoplay is a user setting on the server.
  - Node 24 and npm 11 or later. `npm ci`, then `npm start` for the dev server and `npm test` for Vitest.
  - In dev, the client opens server selection first, since `config.json` lists no servers (by reading).
