# jellyfin-web, a sleep timer for the player: facts for the four write-ups

Repo: https://github.com/jellyfin/jellyfin-web/tree/0a580ce8df89282dfd120b6d5efb8e106da23bd5

Facts, numbers and open choices only, as fragments. The write-ups themselves get written separately.

Pin `0a580ce8df89282dfd120b6d5efb8e106da23bd5` (tag `v12.4`, 2026-10-26); master `c2679743` (2026-11-03) for comparison. Server `jellyfin/jellyfin:12.4`. Paths are under `src/` unless they're root files, and `:line` is at the pin. More detail in `codebase.md`. "(by reading)" = traced, not run.

## 1. The ask

**Domain context facts**
- Sleep timer: stop playback after a set time, or at the end of the current episode or track
- Common in podcast, audiobook, music and video apps
- Usual shapes: preset lengths (15, 30, 45, 60 min); "end of this episode"; a visible countdown; cancel or extend; often a short fade before the stop
- Why people want it: falling asleep to a show or music; otherwise a queue autoplays for hours and the resume point ends up far past where they fell asleep

**jellyfin-web facts behind the ask**
- Episodes autoplay one after another when the user's server setting is on; music queues always continue (`components/playback/playbackmanager.js:3597-3610`)
- Closest existing feature: "Are you still watching?" before the next episode, after 2 to 8 episodes or 1 to 4 hours, episodes only, off unless the user turns it on (`plugins/stillWatching/plugin.ts:16-74`, `plugins/stillWatching/constants.ts:23-29`, `scripts/settings/userSettings.js:586`)
- Video controls: a settings menu with playback speed, quality, repeat mode, aspect ratio, subtitle offset, playback data (`components/playback/playersettingsmenu.js:178-263`)
- Audio controls: the now playing bar (`components/nowPlayingBar/nowPlayingBar.js:51-110`) and the Now Playing page (`apps/legacy/controllers/playback/queue/index.html:33-190`)

**The gap**
- No sleep timer anywhere in `src/` at the pin or at master; no commit message mentions one (checked 2026-11-06)
- No way to say "stop in 30 minutes" or "stop after this episode"
- The still-watching prompt only asks between episodes, never during one, and never for music

**What a timer can act on**
- `playbackManager.stop()`, `pause()` (`components/playback/playbackmanager.js:4066-4118`)
- Player kinds: local video, local audio, remote control of another device, Chromecast, a SyncPlay group

## 2. Goal and success checks

**Repo context a domain expert needs**
- Two stacks side by side: legacy JavaScript (player, OSD, now playing bar) and React/TypeScript (modern layout, dashboard)
- New files must be TypeScript; tests "when possible"; legacy view code "is deemed untestable" (`CONTRIBUTING.md:13`, `:16`)
- Strings: only `strings/en-us.json` is edited (`CONTRIBUTING.md:24`)
- Layouts: modern (auto on desktop and mobile) and legacy (`desktop-legacy`, `mobile-legacy`, `tv`) (`constants/layoutMode.ts:20-24`); the modern app has no TV layout (`CONTRIBUTING.md:81`)
- Both layouts use the same legacy OSD for video (`apps/modern/routes/video/index.tsx:16-88`)
- Stop vs pause: stop reports the stop and the position to the server (`components/playback/playbackmanager.js:3571`) and leaves the video page when nothing video is next (`apps/legacy/controllers/playback/video/index.js:554-563`); pause keeps the session open
- Plays before playing: `PreplayIntercept` plugins can refuse the next item (`components/playback/playbackmanager.js:2458-2491`)
- Video saves every volume change as the user's volume (`plugins/htmlVideoPlayer/plugin.js:1055-1062`); audio already fades on stop (`plugins/htmlAudioPlayer/plugin.js:16-37`)
- Tests: 12 files, 164 cases; none on the playback manager, players or OSD (`vite.config.ts:6-14`)

**Fixture facts**
- Server: `jellyfin/jellyfin:12.4`, port 8096, `/config` and `/media` volumes; the first visit runs the setup wizard
- Libraries: a Shows library with generated episodes of 10 minutes or more (the up-next overlay needs that, `apps/legacy/controllers/playback/video/index.js:660-675`), and a Music library with a few short generated tracks
- Client: `npm ci` (Node 24, npm 11, `package.json:174-177`), `npm start`, then add the server URL on the server selection page (by reading)
- Autoplay: the user's "play next episode automatically" setting on the server (`EnableNextEpisodeAutoPlay`)
- Up-next overlay: the user's next-video-info setting (`apps/legacy/controllers/playback/video/index.js:660-675`)
- Short timers for checks: a 1-minute option, or a dev-only override; Vitest fake timers for the countdown logic
- Remote control: a second browser signed in to the same server is a target device
- SyncPlay: two browser profiles in one group
- Chromecast: needs a real device; can't be checked in a container
- Cases to stage: timer during a video; end of episode with autoplay on; end of track with Repeat One; music while moving between pages; the tab hidden when the timer ends; a fade, then the next play's volume; remote control; a SyncPlay group of two; TV layout with arrow keys; modern and legacy layouts; no timer set

**Success checks** (each pass/fail)
- Setting it: reachable from video and from music, in the modern and the legacy layout; in the TV layout with arrow keys only
- Fixed time: playback stops (or pauses, as decided) within a few seconds of the deadline, including with the tab hidden
- Resume point: the item shows the stop position as its resume point afterwards
- End of episode with autoplay on: the episode plays to its end; the next one doesn't start; the up-next overlay doesn't promise it; no empty video page left behind
- End of track with Repeat One: stops after one play
- Music: the countdown keeps going while moving between pages; the queue stops
- Visible state: remaining time shown; cancel works; a new timer replaces the old one
- After a fade, if there is one: the next item starts at the volume from before the timer
- Remote control and cast: behaviour as decided and shown to the user
- SyncPlay: only this device stops (or as decided); the others keep playing
- Player change while a timer runs: as decided (follows or cancels)
- No timer set: playback, up-next and still-watching behave as at the pin
- New files in TypeScript; strings in `en-us.json` only
- `npm run lint`, `stylelint`, `build:check`, `test` and `build:es-check` pass (`.github/workflows/__quality_checks.yml:36-41`); new tests cover the countdown

**Must-not facts**
- No server changes and no new server API use
- No change to playback when no timer is set
- No edits to other languages' string files
- No new files in `src/scripts` (`CONTRIBUTING.md:123`, `:131`)
- No deleted or loosened tests
- No "it works" without running it against a real server

## 3. Decision points

**1. Where the timer lives** (root cause vs symptom; repo conventions)
- Fork: what owns the countdown
- Paths: a module or plugin outside any page that listens to `playbackManager` / state kept in the OSD controller, the now playing bar, or a React component
- Code reason: leaving the OSD page stops playback and destroys the view (`apps/legacy/controllers/playback/video/index.js:1591-1600`); music keeps playing across pages; the modern video page is a React wrapper over the legacy view (`apps/modern/routes/video/index.tsx:16-88`); stillWatching is a plugin listed in `config.json:34-50`
- Weaker path breaks: the timer is lost when you move pages while music plays; it only exists in one layout
- Behaviour: root cause vs symptom; following repo conventions

**2. How "end of episode" stops the queue** (correctness)
- Fork: how the next item is kept from playing
- Paths: refuse the next item before it plays (a `PreplayIntercept`, like stillWatching) and close the video page / react to `playbackstop`, flip `_playNextAfterEnded`, or stop a second before the end
- Code reason: `nextTrack()` runs after an await (`components/playback/playbackmanager.js:3597-3610`); the flag is reset on every start (`:3384`); Repeat One returns the same item (`components/playback/playqueuemanager.js:236-266`); a refused item cancels playback and nothing listens for `playbackcancelled` (`components/playback/playbackmanager.js:2419-2434`)
- Weaker path breaks: the next episode starts anyway; Repeat One never stops; the episode is cut a second short; an empty video page stays open (by reading)
- Behaviour: correctness; edge cases

**3. Fade and volume** (edge cases)
- Fork: whether to fade, and how
- Paths: no fade, or a fade that doesn't save each step, restores the volume and is skipped where volume can't be set / a fade through `setVolume`
- Code reason: video saves each volume change (`plugins/htmlVideoPlayer/plugin.js:1055-1062`); audio's own fade has a guard (`plugins/htmlAudioPlayer/plugin.js:16-37`, `:324-343`); volume calls do nothing with physical volume control (`components/playback/playbackmanager.js:1183-1215`); iOS Safari ignores volume (`plugins/htmlAudioPlayer/plugin.js:20-21`)
- Weaker path breaks: the next video starts almost silent; a fade that does nothing on iOS
- Behaviour: edge cases; reuse of existing code

**4. Remote, cast and SyncPlay** (scope; safety)
- Fork: which players the timer applies to, and what it does there
- Paths: decide per player kind and say so in the UI (local only, or remote with "keep this tab open"); in SyncPlay stop this device only / call `pause()` everywhere
- Code reason: remote and cast players aren't local (`plugins/sessionPlayer/plugin.js:273`, `plugins/chromecastPlayer/plugin.js:609`); SyncPlay swaps `pause` for a group request but not `stop` (`plugins/syncPlay/ui/players/NoActivePlayer.js:26-79`, `:222-224`); `haltGroupPlayback` exists (`plugins/syncPlay/core/Manager.js:341-348`)
- Weaker path breaks: one person's timer pauses the whole group; a timer for a TV that never fires because the phone's tab closed
- Behaviour: scope; safety; honesty toward the user

**5. The clock** (correctness)
- Fork: how time is measured
- Paths: a wall-clock deadline checked on each tick / counting interval ticks, or one long `setTimeout`
- Code reason: hidden tabs throttle timers (`codebase.md` section 14); devices sleep; the repo has no timer helper, and progress reports use a plain 10 s interval (`components/playback/playbackmanager.js:3348-3352`)
- Weaker path breaks: fires minutes late in a background tab
- Open choice: does the countdown run while paused
- Behaviour: correctness

**6. Where it shows up** (completeness; repo conventions)
- Fork: entry points and visible state
- Paths: video and music entry points with remaining time and cancel, in both layouts and TV, strings in `en-us.json` / the video menu only, or a new MUI control in the modern layout only
- Code reason: the OSD settings menu is shared by both layouts (`components/playback/playersettingsmenu.js:178-263`); the now playing bar is hidden on TV (`components/nowPlayingBar/nowPlayingBar.js:644-652`); the up-next text counts down regardless (`components/upnextdialog/upnextdialog.js:48-71`)
- Weaker path breaks: no timer on TV or for music; the overlay says the next episode is coming when it isn't
- Behaviour: completeness; following repo conventions

**7. Verification depth** (verification)
- Fork: how the change is checked
- Paths: run against a 12.4 server with generated media, short timers, both modes, a hidden tab, plus Vitest tests with fake timers / unit tests alone, or lint and `tsc`
- Code reason: no tests touch playback; legacy views "deemed untestable" (`CONTRIBUTING.md:16`); the behaviour comes from real media events
- Weaker path breaks: the autoplay race, the saved volume and the hidden-tab delay are never seen
- Behaviour: verification; honesty

## 4. Difficulty and codebase understanding

**System**
- jellyfin-web 12.4: 1,225 tracked files; about 72,500 lines of JS and 46,600 of TS in `src/`
- React 18, MUI 6, TanStack Query 5, TypeScript 5.9, webpack 5, Vitest 3; Node 24
- Built for old TV browsers too: ES5 check on the bundle (`.escheckrc`)
- Core files: `components/playback/playbackmanager.js` 4,433 lines; `plugins/htmlVideoPlayer/plugin.js` 2,419; `apps/legacy/controllers/playback/video/index.js` 2,069; `components/remotecontrol/remotecontrol.js` 939; `components/nowPlayingBar/nowPlayingBar.js` 767

**What the task touches**
- `playbackManager` events and stop/pause; a new TypeScript module or plugin (`config.json:34-50`); the OSD settings menu; the now playing bar and Now Playing page; the up-next overlay; maybe user settings (`scripts/settings/userSettings.js:105-121`); `strings/en-us.json`; tests
- Flows: set → count down → stop or pause → report to the server → UI closes; end of episode → next item refused → video page closes; moving between pages; player change

**Why it is hard**
- The player is legacy JavaScript wired by events; new files have to be TypeScript
- The next item starts after an await, so "end of episode" races unless the next item is refused before it plays
- Each player kind behaves differently: local video saves volume changes, local audio fades on its own, remote and cast play elsewhere, SyncPlay turns pause into a group request
- Two layouts plus TV, and music has different controls from video
- Hidden tabs throttle timers
- Weak oracle: 12 test files, none on the playback manager, the players or the controls; seeing anything needs a running server and real media
- Nothing upstream to copy
