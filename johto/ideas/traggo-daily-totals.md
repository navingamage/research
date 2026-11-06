# Traggo: tracked time per day and per week on the list page

- **Repo:** https://github.com/traggo/server/tree/8ca9bb6aa414ae621488a5a158a6235e89cfa57b
- **Stack:** Go 1.25 (gqlgen GraphQL, gorm 1.9, SQLite) backend; React 16 + TypeScript 3.7 UI built with react-scripts 3, Material-UI 4, FullCalendar 4, moment-timezone, Apollo
- **Domain:** Personal time tracking. Tagged time spans, a list ("daily") page, a calendar page and dashboards.
- **Status:** candidate for a frontend slot. Fava was the other candidate and is kept as backup (below).

## Pre-flight (checked 2026-11-06)
- It's a real application: self-hosted, Docker image, install guide at traggo.net.
- **Size:** 272 tracked files. About 4,500 lines of hand-written Go and 6,800 lines of UI TypeScript.
- **Tests:** 58 Go test files. The UI has exactly one test file (`ui/src/utils/time.test.ts`, the relative-time parser). Nothing renders a component in a test.
- **Clean history:** no `CLAUDE.md`, `AGENTS.md`, `.claude/`, `.cursor` or copilot files. No AI co-authored commits in the whole history.
- **Builds from public sources:** Go modules and the public npm registry (yarn v1 lockfile). `go-sqlite3` needs cgo.
- **Build gotcha:** CI (`.github/workflows/build.yml`) runs the UI build on Node 20 with `NODE_OPTIONS=--openssl-legacy-provider`, because react-scripts 3 uses the old webpack hash. Without that flag `yarn build` and `yarn start` fail on Node 17+. Dev mode is `yarn start` in `ui/` with a proxy to the Go server on 3030.
- **Pin:** HEAD of master, 2026-09-27 (`8ca9bb6a`). The last commit is a timezone fix in the calendar range query (#248, fixes #189).

## How time is stored and shown
- A span is saved twice (`model/timespan.go`): `StartUTC`/`EndUTC`, plus `StartUserTime`/`EndUserTime` as the wall clock at recording time with `OffsetUTC`. The API returns ISO strings with the recorded offset (`timespan/convert.go`).
- The list page (`ui/src/timespan/DoneTrackers.tsx`) loads finished spans 30 at a time by infinite scroll (`TimeSpans` query, `pageSize: 30`, ordered by `start_user_time DESC`). Running timers come from a separate `Trackers` query and render in `ActiveTrackers.tsx` above the list.
- Grouping into day headings is in `ui/src/timespan/timespanutils.ts` (`toGroupedTimeSpanProps`): the key is the **start** day of `oldStart || start`, converted to the browser's timezone by `moment()`. A span is never split across headings.
- Durations on a card are computed in `timeutils.ts` (`timeRunning`) from the two instants, so DST is fine for one span.
- The user's first day of the week is a setting (`model/setting.go`, `FirstDayOfTheWeek`) and the UI pushes it into moment's locale (`ui/src/provider/UserSettingsProvider.tsx`). Date display locale is another setting.
- Dashboards get aggregates from the server (`statistics/summary.go`, `stats2`), bucketed by `start_user_time`/`end_user_time` clipped to each range, using the week-start setting (`time/parse.go`). The dashboard needs a configured dashboard entry per tag key; the list page shows nothing aggregated.

## Gap
- No total of tracked time appears anywhere on the list page: not per day heading, not for the week. `grep -rn "total\|sum\|duration" ui/src/timespan` finds nothing. Same at upstream HEAD on 2026-11-06.
- The calendar page shows spans but no totals either.

## Where a solution can go wrong
- **Spans across midnight.** The list groups by start day. A span from 23:00 to 01:00 puts two hours under the first day unless the total splits at the day boundary. The same applies at the week boundary, which depends on the first-day-of-week setting, not Monday.
- **Running timers.** They are excluded from the list query. A day total that ignores the running timer is stale, and one that includes it has to tick.
- **Partial pages.** Only 30 spans are loaded at a time, so the oldest heading on screen can be missing spans that are still on the server. A total computed from the loaded rows alone is wrong there, and so is a week total built from a partly loaded week. The server-side `stats2` already knows how to clip spans to ranges.
- **Which timezone defines a day.** The server buckets by the wall clock at recording time; the list groups by the browser's current timezone. Spans recorded while travelling land on different days depending on which one is chosen. Dashboards and the list can disagree.
- **DST days.** A 23-hour or 25-hour day is fine as long as totals come from instant differences, not from clock arithmetic.
- **Edited starts.** `oldStart` keeps a span under its original heading after its start was edited. A total that reads `start` and a heading that reads `oldStart` disagree.
- **No UI test harness.** Verifying needs either a new jest test around a pure helper, or starting the UI and looking. The legacy build flag above gets in the way of both.

## Backup: fava
- beancount/fava @ `12d6fea6` (2026-11-03). Svelte 5 + strict TypeScript, 524 files, 23 frontend test files plus `svelte-check` and eslint in CI. No AI files or commits.
- Reasons it is the backup, not the pick: a very active single maintainer closes gaps within weeks, the frontend has typed tests and strict lint that catch most mistakes, and an earlier trial on a frontend issue there was finished cleanly in about 22 minutes.
