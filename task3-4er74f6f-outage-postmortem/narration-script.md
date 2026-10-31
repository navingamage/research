# Script: postmortem for the payments API slowdown

**Target length:** about 30 to 35 minutes. The limits are 15:00 to 50:59, and the epilogue has to be 3 to 4 minutes (max 4:00).

**How to use it:**
- Each quoted line says what you're doing, and you do it as you say it. Lines in *[brackets]* are silent actions. ⌨️ lines are what you type.
- Say the mouse movement out loud every time: "I'm moving my mouse to...", "I'm clicking...".
- Move slowly, pause on the thing you're naming, and pause about a second after each click.
- If you use a keyboard shortcut, say it out loud first.
- Do one full dry run. Numbers menus differ a little between versions, so adjust the wording to what you actually see, especially around the pivot tables and the chart.

**The story in the data (so you know where you're heading):**
- The log runs from 13:45 to 15:15 on Thursday October 29, across six servers, pay-api-1 to 6.
- Version 2.14.0 went out to pay-api-4, 5 and 6 at 14:02. The first slow request was at 14:04 and the last at 14:43. The rollback finished at 14:44.
- Every slow request and every timeout happened on those three servers, on 2.14.0. pay-api-1 to 3 were fine the whole time.
- Totals on 2.14.0: 70 timeouts and 259 slow requests (over 2 seconds). There were about 1,300 requests in that 40-minute window, so about 5 percent timed out.
- The cause is PAY-830 in the release notes: the database connection pool went from 40 to 8 per server. At peak traffic, requests queued for a free connection until they hit the 30-second timeout. The database itself was healthy.
- The red herring is the fraud-service deploy at 13:55. If it were the cause, all six servers would have been hit.

---

## Setup before you press Start (not recorded)

1. **Clean the Dock.** Close the Studio browser window and Insightful's window, but leave Insightful running. Turn off "Show suggested and recent apps in Dock" in System Settings, under Desktop & Dock.
2. **UI change:** turn on Dark Mode (System Settings, then Appearance, then Dark).
3. **Numbers.** Open `input 1.xlsx` in Numbers, then use File, then Save, to save it in Downloads as **Incident analysis**. You should see three sheet tabs at the top: Request Log, Deploy Log and Release Notes. Click the Request Log tab, scroll to the top, and click cell A1. Minimize the window.
4. **Pages.** Open `input 2.docx` in Pages and save it in Downloads as **Postmortem**. Scroll to the top and minimize the window.
5. **Goal doc.** In Pages, create a new blank document, paste the Goal, and save it as **Goal**. It should be the only window visible. Saving it first means closing it during the intro won't pop up a save dialog.
6. **TaskRecorder.** Open it on top of the Goal doc. Paste the Goal, add the apps Numbers and Pages, and select the microphone.
7. Quit everything else. Accessibility Inspector should be running with its window closed. Turn Do Not Disturb on. The only window previews in the Dock should be the minimized Numbers and Pages windows.

---

## Intro (under 20 seconds)

*[Click Start Recording]*

*[Straight away, minimize TaskRecorder with its yellow button]*

"I will be beginning my task soon. Here is the workflow I have in mind."

"I'm closing the goal document with the red button at the top left of the window."

---

## Part 1: Numbers, finding when it started and stopped

"I'm moving my mouse to the right side of the Dock and clicking the Incident analysis window to open it in Numbers."

"Last Thursday our payments API slowed down for about 40 minutes, and some card payments timed out. I was the incident lead. During the incident we rolled back a release to stop it, but we never proved what caused it. That's what this is for."

"At the top there are three sheet tabs: Request Log, Deploy Log and Release Notes. I'm on Request Log. This is an export of every request from 13:45 to 15:15 that day."

"I'm moving across the header row. Timestamp and minute, the request ID, the endpoint, which server handled it, the app version that server was running, the HTTP status, the latency in milliseconds, and a result column. Result is ok, slow if it took over 2 seconds, or timeout if it hit our 30-second limit and returned a 504."

"There are about 2,700 rows, so I'm not going to read them. First I'll filter down to just the bad ones."

"I'm moving to the top right of the Numbers window and clicking the Organize button. The Organize sidebar opens on the right."

"At the top of the sidebar I'm clicking the Filter tab, then the Add a Filter button."

"It asks which column. I'm choosing result."

"Now the rule. I'm clicking Text, then is not, and typing ok in the box."
⌨️ `ok`

*[press Return]*

"Now the table only shows slow requests and timeouts."

"Next I'm sorting it by time, so the first and last rows tell me when it started and stopped. I'm clicking the Sort tab at the top of the sidebar, then Add a Column, and choosing timestamp. It's set to ascending, which is what I want."

"I'm moving to the first row of the table. The first bad request is at 14:04 and 16 seconds, on pay-api-6. That's two minutes after something. I'll come back to that."

"I'm scrolling down to the bottom of the table. The last bad request is at 14:43 and 53 seconds. So the incident ran from 14:04 to about 14:44. That's the 40 minutes."

"While I'm scrolling, I'm moving my mouse down the server column. I keep seeing pay-api-4, 5 and 6, and never 1, 2 or 3. And the app version column says 2.14.0 every time. That's a strong hint, but I want to prove it, not eyeball it."

"I'm clicking the Filter tab again and turning off the Filters switch at the top, so the table goes back to every row."

---

## Part 2: Numbers, which servers were hit

"I'm clicking any cell inside the Request Log table. Now I'm moving to the toolbar at the top and clicking the Pivot Table button, then choosing Create Pivot Table on New Sheet."

"Numbers opens a new sheet with an empty pivot table. On the right, the sidebar shows Pivot Options with three sections: Columns, Rows and Values."

"Next to Rows I'm clicking Add Field and choosing app_version. I'm clicking Add Field under Rows again and choosing server. Now each version is broken down by server."

"Next to Columns I'm clicking Add Field and choosing result."

"Next to Values I'm clicking Add Field and choosing request_id. It counts them, which is what I want."

"I'm moving my mouse down the table. Under 2.13.2, all six servers have only ok results. pay-api-1, 2 and 3 have about 450 each, and 4, 5 and 6 have fewer, because they were only on 2.13.2 for part of the time."

"Now under 2.14.0. Only pay-api-4, 5 and 6, and this is where every slow request and every timeout is. I'm pointing at the slow column: 92, 86 and 81. And the timeout column: 22, 24 and 24. The totals are 259 slow and 70 timed out."

"So it's not the database, and it's not the network, because those would hit all six servers. It's something in version 2.14.0. In those 40 minutes we handled about 1,300 requests, so roughly 5 percent of them timed out."

"I'm double-clicking the sheet tab at the top, the new one called Pivot Table, and renaming it By server."
⌨️ `By server`

---

## Part 3: Numbers, the shape over time

"Now I want to see it minute by minute. I'm clicking the Request Log tab at the top, then clicking any cell in the table."

"I'm moving to the toolbar, clicking Pivot Table, and choosing Create Pivot Table on New Sheet again."

"In the sidebar, next to Rows I'm clicking Add Field and choosing minute. Next to Columns, Add Field, then result. And next to Values, Add Field, then request_id, which counts them again."

"This gives me one row for every minute, with how many requests were ok, slow or timed out."

"Now the chart. I'm clicking a cell in the pivot table, moving up to the toolbar and clicking the Chart button. Under 2D, I'm choosing the line chart."

*If Numbers asks which values to chart, choose the summary values. If the chart shows lots of lines instead of three, open the chart's data references and switch "plot rows as series" to "plot columns as series".*

"I'm dragging the chart below the table so they don't overlap, and dragging its bottom right corner to make it bigger."

"I'm moving my mouse along the chart. The ok line is steady the whole time. The slow and timeout lines are flat at zero until 14:04, then they jump up. They're highest around 14:20 to 14:30, which is our lunchtime peak, and they drop straight back to zero at 14:44 when the rollback finished."

"That peak matters. It only got bad when traffic was high. That tells me it's a capacity limit, not a bug that breaks every request."

"I'm double-clicking this sheet tab and renaming it By minute."
⌨️ `By minute`

---

## Part 4: Numbers, finding the change

"Now, what changed at 14:02? I'm clicking the Deploy Log tab at the top."

"I'm moving down the rows. At 13:55, fraud-service 3.8.1 was released. That's close in time, but fraud-service is used by all six servers. If it were the cause, pay-api-1 to 3 would have been hit too, and they weren't. So I can rule it out."

"At 14:02, payments-api 2.14.0 rollout step 1 of 2, to pay-api-4, 5 and 6. That's two minutes before the first bad request, and it's exactly the three servers that were hit."

"At 14:12 the second step was paused. That was me, as a precaution. That's why pay-api-1 to 3 never got the new version, and it's also why we have such a clean comparison."

"Then 14:41 the rollback started, and 14:44 it finished. That matches the end of the incident on the chart."

"So it's 2.14.0. Now which change in it? I'm clicking the Release Notes tab at the top."

"There are four changes. PAY-812 is the currency symbol on receipts. That's just text, so it's not it. PAY-833 is a logging library upgrade. That's possible, but unlikely."

"PAY-835 retries the fraud check once on a 503. That could add a bit of load, but fraud-service wasn't returning 503s."

"And PAY-830. I'm pointing at it. It lowered the database connection pool from 40 to 8 per server, to cut idle connections. That's it. With only 8 connections, every request at peak has to wait for a free one. They queue up, and the ones at the back hit the 30-second timeout. It also explains why the database looked healthy when we checked it at 14:20. The database was fine. The queue was in front of it."

"I'm saving a copy for the postmortem. I'm moving to the File menu at the top left, hovering over Export To, and clicking Excel. In the dialog I'm clicking Next, naming the file, and clicking Export."
⌨️ `Payments_Incident_Analysis`

"I'm done in Numbers. I'm closing the Numbers window with the red button at the top left."

---

## Part 5: Pages, the postmortem

"I'm moving to the Dock and clicking the Postmortem window to open it in Pages."

"This is our postmortem template. The timeline already has what we did during the incident from the incident channel. I'll fill in the rest."

"I'm clicking on the empty line under Summary."
⌨️ `The payments API was slow for 40 minutes on the three servers running 2.14.0, and some card payments timed out.`

"I'm clicking on the empty line under Customer impact."
⌨️ `From 14:04 to 14:44, 70 requests timed out and 259 took over 2 seconds, about 5% timeouts. Only pay-api-4 to 6 were affected.`

"Now the timeline. The two empty rows at the top are for what we learned afterwards. I'm clicking the first empty Time cell."
⌨️ `14:02`

"Pressing Tab to move to the What happened cell. I'm saying it because Tab is a shortcut."
⌨️ `2.14.0 rolled out to pay-api-4, 5 and 6`

"Clicking the second empty Time cell."
⌨️ `14:04`

"Pressing Tab again."
⌨️ `First slow requests and timeouts, only on pay-api-4 to 6`

"I'm scrolling down and clicking on the empty line under Root cause. I'll type this in two parts."
⌨️ `PAY-830 in 2.14.0 lowered the database connection pool from 40 to 8 per server.`

"That's what changed. Now why it hurt."
⌨️ ` At peak traffic, requests waited for a free connection until they hit the 30 second timeout. The database itself was healthy.`

"Last, the follow-up actions table. I'm clicking the first empty Action cell. The first one is mine. We need to see this coming next time."
⌨️ `Alert on connection pool wait time` *[Tab]* `Me` *[Tab]* `Nov 6`

"Clicking the next Action cell. Config changes like this one need a load test, and Aisha made this change, so she's the right owner."
⌨️ `Load test pool and timeout changes before release` *[Tab]* `Aisha` *[Tab]* `Nov 13`

"And the last one. The rollout should have stopped itself."
⌨️ `Auto rollback a rollout step if p99 latency doubles` *[Tab]* `Dev` *[Tab]* `Nov 20`

"I'm scrolling back to the top and reading down to check it. The times in the timeline match the log, the numbers match the pivot table, and the root cause matches the release notes. All good."

"I'm moving to the File menu at the top left, hovering over Export To, and clicking PDF. In the dialog I'm clicking Next, naming the file, and clicking Export."
⌨️ `Postmortem_Payments_API_Oct29`

---

## Epilogue (3 to 4 minutes, max 4:00; the mouse can rest)

"So, to wrap up. The task was to find the real cause of last Thursday's payments API slowdown and write the postmortem. The cause was one line in release 2.14.0, PAY-830, which cut the database connection pool from 40 to 8 per server. At peak traffic, requests queued for a connection until they timed out.

To check my work, I made sure three separate things agree. The pivot table shows every bad request was on the 2.14.0 servers. The chart shows it started two minutes after the deploy and stopped the minute the rollback finished. And the release notes have one change that explains both the timing and why it only got bad at peak.

My approach was to let the data narrow it down before I looked at any changes. If I'd started with the release notes, I'd have had four suspects. Starting with the servers took it down to one release, and the shape of the chart told me it was a capacity limit.

The hardest part was the fraud-service deploy at 13:55. It was so close in time that during the incident, a couple of people blamed it. The server breakdown is what ruled it out, because fraud-service is used by every server and only half of them broke.

If I did this again, I'd pull the connection pool metrics as well, to show the pool was full and not just infer it. With more time, I'd also check how many of the 70 timed-out payments the customer retried, so we know how many actually failed for good.

And one insight for anyone running incidents. Pausing the rollout halfway gave us a perfect comparison: the same code and the same traffic, with only the version different. A staged rollout isn't just safer. It's also the best debugging tool you'll have when something goes wrong."

---

## Ending

"I'm closing my Pages window with the red button at the top left."

*[Click TaskRecorder in the Dock, click Stop Recording, wait for Idle]*
