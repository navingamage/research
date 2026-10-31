# Script: designing a fix for lost payment webhooks

**Target length:** about 30 minutes. The limits are 15:00 to 50:59, and the epilogue has to be 3 to 4 minutes (max 4:00).

**How to use it:**
- Each quoted line says what you're doing, and you do it as you say it. Lines in *[brackets]* are silent actions. ⌨️ lines are what you type.
- Say the mouse movement out loud every time: "I'm moving my mouse to...", "I'm clicking...".
- Move slowly, pause on the thing you're naming, and pause about a second after each click.
- If you use a keyboard shortcut, say it out loud first.
- **Always say the exact location**: which heading, which bullet (first, second, third), and in draw.io where on the canvas (left, middle, right, above, below which box). The reviewer has to be able to follow you without seeing the screen.
- Do one full dry run, mainly to practise the draw.io part: dragging shapes, connecting arrows and labelling them.

**draw.io basics (practise these before recording):**
- Shapes are in the panel on the left, under General. Drag one onto the canvas to add it.
- Double-click a shape or an arrow to type its label. Click an empty part of the canvas to finish.
- Hover over a shape and four small blue arrows appear around it. Drag one of them onto another shape to connect the two.
- Click a shape and press Delete to remove it. Its arrows go with it.

---

## Setup before you press Start (not recorded)

1. **Firefox downloads.** In Firefox, open Settings, then General, then Files and Applications, and set Portable Document Format (PDF) to **Save File**.
2. **Clean the Dock.** Close the Studio browser window and Insightful's window, but leave Insightful running. Turn off "Show suggested and recent apps in Dock" in System Settings, under Desktop & Dock.
3. **UI change:** change the accent color to purple (System Settings, then Appearance, then Accent color).
4. **draw.io.** Install the draw.io desktop app if you don't have it. Make a copy of `input 2.drawio` in Downloads named `Webhook flow.drawio`, so the original input stays untouched. Open the copy, check that the diagram fits on screen with empty space below it, and minimize the window. Don't use any AI or Generate option in draw.io.
5. **Google Docs.** Upload `input 1.docx` and `input 3.docx` to Drive with your mercor.expert account and open them as Google Docs. Put them in one Firefox window as two tabs: tab 1 is the problem brief and tab 2 is the decision record. Click tab 1, scroll to the top, and minimize the window.
6. **Goal doc.** Open the Goal doc (Google Docs) in a **separate** Firefox window. It should be the only window visible.
7. **TaskRecorder.** Open it on top of the Goal doc. Paste the Goal, add the apps draw.io and Google Docs, and select the microphone.
8. Quit everything else. Accessibility Inspector should be running with its window closed. Turn Do Not Disturb on. The only window previews in the Dock should be the minimized draw.io and Firefox windows.

---

## Intro (under 20 seconds)

*[Click Start Recording]*

*[Straight away, minimize TaskRecorder with its yellow button]*

"I will be beginning my task soon. Here is the workflow I have in mind."

"I'm closing the goal document with the red button at the top left of the window."

---

## Part 1: Google Docs, the problem brief

"I'm moving my mouse to the Dock on the left side of the screen and clicking the Firefox window there. It opens on the first tab, the problem brief in Google Docs."

"I'm moving down to the first heading, What happens today, and pointing at the first paragraph under it. The card processor sends us an event every time a payment or refund changes. Our handler replies 200 straight away and hands the event to a background thread in memory. I'm moving to the second paragraph. If the server restarts before that thread finishes, the event is gone. And because we already said 200, the processor never sends it again."

"Moving down to the second heading, The numbers. I'm pointing at the second bullet. About 45 payments a week get stuck as pending, and support spends about 15 hours a week fixing them by hand. That's the cost of doing nothing."

"The third bullet. Servers restart about 25 times a week. So this isn't rare bad luck. It happens every time we deploy."

"And the fourth bullet. This one's important. Handling an event takes 300 milliseconds normally, but up to 12 seconds at peak, because of ledger service."

"Moving down to the third heading, What the processor does. In the first bullet, we have to reply within 10 seconds, or it counts as failed. The second bullet says it retries for up to an hour. And the third bullet says every event has a unique event ID, but the same event can arrive more than once. So whatever we build has to handle duplicates."

"Now the fourth heading, Options we've talked about. I'm pointing at the first bullet, Option A. Do all the work before replying 200. That's the simplest. But 12 seconds at peak is more than the 10-second limit. So at peak the processor would think it failed and send it again while we're still working on the first one."

"The second bullet, Option B. Save the raw event in an inbox table first, reply 200, and then process it from a queue with a retry worker. Anything that keeps failing goes to a dead letter queue. Saving to the database takes a few milliseconds, so we'll always answer well within 10 seconds, and nothing lives only in memory."

"The third bullet, Option C. Keep today's flow and fix stuck payments every night from the processor's report. It catches everything eventually, but a customer could wait up to a day, and support would still see the problem."

"And the last heading at the bottom, Limits. It has to ship this quarter, and we already run Postgres and Pub/Sub. Option B only uses those two, so it fits."

"I'm going with Option B. I'm minimizing Firefox with the yellow button at the top left, because I'll be back in it later."

---

## Part 2: draw.io, where it breaks today

"I'm moving to the Dock and clicking the draw.io window to open our current architecture diagram."

"I'm moving my mouse along the flow from left to right across the middle of the canvas. The grey card processor box on the far left sends the webhook to the load balancer, then to the blue webhook handler box in the payments API."

"I'm pointing at the red dashed box to the right of the handler, the background thread in memory. The arrow into it says hand off, reply 200 right away. From there, one arrow goes up to the green payments database on the top right, and one goes down to the green ledger service on the bottom right."

"This red box is exactly where we lose events. It only exists in the server's memory. A restart wipes it, and we've already said 200."

---

## Part 3: draw.io, drawing the new design

"First I'm removing it. I'm clicking the red background thread box and pressing the Delete key. Its arrows go with it."

**The inbox table**

"In the shape panel on the left side of the window, under General, I'm finding the cylinder, which is the database shape. I'm dragging it onto the canvas, below the webhook handler."

"I'm double-clicking it and typing the label, then clicking an empty part of the canvas."
⌨️ `webhook_inbox (Postgres)`

"Now I'm hovering over the blue webhook handler box. Four small blue arrows appear around it. I'm dragging the bottom blue arrow down onto the inbox table to connect them."

"I'm double-clicking the new arrow and labelling it. This is step one: save the event first."
⌨️ `1. save event, then reply 200`

**The queue**

"Back in the shape panel on the left, under General, I'm dragging the rectangle, the first shape, onto the canvas, to the right of the webhook handler, where the background thread used to be."

"Double-clicking it to label it."
⌨️ `Pub/Sub: webhook-events`

"Hovering over the webhook handler and dragging the blue arrow on its right side onto the queue."

"Double-clicking that arrow to label it."
⌨️ `2. publish event_id`

**The retry worker**

"Dragging another rectangle from the shape panel onto the canvas, to the right of the queue and to the left of the database and ledger boxes."

"Double-clicking to label it."
⌨️ `Webhook worker (retries with backoff)`

"Hovering over the queue and dragging the blue arrow on its right side onto the worker."

"Now the worker does what the background thread used to do. I'm hovering over the worker and dragging the blue arrow on its top onto the Payments DB on the top right."

"Double-clicking that arrow."
⌨️ `update payment, skip seen event_id`

"And one more, from the blue arrow on the worker's right side to the Ledger service on the bottom right. Double-clicking it to label it."
⌨️ `post ledger entry`

**The dead letter queue**

"Last, the dead letter queue. I'm dragging a rectangle from the shape panel to below the worker."

"Double-clicking to label it."
⌨️ `Dead letter queue + alert`

"Hovering over the worker and dragging the bottom blue arrow onto the dead letter queue."

"Double-clicking that arrow."
⌨️ `after 5 failed tries`

"I'm double-clicking the title at the top left of the canvas and changing current to proposed."
⌨️ `Payments: card processor webhook flow (proposed)`

"I'm moving my mouse along the new flow to check it. The handler saves the event and replies, the event ID goes on the queue, the worker processes it and retries if it fails, and anything that keeps failing goes to the dead letter queue and alerts us. Nothing lives only in memory anymore."

---

## Part 4: draw.io, save and export

"I'm moving to the File menu at the top left of the screen and clicking Save."

"Now the File menu again, hovering over Export as, and clicking PNG. In the export options I'm clicking Export, naming the file in the save dialog, and clicking Save."
⌨️ `Webhook_Flow_Proposed`

"I'm done in draw.io. I'm closing the draw.io window with the red button at the top left."

---

## Part 5: Google Docs, the decision record

"I'm moving to the Dock and clicking the Firefox window. I'm clicking the second tab at the top, the decision record template."

"I'm clicking at the end of the Status line, the first line under the title."
⌨️ `Proposed`

"I'm clicking on the empty line under the Context heading."
⌨️ `A restart drops webhooks we've already replied 200 to. About 45 payments a week get stuck, and the processor never retries them.`

"I'm clicking on the empty line under the Decision heading."
⌨️ `Option B. Save each webhook in webhook_inbox before replying 200, then process it from Pub/Sub with a retry worker and a dead letter queue.`

"Under the Why not the other options heading, I'm clicking at the end of the first bullet, Option A."
⌨️ `At peak, handling takes up to 12 seconds, over the 10 second limit, so the processor would retry and send duplicates.`

"And at the end of the second bullet, Option C."
⌨️ `Customers could wait up to a day. Worth adding later as a safety net.`

"Under the last heading, Risks and follow-ups, I'm clicking on the first bullet. If a server crashes after saving but before publishing, the event sits in the inbox and nobody picks it up. So we need a sweeper."
⌨️ `Crash between saving and publishing: a sweeper re-publishes inbox rows not done after 5 minutes.`

"And the second bullet. The processor can send the same event twice."
⌨️ `Duplicates: webhook_inbox is unique on event_id, so the worker skips events it has seen.`

"I'm scrolling back to the top and reading down to check it. The decision matches the diagram, and the reasons against A and C use the numbers from the brief. All good."

"I'm moving to the File menu at the top left of Google Docs, hovering over Download, and clicking PDF Document."

*Only if a new PDF tab or Preview opens on its own:* "The PDF opened automatically, and I don't need it, so I'm closing it." For a Firefox tab, click the small x on that tab. For Preview, use the Preview menu at the top left and click Quit Preview.

---

## Epilogue (3 to 4 minutes, max 4:00; the mouse can rest)

"So, to wrap up. The task was to pick a design that stops us losing webhooks from the card processor and write it up for the team. I picked Option B: save every webhook in an inbox table before we reply, then process it from a queue with a retry worker and a dead letter queue.

To check my work, I went back to the brief's numbers for each option. The 12-second peak against the 10-second limit rules out A. The one-day delay rules out C as the main fix. And B only uses Postgres and Pub/Sub, which we already run, so it can ship this quarter.

My approach was to find the exact point of failure before drawing anything new. Once it was clear the problem was the in-memory hand-off, the fix was obvious: never say 200 until the event is somewhere durable.

The hardest part was the gap between saving the event and publishing it. If a server dies right between the two, we've fixed one way of losing events and created another. That's why the sweeper is in the risks, and I'd want it in the first version, not later.

If I did this again, I'd ask for data on how many events are duplicates today, because that decides how much the event ID check matters. With more time, I'd also add the nightly reconciliation from Option C as a safety net on top of B. It's cheap, and it catches anything we haven't thought of.

And one insight for anyone designing around webhooks. A 200 is a promise. Once you send it, the other side forgets the event. So the rule is simple: only reply once the event is stored somewhere that survives a restart."

---

## Ending

"I'm closing my Firefox window with the red button at the top left, and clicking Close Tabs to confirm."

*[Click TaskRecorder in the Dock, click Stop Recording, wait for Idle]*
