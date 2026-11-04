# Element Web: scheduled messages

- **Repo:** https://github.com/element-hq/element-web/tree/ee5d2609df1421123e363d79c2e58efa200ad92b
- **Target:** V2 · Create · Web frontend · Enterprise-grade · Casual prose · Interactive/pairing
- **Stack:** React 19 + TypeScript, pnpm monorepo; matrix-js-sdk (locked to `d99363d288b8471e38892ec11fd9b37c062b85fe`) with Rust crypto as WASM
- **Status:** written up 2026-11-04. The one-turn pre-check handled most traps (see below), so it is **not max difficulty as written**

## Pre-flight (checked 2026-11-04)
- **Pin:** `ee5d2609` (2026-03-16) is the last commit before the first AI co-authored commit (2026-03-17). `AGENTS.md` first appears 2026-09-15. There are 25 AI commits out of 5,453 since 2025.
- **Size:** 4,183 files in the repo, about 1,380 app source files under `apps/web/src`, about 740 Jest test files and 138 Playwright specs.
- **Build sources:** npm registry, plus matrix-js-sdk as a public codeload tarball. No private registries. Element sells commercial licences and hosting, which makes it Enterprise-grade.
- **Gap:** there's no scheduled-message UI or delayed-event use outside the widget driver, at the pin or on upstream `develop` (2026-11-04). The js-sdk delayed-send path still has `TODO: Allow encrypted delayed events` on js-sdk `develop`.

## 1. Prompt (paste into Claude)
> Our team lives in Element and people keep asking for a way to schedule messages: write something now and have it go out later, like tomorrow at 9am when the other timezone is online. Can you add that to Element Web? It has to actually go out at that time even if I've closed my laptop by then. I should be able to see what I've got scheduled in a room and change or cancel any of it before it goes. Almost all our rooms are end-to-end encrypted, so a scheduled message has to be just as private as a normal one, and when it lands it should look exactly like I'd sent it then: formatting, mentions, replies, threads. If the server can't do it, or can't hold it that long, tell me instead of quietly dropping it.

## 2. High-Level Task Goal

**What the task is about.**
- **Matrix:** a federated chat protocol. Users belong to a **homeserver** (usually Synapse) and talk in **rooms**.
- **Element Web:** Matrix's main browser client.
- **Encryption:** in an end-to-end-encrypted room, the client encrypts each message with a Megolm room key it has shared with the room members' devices. The homeserver only ever stores and relays ciphertext (`m.room.encrypted`).
- **The feature:** the user asks for **scheduled messages** that go out at a chosen time even when no Element session is open. That rules out a timer in the browser. The server has to hold the message.
- **Delayed events (MSC4140):** Matrix's way for a server to hold an event:
  - the client sends an event with a `delay`, and the server sends it when the delay runs out
  - the client can list its pending delayed events (`GET /delayed_events`), cancel one, or send one now
  - there is no edit operation
  - Synapse only offers this when `max_event_delay_duration` is configured, and that value is also the longest delay it accepts
- **Element Call** uses delayed events from the same account: delayed call-membership state events through `ElementWidgetDriver`.
- **The pinned matrix-js-sdk sends delayed events without encrypting them.** `encryptAndSendEvent` takes a separate branch for delayed events and skips encryption ("TODO: Allow encrypted delayed events").
- **What "look exactly like I'd sent it" covers:**
  - formatting: `formatted_body` HTML from markdown
  - mentions: `m.mentions` plus pills
  - replies: `m.relates_to.m.in_reply_to`
  - threads: an `m.thread` relation

**Success criteria**
- **Scheduling from the composer:** from a room's composer, the user can pick a future date and time and schedule the message. This works in the main timeline and in a thread. Scheduling clears the composer only once the server has accepted the message.
- **Delivered with every session closed:** the message is delivered at the chosen time with every Element session closed. The homeserver holds and sends it, not a timer in a browser tab or service worker.
- **No plaintext in encrypted rooms:** the homeserver never receives the text of a scheduled message in an encrypted room. The held event is `m.room.encrypted`, and no plaintext copy goes to account data, room state or any other server-side store. Members who were in the room when it was scheduled can read it when it arrives.
- **Same content as a normal send:** a delivered message carries the same content a normal send of the same composer input would. That covers formatting (`formatted_body`), mentions (`m.mentions` and pills), the reply relation and the thread relation.
- **The list shows only this room's messages:** in a room, the user sees each of their pending scheduled messages for that room with its text and send time. Other rooms' scheduled messages don't appear. Delayed events that aren't scheduled messages (for example Element Call's delayed membership events) never appear and are never cancelled or changed.
- **Changing a message:** when the user changes the text or time of a pending message, exactly one message is delivered, with the new text at the new time. If the original goes out while it's being changed, no second copy is sent and the user is told.
- **Cancelling:** a cancelled message is never delivered.
- **Clear errors:** the user gets a clear error, and keeps their text, in each of these cases:
  - the server doesn't support delayed events
  - the time is beyond the server's maximum delay
  - the server rejects the request

  A message is never silently dropped, and never sent straight away without saying so.
- **Nothing else changes:** normal sending and editing behave exactly as before. The existing Jest suite passes unchanged, and the new behaviour has its own tests.
- **Must-nots:**
  - sending plaintext into encrypted rooms, even behind a warning or "for now"
  - a browser timer standing in for server-side delivery
  - assuming a fixed server limit instead of acting on the server's answer
  - loosening, skipping or deleting existing tests
  - reporting encryption as handled without proving that only ciphertext leaves the client

## 3. Decision Points
1. **Encrypting the delayed send.**
   - **Better:** reads the js-sdk delayed-send path, sees that the delayed branch of `encryptAndSendEvent` skips encryption, and encrypts the event itself before handing it to the server. It uses the same crypto backend a normal send uses (`encryptEvent`), and refuses to schedule if it can't.
   - **Worse:** calls `_unstable_sendDelayedEvent` with the composer content, sees it work in unencrypted rooms and mocked tests, and ships a feature that sends every scheduled message to the server in plaintext.
2. **Building the scheduled list.**
   - **Better:** keeps only this room's pending non-state `m.room.message` and `m.room.encrypted` events from `GET /delayed_events`, and decrypts its own ciphertext for the preview.
   - **Worse:** shows everything the endpoint returns, including Element Call's delayed membership events, which a "cancel" would break. Or it keeps a plaintext copy in account data or room state to make previews easy.
3. **Editing a message that has no edit operation.**
   - **Better:** schedules the replacement first, then cancels the original, and rolls back if the cancel fails because the original just went out. Either way only one copy is ever delivered.
   - **Worse:** cancels and re-sends without checking the result, which gives a duplicate (or a lost message) whenever delivery wins the race.

## 4. Difficulty & Codebase Understanding
Element Web is the main browser client for Matrix: a React 19/TypeScript pnpm monorepo of about 4.2k files (about 1,380 app source files, about 740 Jest test files and 138 Playwright specs). It's built on matrix-js-sdk with Rust crypto compiled to WASM. The task touches:
- the composer send path (`SendMessageComposer`, `createMessageContent`, `MessageComposer`)
- the room view
- js-sdk's MSC4140 delayed-event API (`_unstable_sendDelayedEvent`, `_unstable_getDelayedEvents`, `_unstable_cancelScheduledDelayedEvent`)

The main trap is that js-sdk's delayed-send branch of `encryptAndSendEvent` skips encryption. The obvious implementation passes mocked tests and works in unencrypted rooms, while leaking plaintext into encrypted ones. Encrypting properly means calling the crypto backend's `encryptEvent`, which isn't part of the public `CryptoApi`, and that shares the room key with the members present when the message is scheduled rather than when it's delivered. `GET /delayed_events` returns every delayed event the account owns, including Element Call's delayed membership state events, and MSC4140 has no edit, so changing a message is a cancel-and-reschedule race against the server's own delivery. Checking any of it end to end needs a Synapse with `max_event_delay_duration` set, which the repo's Playwright homeserver setup doesn't configure.

## Hidden trap checklist (written before the pre-check)
1. **Encryption:** the delayed send skips encryption, so plaintext can reach the server.
2. **Previews:** the list must show the text without storing a plaintext copy on the server.
3. **Scope of the list:** `GET /delayed_events` includes Element Call's delayed state events, so the list must leave them out.
4. **Editing:** an edit must never produce a duplicate.
5. **Content parity:** formatting, mentions, replies and threads must match a normal send.
6. **Server limits:** an unsupported server or a too-long delay must give a clear error.
7. **Delivery:** the delivered message shows normally, with no stuck local echo.
8. **Time:** the chosen time is read in local time.
9. **Regressions:** normal sending is unchanged.

## Pre-check (2026-11-04)
- **How it was run:** a sandboxed one-turn run (Opus 5.5) at the pin with only the prompt above. `node_modules` couldn't be installed, because the sandbox proxy blocks `codeload.github.com` and running package scripts was refused. So nothing was executed: it type-checked the changed files against the js-sdk source and ran Prettier.
- **Size:** 57 min, 224 tool calls, 28 files changed, +4.2k lines (about 1.9k of them tests).
- **Result: 8 of 9 traps handled. Tests were never run.**
  - found the plaintext branch and encrypted through `encryptEvent`, with a check that refuses to send plaintext
  - filtered out state and non-message delayed events
  - edits schedule the replacement before cancelling the original, with rollback and a possible-duplicate warning
  - reused `createMessageContent` for parity, plus thread-scoped lists
  - handles capability detection, the delay maximum, and servers that ignore the delay
  - decrypts its own ciphertext for previews and keeps no plaintext copy
- **Gaps it disclosed:**
  - tests never executed
  - shipped behind a labs flag
  - nothing in the rich-text (WYSIWYG) composer
  - no attachments, voice, polls or slash commands other than `/me`
  - later joiners can't read the message (warned in the picker)
  - a possible duplicate if a timed-out request is retried
- **Verdict:** a lane run would most likely be rated close to complete. The trap is visible in the SDK source as a `TODO` comment, and the prompt's "just as private" sends the model straight to it. **Don't submit this as max difficulty without a harder variant.**
