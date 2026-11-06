# listmonk: codebase context for restarts, reloads and upgrades during a send

Pin `30846f84154659ad0b365a3d7c9fe2231dd0d3c2` (tag `v5.1.0`, 2025-09-09). Released image `listmonk/listmonk:v5.1.0` (amd64, arm64, armv7, armv6). Paths are repo-relative and `:line` is at the pin. "(by reading)" means traced, not run.

## 1. What it is and who runs it
- "a standalone, self-hosted, newsletter and mailing list manager… packed into a single binary. It uses a PostgreSQL database as its data store." (`README.md:5`)
- AGPL-3.0, maintained by one main author since 2018 (first commit 2018-10-25; 1,792 commits up to the pin).
- People self-host it to send newsletters and transactional mail to their own lists, through their own SMTP servers or relays. The admin UI and the REST API live in the same process that sends the mail.

## 2. Repo layout
- 458 tracked files. 82 Go files, about 18,000 lines.
- `cmd/`: the binary. `main.go` (317 lines: startup, signals, shutdown), `init.go` (1,100: config, DB, campaign manager, cron, reload), `campaigns.go` (746: campaign API), `settings.go` (362), `admin.go` (111), `manager_store.go` (152: the manager's DB calls), `upgrade.go` (163), `install.go` (353), `handlers.go` (routes).
- `internal/manager/`: the campaign sender. `manager.go` (665: scan loop, workers, stats), `pipe.go` (240: one running campaign), `message.go`.
- `queries.sql` (1,331 lines, all SQL), `schema.sql` (432, including default settings), `internal/migrations/` (version-keyed upgrades, latest `v5.1.0.go`).
- `frontend/`: the Vue admin UI, embedded into the binary at build time. 11 Cypress specs under `frontend/cypress/e2e/`.
- Deploy files: `Dockerfile`, `docker-entrypoint.sh`, `docker-compose.yml`, `listmonk@.service`, `listmonk-simple.service`, `config.toml.sample`. Dev suite in `dev/`.
- Docs: `docs/docs/content/` (installation, configuration, upgrade, APIs).

## 3. Startup (`cmd/main.go`)
- Connect to Postgres once, with no retry: a failed connect is fatal (`cmd/init.go:320-325`).
- `--install` runs and exits (`cmd/main.go:137-141`). With `--idempotent` it only installs when the `settings` table is missing (`cmd/install.go:45-50`); without it, it wipes the listmonk tables (`cmd/install.go:24-27`).
- No schema: fatal "Run --install" (`cmd/main.go:143-148`). `--upgrade` runs migrations and exits (`:150-153`).
- Pending migrations: fatal, "Backup the database and run listmonk --upgrade" (`cmd/upgrade.go:96-112`, called at `cmd/main.go:155-156`).
- Settings are loaded from the DB at start (`cmd/main.go:161-164`). Most settings live in the DB, not in the config file.
- Started in the background: the bounce processor if enabled (`cmd/main.go:230-232`), the slow-query cron if `app.cache_slow_queries` (`:235-237`), the campaign manager (`:241`), the update checker every 24 h if `app.check_updates` (`:283-285`).
- The update checker calls `https://update.listmonk.app/update.json` (`cmd/updates.go:13`, `:39-44`). `app.check_updates` defaults to true (`schema.sql:243`).
- First run with no users: the login page shows a setup form for the first admin until one exists (`cmd/auth.go:45-53`, `cmd/main.go:279`). `LISTMONK_ADMIN_USER` and `LISTMONK_ADMIN_PASSWORD` create it during `--install` instead (`cmd/install.go:78-96`). `LISTMONK_ADMIN_API_USER` also creates an API user and prints its token to stderr (`cmd/install.go:334-351`).

## 4. Signals, reload and shutdown
- The only signal handled is SIGHUP (`cmd/main.go:294`). There's no handler for SIGTERM or SIGINT, so Go's default applies and the process exits at once, with no cleanup (by reading).
- On SIGHUP, `awaitReload` runs the closer, waits for it or 3 s, then re-execs the same binary with the same args (`cmd/init.go:948-978`).
- The closer: HTTP shutdown with a 1 s timeout, `mgr.Close()`, DB close, messengers close (`cmd/main.go:296-316`).
- `mgr.Close()` only closes the pipe queue and the non-campaign queue (`internal/manager/manager.go:395-398`). Workers return as soon as that queue is closed (`internal/manager/manager.go:524-527`), so queued campaign messages are dropped and no campaign cleanup runs (by reading).
- What sends SIGHUP:
  - saving settings with no running campaign, after 500 ms (`cmd/settings.go:283-287`)
  - saving settings with a running campaign only sets `needs_restart` (`cmd/settings.go:271-281`); the UI then says "Settings changed. Pause all running campaigns and restart the app" (`i18n/en.json:488`)
  - the Restart button, `POST /api/admin/reload`, with no check for running campaigns (`cmd/admin.go:101-111`, route `cmd/handlers.go:103`); the UI asks "Ensure running campaigns are paused. Restart?" (`i18n/en.json:403`)
  - an external `kill -HUP`
- `needs_restart` is exposed in `GET /api/config` (`cmd/admin.go:30`, `:70-73`).

## 5. How a campaign is sent (`internal/manager`)
- Config from settings: batch size, concurrency, message rate, max send errors, sliding window; `ScanInterval` is a fixed 5 s, and `--passive` turns scanning off (`cmd/init.go:536-560`, flag at `:167`).
- `Run()` starts the scanner, `Concurrency` workers, and a loop that fetches batches for each running campaign (`internal/manager/manager.go:264-304`).
- Scanner, every 5 s (`internal/manager/manager.go:403-434`):
  - sends the in-memory sent counts to the DB and resets them (`internal/manager/manager.go:539-557`, `queries.sql:730-735`)
  - picks up campaigns that are `running`, or `scheduled` and due, and marks them running (`queries.sql:688-746`)
- One pipe per running campaign (`internal/manager/pipe.go:27-70`). `NextSubscribers` fetches a batch and pushes each message into the campaign queue, blocking when it's full (`internal/manager/pipe.go:76-135`).
- The batch query moves the checkpoint as soon as it fetches: `SET last_subscriber_id = (SELECT MAX(id) FROM subs)` (`queries.sql:850-853`). It reads the old checkpoint without a lock (`cmd/manager_store.go:49-66`, `queries.sql:792-799`).
- Workers (`internal/manager/manager.go:438-535`):
  - skip messages whose pipe is stopped (`internal/manager/manager.go:449-454`)
  - send up to `MessageRate` messages, then sleep a second, per worker (`:456-461`). The UI help says the net rate is concurrency × rate (`i18n/en.json:498`)
  - after a send, track the highest subscriber ID sent and add to `sent` (`internal/manager/manager.go:513-520`)
- Batches are sent by several workers at once, so they finish out of order (by reading).

## 6. Pause, cancel and cleanup
- `PUT /api/campaigns/:id/status` updates the DB first, then calls `StopCampaign` on this process's manager for `paused` or `cancelled` (`cmd/campaigns.go:338-366`).
- `StopCampaign` marks the pipe as stopped, nothing more (`internal/manager/manager.go:386-392`, `internal/manager/pipe.go:157-168`). Workers then drop that campaign's queued messages.
- When the pipe's last message is done, `cleanup` writes `UpdateCampaignCounts(id, 0, sent, lastID)` (`internal/manager/pipe.go:188-198`). That query sets `last_subscriber_id = (CASE WHEN $4 > 0 THEN $4 ELSE to_send END)` (`queries.sql:917-923`):
  - after a pause, the checkpoint goes back to the highest ID actually sent
  - lower IDs that were still queued when the pause landed are skipped on resume, because sends finish out of order (by reading)
  - if nothing was sent by this pipe, the checkpoint becomes `to_send`, a count rather than an ID (by reading)
- A paused pipe logs "stop processing campaign (name)" when it's drained (`internal/manager/pipe.go:213-216`). That's the only sign the drain finished. The last 5,000 log lines are readable through `GET /api/logs` (`cmd/main.go:77`, route `cmd/handlers.go:104`).
- Too many send errors pause the campaign the same way (`internal/manager/pipe.go:139-152`, `:201-210`). `app.max_send_errors` defaults to 1000 (`schema.sql:233`).
- `GET /api/campaigns/running/stats` reads campaigns with status `running` from the DB (`cmd/campaigns.go:421-452`, `queries.sql:679-680`). It goes empty as soon as the status changes, while the drain may still be running.
- Status rules: only `running` campaigns can be paused or cancelled; only `paused` and `draft` ones can start (`docs/docs/content/apis/campaigns.md:395-398`).

## 7. What a hard stop does (by reading)
- The current batch was already checkpointed at fetch time, so its unsent remainder (up to `batch_size`, default 1000, `schema.sql:232`) is skipped on the next start.
- `sent` lags by up to the 5 s since the last sync, so the counter is low after a kill.
- The same happens on a SIGHUP reload during a send, through `mgr.Close()` (section 4).
- Upstream changed this after the pin: commit `a729a4a` (2026-10-16), "Fix campaign resume checkpoints to avoid skipping unsent subscribers", says the batch query advanced the checkpoint before sending, so crashes or pause/resume with out-of-order sends could lose unsent messages (closes issue #3022). It's in `v7.0.0-beta.1` (2026-10-20) and not in any stable release up to `v6.2.0` (2026-06-26).

## 8. Running more than one instance
- Every instance without `--passive` scans and sends (`cmd/init.go:556-560`).
- Two senders read the same checkpoint with no lock, so they can fetch the same batch (`cmd/manager_store.go:49-66`) (by reading).
- `message_rate` and `concurrency` are per instance (`internal/manager/manager.go:456-461`).
- A pause only stops the pipe in the instance that got the request (`cmd/campaigns.go:360-363`). Another instance keeps sending what it has queued (by reading).
- A settings save reloads only the instance that got it (`cmd/settings.go:283-287`).
- The importer runs inside the app process (`cmd/init.go:589-590`). The slow-query cron runs in every instance that has it enabled (`cmd/main.go:235-237`).

## 9. Docker and service files
- `Dockerfile`: `alpine:latest`, copies a prebuilt `listmonk` binary and `config.toml.sample`, `ENTRYPOINT ["docker-entrypoint.sh"]`, `CMD ["./listmonk"]` (`Dockerfile:1-26`).
- `docker-entrypoint.sh`:
  - `PUID`/`PGID` default to 0 (`docker-entrypoint.sh:5-6`)
  - `LISTMONK_*_FILE` variables are read from files, for Docker secrets (`docker-entrypoint.sh:37-60`)
  - as root with a non-zero `PUID` it runs `su-exec ${PUID}:${PGID} "$@"` without `exec`, so the shell stays PID 1; otherwise `exec "$@"` (`docker-entrypoint.sh:71-75`)
- `docker-compose.yml`:
  - `image: listmonk/listmonk:latest`, `restart: unless-stopped`, port `9000:9000` on all interfaces (`docker-compose.yml:12-16`)
  - `depends_on: - db` with no health condition (`docker-compose.yml:20-21`), while the DB has a `pg_isready` healthcheck (`:56-60`)
  - command `sh -c "./listmonk --install --idempotent --yes --config '' && ./listmonk --upgrade --yes --config '' && ./listmonk --config ''"` (`docker-compose.yml:22`). So every start can install or upgrade, and a shell starts listmonk
  - DB credentials `listmonk`/`listmonk` (`docker-compose.yml:4-7`); DB port on `127.0.0.1` (`:51`); `./uploads` mounted (`:41-43`)
  - no healthcheck for the app and no `stop_grace_period`, so Docker's 10 s default applies
- With `sh -c` as the command, whether the final `./listmonk` replaces the shell depends on the shell (by reading, not run). Either way listmonk exits on SIGTERM without draining (section 4).
- `listmonk@.service` runs `--upgrade --yes` before every start (`listmonk@.service:14`), `Restart=on-failure` (`:16`).
- Defaults in `schema.sql`: `app.root_url` `http://localhost:9000` (`schema.sql:226`), concurrency 10, message rate 10, batch size 1000 (`:230-232`), sliding window off (`:234-236`), uploads on the filesystem at `uploads` (`:258-262`).

## 10. Upgrade and restore docs
- "Always take a backup of the Postgres database before upgrading" (`docs/docs/content/upgrade.md:3-4`).
- Binary: stop the instance, replace the binary, `--upgrade`, start (`docs/docs/content/upgrade.md:7-10`). For stopping it suggests `pkill -9 listmonk` (`:14`).
- Docker: `docker compose down app`, `pull`, `up app -d`, relying on the compose command's `--upgrade` (`docs/docs/content/upgrade.md:16-23`).
- Downgrade needs the pre-upgrade database (`docs/docs/content/upgrade.md:41-50`).
- None of these steps mention running campaigns.
- A database dump keeps campaign status and `last_subscriber_id`, so restoring a dump taken mid-send brings back a running campaign at that checkpoint (by reading).

## 11. Build, run and tests
- Go 1.24.1 (`go.mod:3`). In the `Makefile`: `make build`, `make dist` (the binary with the frontend packed in by stuffbin), and `make test`, which runs `go test ./...`.
- There are no Go test files (0 `_test.go`). The 11 Cypress specs drive the admin UI against a running app and DB.
- CI at the pin only builds: "Build Sanity Check" runs `make dist` (`.github/workflows/build-sanity.yml:21-22`).
- Dev suite: `dev/docker-compose.yml` has MailHog `v1.0.1` as an SMTP sink on 1025 with a UI on 8025 (`dev/docker-compose.yml:12-18`), Postgres 13, and backend and frontend containers built from source.
- API auth: BasicAuth `api_user:token` or `Authorization: token api_user:token`; API users are created under Admin → Users (`docs/docs/content/apis/apis.md:10-19`).
- Useful endpoints: `GET /health` and `/api/health` (`cmd/handlers.go:94`, `:267`), `PUT /api/campaigns/:id/status` (`:161`), `GET /api/campaigns/running/stats` (`:150`), `GET /api/logs` (`:104`), `POST /api/import/subscribers` (`:139`), `PUT /api/settings` (`:101`).

## 12. Known traps at the pin
- **SIGTERM mid-send** skips the rest of the fetched batch for good and leaves `sent` low.
- **SIGHUP mid-send** (the Restart button, or a settings save that finds no running campaign in this instance) does the same through `mgr.Close()`.
- **PID 1.** `sh -c`, or the entrypoint's `su-exec` path, can sit between Docker and listmonk. A HUP sent to the container may never reach listmonk, and a stop waits 10 s and kills.
- **Pause isn't instant.** The DB says paused at once; the drain finishes later, and only a log line says so.
- **Pause isn't exact.** Out-of-order sends and the `to_send` fallback can skip subscribers on resume.
- **Two senders** fetch the same batch, double the rate, and ignore each other's pauses. `--passive` exists for extra instances.
- **Rate is per worker.** 10 workers × 10 = 100 messages a second by default.
- **Restoring a dump** taken mid-send resumes the campaign from that point, so people get it twice.
- **Startup** refuses to run with pending migrations, and the shipped command upgrades on every start. A failed migration crash-loops under `restart: unless-stopped`.
- **An empty database** (for example after a failed restore) is silently installed fresh by `--install --idempotent`.
- **The first-admin setup page** is open to anyone who reaches port 9000 until an admin exists.
- **`root_url`** defaults to localhost, so links in sent mail point to localhost until it's set.
- **The update checker** makes a live outbound call every 24 h.
- **Uploads** are on the container filesystem unless the volume and the media setting match.
- **No DB connect retry**, and `depends_on` doesn't wait for Postgres health.

## 13. Pin vs master
- Master `91b76ec` (2026-11-06) splits `queries.sql` into `queries/*.sql`.
- `a729a4a` (2026-10-16) moves checkpointing out of the batch query; `update-campaign-counts` becomes `last_subscriber_id = GREATEST(last_subscriber_id, $4)`. `c09b1ad` the same day fixes a race in the sent-count sync (closes #3031). Both are in `v7.0.0-beta.1` only.

## 14. Unverified items
- Whether the alpine `sh` in the image execs the last command of `sh -c`.
- The exact skip count for out-of-order sends after a pause, and the `to_send` fallback's effect on real IDs.
- How the SMTP pool's `max_msg_retries` (2 by default, `schema.sql:274`) interacts with a stop mid-retry.
