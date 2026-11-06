# jellyfin-web: codebase context for a player sleep timer

Pin `0a580ce8df89282dfd120b6d5efb8e106da23bd5` (tag `v12.4`, 2026-10-26). Server to run it against: Jellyfin 12.4 (`jellyfin/jellyfin:12.4` on Docker Hub, amd64 and arm64). Paths are under `src/` unless they're root files or start with `.github/`, and `:line` is at the pin. "(by reading)" means traced, not run.

## 1. What it is and who runs it
- The web client for Jellyfin, the self-hosted media server. The server serves it, and the README says it's "the frontend used for most of the clients available for end users, such as desktop browsers, Android, and iOS" (`README.md:34`).
- GPL-2.0-or-later (`package.json:6`). A volunteer project with no hosted product.
- People use it to watch shows and films and to play music and audiobooks from their own server. It can also control another device (remote control), cast to Chromecast, and watch in sync with other people (SyncPlay).

## 2. Repo layout
- 1,225 tracked files, 1,173 of them under `src/`: 316 `.ts`, 289 `.tsx`, 245 `.js`, 111 `.scss`, 110 `.json` (108 of them translation files), 68 `.html`.
- About 72,500 lines of JavaScript and 46,600 lines of TypeScript in `src/`.
- `apps/legacy`: the main app, and the only one with a TV layout. `apps/modern`: the MUI rewrite, which reuses legacy views. `apps/dashboard`: admin. `apps/wizard`: first-run setup.
- Player code:
  - `components/playback/playbackmanager.js` (4,433 lines): the hub every control goes through
  - `components/playback/playqueuemanager.js` (285): the queue and repeat modes
  - `components/htmlMediaHelper.js` (394): shared helpers for the HTML players, including the saved volume
  - `plugins/htmlVideoPlayer/plugin.js` (2,419) and `plugins/htmlAudioPlayer/plugin.js` (620): the local players
  - `plugins/sessionPlayer/plugin.js` (666): remote control of another device; `plugins/chromecastPlayer/plugin.js` (1,118); `plugins/syncPlay/` (group watching)
  - `plugins/stillWatching/plugin.ts` (77): the "Are you still watching?" prompt
- Player UI:
  - `apps/legacy/controllers/playback/video/index.js` (2,069) and `index.html` (102): the video on-screen controls (OSD), used by both layouts
  - `components/playback/playersettingsmenu.js` (294): the OSD settings menu
  - `components/upnextdialog/upnextdialog.js` (261): the "Next episode playing in…" overlay
  - `components/nowPlayingBar/nowPlayingBar.js` (767): the bar at the bottom of the page while audio plays
  - `components/remotecontrol/remotecontrol.js` (939): the Now Playing page (route `queue`)
- Settings: `scripts/settings/userSettings.js` (770), `scripts/settings/appSettings.js` (277). Strings: `strings/en-us.json` (1,949 lines) plus 107 other languages.

## 3. Conventions (`CONTRIBUTING.md`)
- New files must be TypeScript (`CONTRIBUTING.md:13`). API calls go through the TypeScript SDK (`:14`). New pages are React on `components/Page.tsx` (`:15`).
- Unit tests "when possible", and legacy view code "is deemed untestable" (`CONTRIBUTING.md:16`).
- Don't overuse dynamic imports (`CONTRIBUTING.md:18`). Don't reference browser globals; import instead (`:19`).
- Only `en-us` strings are committed; other languages come through Weblate (`CONTRIBUTING.md:24`). Don't rename existing keys (`:25`).
- `src/scripts` is deprecated, so no new files there (`CONTRIBUTING.md:123`, `:131`).
- "ALL media player implementations are plugins" (`CONTRIBUTING.md:94`). The modern app "Does not currently support TV layout" (`:81`); the legacy app does (`:84`).
- The build has to run on old TV browsers: `browserslist` goes back to Chrome 27 and `.escheckrc` checks the bundle is ES5.
- Code style: 4-space indent, single quotes, ESLint flat config (`eslint.config.mjs`) and Stylelint.

## 4. Layouts
- Modes: `auto`, `desktop`, `desktop-legacy`, `modern`, `mobile`, `mobile-legacy`, `tv` (`constants/layoutMode.ts:2-17`). `desktop-legacy`, `mobile-legacy` and `tv` use the legacy app (`constants/layoutMode.ts:20-24`).
- `layoutManager.modern = !isLegacyLayout` (`components/layoutManager.js:41`), so auto on a desktop browser gets the modern app (by reading).
- The video page in the modern layout is an MUI top bar (SyncPlay and remote play buttons) over the legacy OSD view (`apps/modern/routes/video/index.tsx:16-88`). The bottom controls and the settings button are the same legacy code in both layouts.
- The now playing bar hides on the TV layout and while a local video plays (`components/nowPlayingBar/nowPlayingBar.js:644-652`). On TV the audio controls are on the Now Playing page.

## 5. Playback manager (`components/playback/playbackmanager.js`)
- One shared instance, `playbackManager` (`components/playback/playbackmanager.js:4423`). Most methods default to the current player.
- Stop and pause:
  - `stop()` sets `_playNextAfterEnded = false` for local players, then calls `player.stop(true, true)` (`components/playback/playbackmanager.js:4066-4077`)
  - `pause()` and `unpause()` call the player directly (`components/playback/playbackmanager.js:4108-4118`)
- Volume: `setVolume`, `getVolume`, `volumeUp`, `volumeDown` do nothing when the app host has physical volume control (`components/playback/playbackmanager.js:1183-1215`, check at `:50-52`). `getVolume` returns 1 in that case.
- What happens when an item ends (`onPlaybackStopped`, `components/playback/playbackmanager.js:3534-3611`):
  - the next item is taken from the queue only if `_playNextAfterEnded` is still true and there was no error (`components/playback/playbackmanager.js:3550`)
  - the stop and its position go to the server (`reportPlaybackStopped`, `:3571`)
  - `playbackstop` fires on the player and on the manager (`:3580-3581`)
  - after `getCurrentUser()` resolves, `nextTrack()` runs if the user has `EnableNextEpisodeAutoPlay` or the next item isn't a video (`:3597-3610`). So the next item starts after an await, not in the same tick
- `_playNextAfterEnded` starts true (`components/playback/playbackmanager.js:729`) and is set back to true on every start (`:3384`). It's set false by `stop()` (`:4070`), when a new item replaces the playing one (`:2645`) and on app close (`:3814`).
- Before an item plays, `playInternal` runs every `PreplayIntercept` plugin in priority order (`components/playback/playbackmanager.js:2385-2417`, `:2458-2491`). A rejection cancels playback: the player is destroyed and `playbackcancelled` fires (`:2419-2434`). `isFirstItem` is false for items after the first (`:2395`).
- Nothing listens for `playbackcancelled` (only a constant at `apps/legacy/features/playback/constants/playbackManagerEvent.ts:8`).
- Queue: `getNextItemInfo` returns the same index for `RepeatOne` and wraps for `RepeatAll` (`components/playback/playqueuemanager.js:236-266`). `nextTrack` plays it through `playInternal` (`components/playback/playbackmanager.js:3210-3233`).
- Player changes fire `playerchange` (`components/playback/playbackmanager.js:67-77`, set at `:953-982`, cleared at `:945-951`).
- Progress goes to the server on player events and every 10 s (`components/playback/playbackmanager.js:3348-3352`, `:3772-3786`).
- Players are local unless they say otherwise (`components/playback/playbackmanager.js:3723-3725`). Players with their own playlist (`getPlaylist`) manage their own queue (`:42-48`).

## 6. Players
- Local video (`plugins/htmlVideoPlayer/plugin.js`):
  - every volume change is saved as the user's volume (`plugins/htmlVideoPlayer/plugin.js:1055-1062`), and the saved volume is applied when the element is created (`:1835-1838`)
  - `setVolume(0..100)` maps to the element as a cube (`Math.pow(val / 100, 3)`), and `getVolume` takes the cube root (`plugins/htmlVideoPlayer/plugin.js:2223-2235`)
  - `destroy()` removes the video element and its dialog; it doesn't navigate (`plugins/htmlVideoPlayer/plugin.js:963-1000`)
- Local audio (`plugins/htmlAudioPlayer/plugin.js`):
  - an existing fade-out: volume steps down by 0.15 every 100 ms (`plugins/htmlAudioPlayer/plugin.js:16-37`), used by `stop(true)` and then put back to the original volume (`:236-268`)
  - no fade on TV browsers (`plugins/htmlAudioPlayer/plugin.js:47-51`). A comment says iOS Safari ignores volume changes and always reports the system volume (`:20-21`)
  - while fading, volume changes aren't saved and `timeupdate` isn't sent (`plugins/htmlAudioPlayer/plugin.js:324-343`)
- Saved volume (`components/htmlMediaHelper.js:6-14`): read as `appSettings.get('volume') || 1`; `saveVolume` skips falsy values, so 0 is never saved.
- End of media: `onEndedInternal` fires the player's `stopped` event (`components/htmlMediaHelper.js:342-360`), which leads to `onPlaybackStopped`.
- Remote control of another device: `isLocalPlayer = false` (`plugins/sessionPlayer/plugin.js:273`). Stop and pause are commands sent to that device (`plugins/sessionPlayer/plugin.js:390-392`, `:443-445`).
- Chromecast: `isLocalPlayer = false` (`plugins/chromecastPlayer/plugin.js:609`).
- SyncPlay: while in a group, `pause`, `unpause`, `playPause`, `seek`, `nextTrack`, `previousTrack` and the queue calls on `playbackManager` are swapped for group requests (`plugins/syncPlay/ui/players/NoActivePlayer.js:26-79`). `stop` isn't swapped, so it stops this device only (`plugins/syncPlay/ui/players/NoActivePlayer.js:222-224`). The manager has `haltGroupPlayback`, which stops this device and tells the server not to wait for it (`plugins/syncPlay/core/Manager.js:341-348`).

## 7. Video OSD (`apps/legacy/controllers/playback/video/index.js`)
- Settings button (`apps/legacy/controllers/playback/video/index.html:87`) → `playerSettingsMenu.show(...)` (`apps/legacy/controllers/playback/video/index.js:956-984`).
- The settings menu is a list of entries with an `asideText` (aspect ratio, playback speed, quality, repeat mode, subtitle offset, playback data), shown with `actionsheet` (`components/playback/playersettingsmenu.js:178-263`). Each entry opens its own sub-menu (`components/playback/playersettingsmenu.js:265-290`).
- The time line shows "Ends at" (`apps/legacy/controllers/playback/video/index.js:794-835`, markup `apps/legacy/controllers/playback/video/index.html:65-66`).
- Up next:
  - shown for episodes when the user has the next-video overlay on, 30 to 40 s before the end, for items of at least 10 minutes (`apps/legacy/controllers/playback/video/index.js:660-675`)
  - or when a skip segment runs to the end and there's a next item (`apps/legacy/controllers/playback/video/index.js:649-658`)
  - built in `showComingUpNext` (`apps/legacy/controllers/playback/video/index.js:683-697`)
- When playback stops and the next item isn't a video, the OSD goes back (`apps/legacy/controllers/playback/video/index.js:554-563`). Leaving the OSD page stops playback (`apps/legacy/controllers/playback/video/index.js:1591-1600`).
- On `playerchange` the OSD rebinds to the new player, or to none (`apps/legacy/controllers/playback/video/index.js:508-510`, `:581-602`).
- Keyboard shortcuts: `onKeyDown` (`apps/legacy/controllers/playback/video/index.js:1217`).

## 8. Up-next overlay (`components/upnextdialog/upnextdialog.js`)
- The countdown is only text, refreshed every 400 ms (`components/upnextdialog/upnextdialog.js:48-71`, `:203-214`). The next item starts when the current one ends (section 5), not from the overlay.
- "Start now" calls `nextTrack` (`components/upnextdialog/upnextdialog.js:104-114`).
- With autoplay off the text says "Next episode" with no countdown (`components/upnextdialog/upnextdialog.js:216-220`).

## 9. Audio UI
- Now playing bar buttons: previous, play/pause, stop, next, mute, volume, AirPlay, lyrics, repeat, shuffle, favourite, and a "more" menu (`components/nowPlayingBar/nowPlayingBar.js:51-110`).
- It hides when a local player stops and the next item isn't audio (`components/nowPlayingBar/nowPlayingBar.js:612-624`). It lives outside any page, so it stays across navigation.
- Pages can turn it off (`components/Page.tsx:32`); the video page does (`apps/modern/routes/video/index.tsx:85`).
- Now Playing page: `apps/legacy/controllers/playback/queue/index.js:13-14` mounts `RemoteControl`. Its buttons are in `apps/legacy/controllers/playback/queue/index.html:33-190`.

## 10. Still Watching, the closest existing feature
- A `PreplayIntercept` plugin with priority 10, for episodes only (`plugins/stillWatching/plugin.ts:16-74`).
- It counts played items and session time, and uses the time since the last input (`scripts/inputManager.js:19-21`). Over the threshold it shows a confirm with "Stop watching" and "Continue watching" (`plugins/stillWatching/plugin.ts:55-70`).
- Thresholds per option, from 2 episodes or 1 hour to 8 episodes or 4 hours (`plugins/stillWatching/constants.ts:23-29`).
- Its setting is per device: `userSettings.stillWatchingPrompt` uses `set(..., false)` (`scripts/settings/userSettings.js:581-587`). The getter defaults to disabled (`scripts/settings/userSettings.js:586`); the settings page shows "default" when nothing is saved (`components/playbackSettings/playbackSettings.js:154`).
- The settings page hides the field when the plugin isn't loaded (`components/playbackSettings/playbackSettings.js:250-252`). Plugins are listed in `config.json:34-50`.
- The plugin base class is `PreplayInterceptPlugin` (`types/plugin.ts:38-52`).

## 11. Settings storage
- `userSettings.set(name, value, enableOnServer)` writes localStorage through `appSettings`, and also the server's display preferences unless `enableOnServer === false` (`scripts/settings/userSettings.js:105-121`). Server preferences follow the user to other devices.
- `appSettings` is localStorage keyed per user (`scripts/settings/appSettings.js:263-275`).

## 12. Strings
- Keys that already exist: `ButtonCancel` "Cancel" (`strings/en-us.json:113`), `ButtonStop` "Stop" (`:151`), `EndsAtValue` "Ends at {0}" (`:336`), `HeaderSecondsValue` (`:540`), `Off` (`:1306`), `ValueMinutes` "{0} min" (`:1739`).
- There's no sleep timer key in any language.

## 13. Build, run and tests
- Node 24 (`.nvmrc`) and npm 11 or later (`package.json:174-177`). `npm ci`, then:
  - `npm start`: webpack dev server (`package.json:162`), default port 8080
  - `npm run build:development` or `build:production` (`package.json:164-165`)
  - `npm run build:check`: `tsc --noEmit` (`package.json:166`)
  - `npm run lint`, `npm run stylelint` (`package.json:169`, `:172`)
  - `npm test`: Vitest with jsdom (`package.json:170`, `vite.config.ts:6-14`)
- CI runs `build:es-check`, `lint`, `stylelint`, `build:check` and `test` (`.github/workflows/__quality_checks.yml:36-41`).
- 12 test files and 164 test cases. Two are under playback (`apps/legacy/features/playback/utils/itemText.test.ts`, `mediaSegments.test.ts`). Nothing tests the playback manager, the players, the OSD or the now playing bar.
- A server for checks: `jellyfin/jellyfin:12.4` on port 8096 with a `/config` and a `/media` volume. The first visit runs the setup wizard (admin user, libraries).
- In dev, `servers` in `config.json` is empty (`config.json:33`), so the client opens server selection (`/selectserver`, `/addserver`, `components/router/appRouter.js:16-21`) and takes the server URL there (by reading).
- Test media can be generated (an ffmpeg test pattern named like `Show/Season 01/Show S01E01.mkv`, and short tones for music). Episode autoplay is a user setting on the server.

## 14. Timing facts that aren't in the repo
- Hidden tabs throttle timers. Chrome allows at most one wake-up a second in a hidden tab, and after 5 minutes it can batch timers to once a minute for a page that has been silent for 30 seconds (unverified here; from Chrome's timer throttling notes).
- A countdown kept as a deadline (`Date.now()` compared on each tick) stays right under throttling and after the device sleeps. A count of interval ticks doesn't.
- Media playback rate changes how long "the rest of this episode" takes in wall time; "Ends at" already accounts for it (`apps/legacy/controllers/playback/video/index.js:832`).

## 15. Known traps at the pin
- **Where the timer lives.** The OSD controller goes away with the video page, and the modern video page is a React wrapper around it. Music keeps playing across pages, so a timer that lives in a page dies on navigation while the music goes on.
- **End of episode.** The next item starts after an await in `onPlaybackStopped`, so reacting to `playbackstop` races it. `_playNextAfterEnded` is reset on every start. With Repeat One the "next" item is the same one. Stopping just before the end leaves the episode short of the end.
- **Cancelled next item.** With a next video queued, the OSD stays on the page when the episode ends, and nothing listens for `playbackcancelled`. So refusing the next item may leave a video page with no player (by reading, not run).
- **The up-next overlay** still counts down to an episode that a timer will stop.
- **Fade-out on video** goes through `onVolumeChange`, which saves each step as the user's volume, so the next video starts quiet. Audio has the `_isFadingOut` guard; video doesn't. Volume does nothing on iOS Safari and with physical volume control.
- **Remote and cast.** The timer runs in this tab, but the media plays on another device. If the tab closes or the phone sleeps, it never fires.
- **SyncPlay.** `pause` is a group request and pauses everyone. `stop` stops this device only, and the group may wait for it.
- **Player change mid-timer.** Casting or switching devices fires `playerchange`, and the timer has to follow or cancel.
- **Hidden tabs** delay timers (section 14).
- **Two layouts and TV.** The OSD is shared, but the audio entry points differ: on TV the now playing bar is hidden. TV needs arrow-key focus, which `actionsheet` already handles.
- **Live TV and items with no runtime** have no "end of episode".
- **Strings** go in `en-us.json` only.

## 16. Pin vs master
- No sleep timer in `src/` at the pin, or at master `c2679743` (2026-11-03). No commit message in the history mentions one (checked 2026-11-06).
- Master is removing the old `jellyfin-apiclient` package ("rip-apiclient" pull requests), so player code near the API differs after the pin.

## 17. Unverified items
- Whether refusing the next episode leaves the OSD open with no player (section 15).
- Chrome's exact throttling rules for a hidden tab that's playing media.
- Whether the dev server needs anything beyond adding the server URL on first load.
