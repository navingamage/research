# Studio idea text (paste into "Describe your idea")

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
