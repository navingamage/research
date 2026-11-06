# listmonk, restarts and upgrades during a send: facts for the four write-ups

Facts, numbers and open choices only, as fragments. The write-ups themselves get written separately.

Pin `30846f84154659ad0b365a3d7c9fe2231dd0d3c2` (tag `v5.1.0`, 2025-09-09); master `91b76ec` (2026-11-06) for comparison. Image `listmonk/listmonk:v5.1.0`. Paths are repo-relative and `:line` is at the pin. More detail in `codebase.md`. "(by reading)" = traced, not run.

## 1. The ask

**Domain context facts**
- listmonk: self-hosted newsletter and mailing list manager; one Go binary plus Postgres (`README.md:5`)
- A campaign send runs inside the app process: batches of subscribers from Postgres, worker goroutines pushing mail to SMTP
- Operating it: deploys, restarts, upgrades, settings changes, backups and restores, often while a campaign is going out to thousands of people
- What operators expect: a restart or upgrade mid-send neither skips anyone nor sends anyone twice, and the sent count stays right
- Usual shutdown contract in containers: SIGTERM, then a grace period (Docker's default is 10 s), then SIGKILL

**listmonk facts behind the ask**
- Only SIGHUP is handled; no SIGTERM handler (`cmd/main.go:294`)
- The batch query moves the checkpoint when it fetches, before anything is sent (`queries.sql:850-853`)
- A batch is up to `batch_size`, 1000 by default (`schema.sql:232`)
- The sent count reaches the DB every 5 s (`internal/manager/manager.go:403-434`, `:539-557`)
- Saving settings during a send doesn't reload, and the UI says "Pause all running campaigns and restart the app" (`cmd/settings.go:271-281`, `i18n/en.json:488`)
- The Restart button reloads at once, running campaigns or not (`cmd/admin.go:101-111`)
- Upgrade docs: back up, stop (`pkill -9` for a binary install), `docker compose down/pull/up`; nothing about running campaigns (`docs/docs/content/upgrade.md:3-23`)
- Shipped compose: `latest` tag, a `sh -c` chain that installs and upgrades on every start, no app healthcheck, `depends_on` without a health condition, port 9000 on all interfaces, DB credentials `listmonk`/`listmonk` (`docker-compose.yml:4-22`)
- Upstream confirmed the skipping after the pin: `a729a4a` (2026-10-16, issue #3022), only in `v7.0.0-beta.1`; every stable release up to `v6.2.0` still moves the checkpoint at fetch time

**The gap**
- No supported way at the pin to restart, upgrade or change settings during a send without losing or repeating messages
- No API that says a paused campaign has finished draining; only a log line (`internal/manager/pipe.go:213-216`), readable through `GET /api/logs` (`cmd/handlers.go:104`)

**Ways a send gets interrupted**
- `docker stop`, `docker compose down`, a host reboot: SIGTERM, then SIGKILL
- A settings save when this instance has no running campaign: SIGHUP after 500 ms (`cmd/settings.go:283-287`)
- The Restart button or `POST /api/admin/reload`: SIGHUP (`cmd/handlers.go:103`)
- An upgrade: the app has to stop, and startup refuses pending migrations (`cmd/upgrade.go:96-112`)
- A crash or OOM kill
- A second app container (scaling, a blue-green deploy)
- A restore from a dump taken mid-send

## 2. Goal and success checks

**Repo context a domain expert needs**
- One process serves HTTP and runs the campaign manager, importer, bounce processor and cron (`cmd/main.go:230-241`)
- `--passive`: an instance that serves the UI and API but doesn't send (`cmd/init.go:167`, `:556-560`)
- Campaign columns: `to_send`, `sent`, `last_subscriber_id`, `max_subscriber_id` (`schema.sql:117-118`)
- Pause: the DB status changes first, the in-memory pipe drains later; cleanup writes the checkpoint back to the highest ID sent (`cmd/campaigns.go:338-366`, `internal/manager/pipe.go:188-198`)
- Status rules: only running campaigns can be paused; only paused or draft ones can start (`docs/docs/content/apis/campaigns.md:395-398`)
- Image: alpine plus the released binary; entrypoint handles `PUID`/`PGID` and `*_FILE` secrets (`docker-entrypoint.sh:5-75`)
- Migrations are version-keyed; `--upgrade` is idempotent; startup fails while one is pending (`cmd/upgrade.go:49`, `:96-112`)
- First install: `--install --idempotent`; admin and API users from env vars, the API token printed to stderr (`cmd/install.go:78-96`, `:334-351`)
- API auth: `api_user:token` as BasicAuth or `Authorization: token` (`docs/docs/content/apis/apis.md:10-19`)
- Rate: per worker, so 10 × 10 = 100 messages a second by default (`i18n/en.json:498`, `schema.sql:230-231`)

**Fixture facts**
- Images: `listmonk/listmonk:v5.1.0` (amd64, arm64, armv7, armv6; pushed 2025-09-09) and `postgres:17-alpine` (`docker-compose.yml:47`)
- Mail sink: MailHog `v1.0.1` from the repo's dev suite (`dev/docker-compose.yml:12-18`), or Mailpit; both have an HTTP API to count messages per recipient
- SMTP pointed at the sink with `PUT /api/settings` (`cmd/handlers.go:101`)
- Subscribers: CSV import through `POST /api/import/subscribers` (`cmd/handlers.go:139`) or SQL inserts; enough for several batches (for example 5,000 at batch size 1000)
- Slow sending so an interruption lands mid-batch: concurrency 1 to 2, message rate 5 to 10, or the sliding window (`schema.sql:234-236`)
- Ground truth: the subscriber table vs per-recipient counts in the sink; every subscriber exactly once
- Drain signal: "stop processing campaign" in `GET /api/logs` or the container log
- Cases to stage:
  - restart mid-send
  - upgrade mid-send (tag change plus `--upgrade`)
  - settings change mid-send
  - Restart button mid-send
  - `kill -9`
  - two app containers
  - pause and resume
  - restore from a dump taken mid-send
  - an empty database at start

**Success checks** (each pass/fail)
- Restart mid-send: every subscriber gets exactly one message (sink count = subscriber count, no duplicates, none missing)
- Upgrade mid-send: same
- Settings change mid-send: same; the Restart button either goes through the same procedure or is ruled out in the runbook
- The campaign's `sent` matches the sink at the end
- `docker stop` during a send: drained first, or the runbook keeps stops away from sends, stated either way
- Exactly one instance sends; any extra instance runs `--passive`
- Campaigns the procedure paused are resumed; campaigns someone else paused stay paused
- A backup is taken before every upgrade; a restore never resumes a half-sent campaign without a decision
- The app waits for Postgres health; the app container reports healthy only when `/health` answers (`cmd/handlers.go:267`)
- The image tag is pinned; no `latest`
- No secrets in the compose file; the DB port isn't public; `root_url` is set to the public address (`schema.sql:226`)
- The first-admin setup page isn't reachable from outside before an admin exists
- App code unchanged from v5.1.0: no Go, SQL or frontend edits; the released image as is
- The runbook says what the scripts do, step for step
- The whole check runs from a clean checkout with one command

**Must-not facts**
- No changes to Go code, SQL or the frontend, and no image built from modified sources
- No hand edits to the database outside a documented recovery step
- No mail to real addresses; the sink only
- No deleted or loosened tests (there are no Go tests; the Cypress specs stay as they are)
- No "it works" without a mid-send interruption counted in the sink

## 3. Decision points

**1. Where the safety comes from** (root cause vs symptom; scope)
- Fork: how in-flight messages are protected when the process stops
- Paths: pause running campaigns through the API, wait for each drain, then stop, and resume afterwards / a longer `stop_grace_period`, `init: true`, or trusting Docker's stop
- Code reason: no SIGTERM handler (`cmd/main.go:294`); the checkpoint moves at fetch (`queries.sql:850-853`); cleanup only runs after the pipe drains (`internal/manager/pipe.go:188-216`); `mgr.Close()` drops queued messages (`internal/manager/manager.go:395-398`)
- Weaker path breaks: up to a batch skipped per interruption; a longer grace period changes nothing, because listmonk never drains on SIGTERM
- Behaviour: root cause vs symptom

**2. Knowing the drain is done** (correctness)
- Fork: when it's safe to stop after a pause
- Paths: wait for the "stop processing campaign" line for each paused campaign / a fixed sleep, or the running-stats endpoint going empty
- Code reason: the status changes in the DB at once (`cmd/campaigns.go:354-363`); running stats read the DB (`queries.sql:679-680`); the drain is in memory and only logged (`internal/manager/pipe.go:213-216`); logs over the API (`cmd/main.go:77`, `cmd/handlers.go:104`)
- Weaker path breaks: a stop mid-drain means cleanup never writes the checkpoint back, so the batch is skipped
- Behaviour: correctness

**3. Pause isn't exact** (edge cases; honesty)
- Fork: what the procedure claims and does about the gaps left in pause and resume
- Paths: measure it against the sink, document the residue, and guard the zero-sent case / call pause exact
- Code reason: cleanup sets the checkpoint to the highest ID sent, so lower IDs still queued under out-of-order sends are skipped; with nothing sent it writes `to_send` instead (`internal/manager/pipe.go:196`, `queries.sql:917-923`) (by reading); upstream's fix names pause/resume as well as crashes (`a729a4a`)
- Weaker path breaks: "zero loss" claimed while a few are skipped at concurrency above 1; a campaign paused right after it starts resumes with almost nobody left
- Behaviour: edge cases; honesty

**4. One sender** (scope; safety)
- Fork: what happens with more than one app container
- Paths: one sending instance, every other one `--passive`, enforced in the compose file / scale the app service freely
- Code reason: two instances read the same checkpoint with no lock (`cmd/manager_store.go:49-66`, `queries.sql:792-799`); the rate is per instance (`internal/manager/manager.go:456-461`); a pause only stops the instance that got it (`cmd/campaigns.go:360-363`); a settings save reloads only that instance (`cmd/settings.go:283-287`)
- Weaker path breaks: duplicates; twice the rate against SMTP limits; a paused campaign that keeps going from the other container
- Behaviour: scope; safety

**5. Upgrades and migrations** (reversibility; operations)
- Fork: when `--upgrade` runs
- Paths: an explicit step after a backup, with sends drained and the app stopped / the shipped `--install && --upgrade && run` on every start
- Code reason: shipped command (`docker-compose.yml:22`); startup refuses pending migrations (`cmd/upgrade.go:96-112`); back up first (`docs/docs/content/upgrade.md:3-4`); a downgrade needs the pre-upgrade DB (`docs/docs/content/upgrade.md:41-50`); an empty DB is silently installed fresh (`cmd/install.go:45-50`)
- Weaker path breaks: a migration with no backup; a failed migration crash-looping under `restart: unless-stopped`; a failed restore that comes up as a brand-new empty listmonk
- Behaviour: operations; reversibility

**6. Signals and PID 1** (correctness)
- Fork: which process gets SIGTERM and SIGHUP
- Paths: listmonk as PID 1, or under an init that forwards signals, with reloads done through `docker kill -s HUP` or the API / `sh -c`, or the entrypoint's `su-exec` branch, as PID 1
- Code reason: `sh -c` command (`docker-compose.yml:22`); `su-exec` without `exec` when `PUID` is set (`docker-entrypoint.sh:71-75`); SIGHUP is the reload (`cmd/main.go:294`, `cmd/init.go:948-978`)
- Weaker path breaks: every stop waits 10 s and kills; a HUP to the container never reaches listmonk
- Behaviour: correctness; edge cases

**7. Verification depth** (verification)
- Fork: how "no skips, no duplicates" is shown
- Paths: a scripted run against a mail sink, with several batches, an interruption mid-batch for each case, and per-recipient counts compared with the subscriber table / "the campaign finished", or the `sent` counter
- Code reason: `sent` syncs every 5 s and is low after a kill (`internal/manager/manager.go:539-557`); no Go tests at all; CI only builds (`.github/workflows/build-sanity.yml:21-22`); the Cypress specs don't cover sending
- Weaker path breaks: skipped subscribers never noticed; a campaign marked finished while a batch got nothing
- Behaviour: verification; honesty

## 4. Difficulty and codebase understanding

**System**
- listmonk 5.1.0: Go 1.24 (`go.mod:3`) with Postgres; Vue admin UI packed into the binary; AGPL-3.0
- 458 tracked files; 82 Go files, about 18,000 lines; 0 Go test files; 11 Cypress specs
- Released image: alpine plus a prebuilt binary
- Core files: `cmd/init.go` 1,100 lines; `queries.sql` 1,331; `cmd/campaigns.go` 746; `internal/manager/manager.go` 665; `internal/manager/pipe.go` 240

**What the task touches**
- The compose file, env and secrets, healthchecks, the init and PID 1 setup, scripts (pause, wait for drain, back up, stop, upgrade, start, resume), restore, the runbook, and a check harness with a mail sink
- Flows:
  - send: scan every 5 s → fetch a batch (checkpoint moves) → workers send → sent synced every 5 s
  - pause: DB status → pipe marked stopped → queued messages dropped → cleanup writes the checkpoint → log line
  - reload: SIGHUP → closer → re-exec
  - stop: SIGTERM → exit

**Why it is hard**
- The obvious fix, a graceful shutdown, isn't available without changing the app, so the safety has to come from the procedure around it
- The checkpoint moves before sending, and pause is close to exact but not quite
- Nothing in the API says a drain is finished
- Signals in containers: a shell as PID 1, `su-exec` without `exec`
- Several things trigger a reload, and one ignores running campaigns
- More than one container means duplicates through an unlocked checkpoint
- Proving "exactly once" needs a mail sink and an interruption timed mid-batch
- Weak oracle: no Go tests, CI only builds, and the app's own counters are wrong after a kill
