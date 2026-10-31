# Studio idea text (paste into "Describe your idea")

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
