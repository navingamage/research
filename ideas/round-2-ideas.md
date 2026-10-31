# Next 3 task ideas

Each one is from my day job and uses a different workflow and set of apps from Task 1 and from the others, so none of them repeats a workflow. Each is planned for 30 to 35 minutes, mostly narration, with about 3 minutes of typing.

| # | Idea | Apps | Shape |
|---|---|---|---|
| 2 | Review a junior engineer's pull request | Visual Studio Code, Google Docs | Read code, run tests, prove a bug, write mentoring feedback |
| 3 | Postmortem for a payment API outage | Numbers, Pages | Dig through logs, find the cause, write the postmortem |
| 4 | Design a fix for lost payment webhooks | draw.io, Google Docs | Read the brief, change the architecture diagram, write the decision record |

Before submitting, check that Visual Studio Code, Numbers, Pages and draw.io are on the validated apps list.

---

## Idea 2: Review a junior engineer's pull request

**Domain / Role:** Software engineering. Tech lead on a payments platform team, reviewing code from a junior engineer.

**Goal:** One of the junior engineers on my team has put up a change that adds automatic retries to our refund service, and I need to review it before it goes anywhere near production. In Visual Studio Code I'll go through the changed files in the Source Control view and read the diff, then run the existing tests from the Testing panel. I'm worried a retry after a timeout could refund a customer twice, so I'll add one small test that shows it, run it and see it fail. Then in Google Docs I'll write my review feedback for the engineer: what they did well, what has to change before merge, and one thing to learn from it. Deliverables are the new failing test file and the review feedback as a PDF.

**Starting materials:** Using existing material. A small Python refund service project with the engineer's changes, its existing tests, and a blank review feedback template. All AI-generated before recording. The people and the code are made up.

**Steps (with the apps for each):**
- Visual Studio Code: open the Source Control view, go through each changed file in the side-by-side diff and point out what the change does and where it's risky.
- Visual Studio Code: run the existing tests from the Testing panel and show they all pass, even with the problem in the code.
- Visual Studio Code: add one short test where the first refund call times out and the retry goes through, run it, and show that it fails because the customer gets refunded twice.
- Google Docs: fill in the review template with what's good, the must-fix items (an idempotency key on the retry, not catching every exception), a nice-to-have, and a learning note for the engineer, then download it as a PDF.

**Deliverable:** The new test file that fails and proves the double refund, and the review feedback PDF. A reviewer can check that the failing test matches the bug described in the feedback and that each must-fix item points to a real line in the change.

**Apps:** Visual Studio Code, Google Docs

---

## Idea 3: Postmortem for a payment API outage

**Domain / Role:** Software engineering. Tech lead and incident owner for a payments API.

**Goal:** Last Thursday our payments API slowed down badly for about 40 minutes and a few percent of card payments timed out. I was the incident lead, and now I need to work out the real cause and write the postmortem. In Numbers I'll open the request log export, filter down to the failed and slow requests, and build a pivot table and a chart by minute, version and server to see where the problem actually was. I'll compare that against the deploy log and the release notes to find the change that caused it. Then in Pages I'll write the postmortem with the impact, a short timeline, the root cause and the follow-up actions with owners. Deliverables are the log analysis workbook and the postmortem as a PDF.

**Starting materials:** Using existing material. A CSV export of request logs for the incident window, a deploy log, the release notes for that week's release, and a blank postmortem template. All AI-generated before recording. The company, the people and the data are made up.

**Steps (with the apps for each):**
- Numbers: open the request log, sort and filter to the timeouts and slow requests, and point out when they started and stopped.
- Numbers: build a pivot table of failures by app version and server, and a chart of slow requests by minute, to show only the servers on the new version were affected.
- Numbers: check the deploy log and release notes tables against the chart and find the change that lines up with the start of the incident.
- Pages: fill in the postmortem template with the summary, customer impact, timeline, root cause and three follow-up actions with owners, then export it as a PDF.

**Deliverable:** The log analysis workbook with the filter, pivot table and chart, and the postmortem PDF. A reviewer can check that the root cause in the postmortem matches what the pivot table and chart show.

**Apps:** Numbers, Pages

---

## Idea 4: Design a fix for lost payment webhooks

**Domain / Role:** Software engineering. Tech lead deciding a design change for a payments service.

**Goal:** Our payments service sometimes loses the webhooks the card processor sends us when a server restarts, so some payments get stuck as pending until someone fixes them by hand. I need to pick a design to fix this and get it written up for the team. In Google Docs I'll read the problem brief with the numbers and the three options we've talked about. In draw.io I'll open our current architecture diagram and update it with the option I pick: storing each webhook first, then a queue, a retry worker and a dead letter queue. Then back in Google Docs I'll write a short decision record with the option I chose, why I didn't pick the other two, and the risks. Deliverables are the updated diagram and the decision record as a PDF.

**Starting materials:** Using existing material. A problem brief doc, the current architecture diagram as a draw.io file, and a blank decision record template. All AI-generated before recording. The company and the systems are made up.

**Steps (with the apps for each):**
- Google Docs: read the problem brief and point out the numbers and limits that matter, like how many webhooks we lose, the peak volume, and that the processor only retries for one hour.
- draw.io: open the current diagram, show where a webhook gets lost today, then drag in the new parts from the shape panel, connect them with arrows and label each step.
- draw.io: export the updated diagram as a PNG.
- Google Docs: fill in the decision record with the context, the chosen option, why the other two lost, and the risks and follow-ups, then download it as a PDF.

**Deliverable:** The updated architecture diagram (.drawio and PNG) and the decision record PDF. A reviewer can check that the diagram shows the option the decision record picks and that the reasons hold up against the numbers in the brief.

**Apps:** draw.io, Google Docs
