# Studio idea text (paste into "Describe your idea")

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
