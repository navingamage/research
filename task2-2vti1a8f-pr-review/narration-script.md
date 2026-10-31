# Script: reviewing a junior engineer's pull request

**Target length:** about 30 minutes. The limits are 15:00 to 50:59, and the epilogue has to be 3 to 4 minutes (max 4:00).

**How to use it:**
- Each quoted line says what you're doing, and you do it as you say it. Lines in *[brackets]* are silent actions. ⌨️ lines are what you type.
- Say the mouse movement out loud every time: "I'm moving my mouse to...", "I'm clicking...".
- Move slowly, pause on the thing you're naming, and pause about a second after each click.
- If you use a keyboard shortcut, say it out loud first ("I'm pressing Command S to save").
- Do one full dry run. VS Code's layout can differ a little by version, so adjust the wording to what you actually see.

---

## Setup before you press Start (not recorded)

1. **Firefox downloads.** In Firefox, open Settings, then General, then Files and Applications, and set Portable Document Format (PDF) to **Save File**, so the PDF doesn't open in another app.
2. **Clean the Dock.** Close the Studio browser window and Insightful's window, but leave Insightful running. Turn off "Show suggested and recent apps in Dock" in System Settings, under Desktop & Dock.
3. **Project.** Unzip `input 1.zip` in Downloads, which gives you a `refund-service` folder. In Terminal, run these two lines, then quit Terminal:
   ```
   cd ~/Downloads/refund-service
   python3 -m venv .venv && .venv/bin/pip install pytest
   ```
4. **VS Code.**
   - Install the **Python** extension from Microsoft.
   - Turn off AI. Open Settings, search for "disable AI features", and tick it. Don't sign in to GitHub or Copilot. The Accounts icon at the bottom left should show no account.
   - Use File, then Open Folder, to open `refund-service`. If it asks, click "Yes, I trust the authors".
   - Click the Testing icon (the beaker) once so it finds the 3 tests, then click back to the Explorer icon.
   - Close every editor tab and any notification pop-ups.
   - **UI change:** set the color theme to Solarized Light (Settings, then Themes, then Color Theme).
   - Minimize VS Code with its yellow button.
5. **Google Docs.** Upload `input 2.docx` to Drive with your mercor.expert account and open it as a Google Doc. Keep it as the only tab in one Firefox window, then minimize that window.
6. **Goal doc.** Open the Goal doc (Google Docs) in a **separate** Firefox window. It should be the only window visible.
7. **TaskRecorder.** Open it on top of the Goal doc. Paste the Goal, add the apps Visual Studio Code and Google Docs, and select the microphone.
8. Quit everything else. Accessibility Inspector should be running with its window closed. Turn Do Not Disturb on. The only window previews in the Dock should be VS Code and the minimized Firefox window.

---

## Intro (under 20 seconds)

*[Click Start Recording]*

*[Straight away, minimize TaskRecorder with its yellow button]*

"I will be beginning my task soon. Here is the workflow I have in mind."

"I'm closing the goal document with the red button at the top left of the window."

---

## Part 1: VS Code, reading the change

"I'm moving my mouse to the Dock on the left side of the screen and clicking the VS Code icon to open the refund service project."

"Sam is one of the junior engineers on my team. He's changed the refund service so that if the payment processor times out, we try the refund again. I've pulled his changes into my local copy, so they show up as changes in VS Code."

"On the far left is the Activity Bar. I'm clicking the third icon down, the one that looks like a branch with a small number on it. That's the Source Control view."

"Under Changes there are three files with an M next to them, which means modified: config dot py, refunds dot py, and test underscore refunds dot py."

**config.py**

"I'm clicking config dot py first. It opens side by side. The old version is on the left and Sam's version is on the right, and the new lines are highlighted in green."

"He's added two settings. Max attempts is 3, and the retry delay is 2 seconds. That's fine. Keeping them in config means we can tune them without changing code."

**refunds.py**

"Now I'm clicking refunds dot py in the Changes list. This is the real change."

"I'm moving my mouse down the right side. Before, we called the processor once, and if it timed out we raised refund failed. Now there's a for loop that tries up to three times."

"I'm pointing at the line that says except Exception. That catches every kind of error, not just timeouts. So if the processor rejects a refund, for example because the amount is more than the customer paid, we'll try it again three times. That will never succeed, and it just wastes time."

"I'm moving to the sleep line under it. The wait is the delay times the attempt number, so zero, then 2 seconds, then 4. It also sleeps after the last attempt, which is pointless."

"And the big one. I'm pointing at the line that calls processor dot refund. Every attempt calls it the same way, with no idempotency key. A timeout doesn't mean the refund failed. It means we don't know. If the processor did the refund and we just didn't get the answer back, the retry refunds the customer a second time. That's real money going out twice."

**test_refunds.py**

"I'm clicking test underscore refunds dot py, the third file."

"He's added a test called test retries after timeout. I'm pointing at the first line inside it. The fake processor times out before it does anything, then the retry goes through. That's a good test, but it only covers the easy case where nothing happened on the first call. It doesn't cover the case where the refund went through and only the response was lost."

---

## Part 2: VS Code, running the tests

"Let me see what the tests say. I'm moving back to the Activity Bar on the left and clicking the beaker icon near the bottom. That's the Testing panel."

"At the top of the panel there's a button with two small play arrows, Run Tests. I'm clicking it."

*[wait for the tests to finish]*

"All three tests have green ticks: refund succeeds, rejects zero amount, and Sam's new retry test. So the tests are green, but I already know there's a bug. I want a test that shows it, so it can't come back later."

---

## Part 3: VS Code, a test that proves the bug

"I'm clicking the Explorer icon at the top of the Activity Bar, the one that looks like two pages."

"In the file tree I'm moving my mouse to the tests folder, right-clicking it, and choosing New File."

"I'm typing the file name, test underscore refund underscore retries dot py, and pressing Return."
⌨️ `test_refund_retries.py`

*Typing tips: press Return after each ⌨️ line, and leave one empty line between the imports and the def line. VS Code indents the lines after `def` for you, so don't type those leading spaces yourself. It also adds closing brackets and quotes as you type, so just keep typing over them.*

"First, the two imports. The refund service, and the fake processor we use in tests."
⌨️ `from refund_service.refunds import RefundService`
⌨️ `from tests.fakes import FakeProcessor`

"Now the test. The fake processor has an option called lose first responses. It does the refund, then times out before we hear back. That's exactly the case I'm worried about."
⌨️ `def test_retry_does_not_refund_twice():`
⌨️ `    processor = FakeProcessor(lose_first_responses=1)`

"Then I issue one refund of 40 dollars, which is 4000 cents."
⌨️ `    RefundService(processor).issue_refund("pay_300", 4000)`

"And the customer should only be refunded once."
⌨️ `    assert len(processor.refunds) == 1`

"I'm moving to the File menu at the top left, and clicking Save."

"Back to the Testing panel. I'm clicking the beaker icon on the left. The new test shows up under test refund retries. I'm clicking the Run Tests button with the two play arrows at the top again."

*[wait for the tests to finish]*

"Three green ticks and one red cross. The red one is my new test. I'm clicking it to see why."

"The message says it expected 1 refund and got 2. Same payment, same amount, twice. So that's the bug, proven. The test fails now, and it will pass once Sam sends an idempotency key on every attempt."

"I'm done in VS Code. I'm closing the VS Code window with the red button at the top left."

---

## Part 4: Google Docs, the review feedback

"I'm moving to the Dock and clicking the Firefox window there to open my review template in Google Docs."

"This is the template I use for code reviews. The change, the author and the reviewer are filled in already. I'll fill in the rest."

"I'm clicking at the end of the Decision line and typing."
⌨️ `Request changes`

"Under What's good, I'm clicking on the empty bullet. I always start with what's good, especially for a junior engineer."
⌨️ `Small, focused change with a test, and every failed attempt is logged.`

"Under Must fix before merge, I'm clicking on the first numbered line."
⌨️ `Create one idempotency key per refund before the loop and send it on every attempt. Without it a retry can refund twice. test_refund_retries.py shows it.`

"And the second numbered line."
⌨️ `Only retry ProcessorTimeout. Catching Exception retries rejected refunds too.`

"Under Nice to have, I'm clicking on the bullet."
⌨️ `Don't sleep after the last attempt, and add a test for a rejected refund.`

"And the last section, Something to learn from this. I'm clicking on the empty line under it. This is the part I care about most, because it's the lesson Sam can use again."
⌨️ `A timeout doesn't mean it failed. It means we don't know. Before you retry anything that moves money, ask what happens if the first call worked.`

"I'm scrolling back to the top and reading down to check it. The decision, two must-fix items that both point at real lines in the change, a nice-to-have, and the lesson. All good."

"I'm moving to the File menu at the top left of Google Docs, hovering over Download, and clicking PDF Document to save the review."

*Only if a new PDF tab or Preview opens on its own:* "The PDF opened automatically, and I don't need it, so I'm closing it." For a Firefox tab, click the small x on that tab. For Preview, use the Preview menu at the top left and click Quit Preview.

---

## Epilogue (3 to 4 minutes, max 4:00; the mouse can rest)

"So, to wrap up. The task was to review Sam's change that adds retries to our refund service before it goes near production. My decision is request changes, with two must-fix items.

To check my work, I didn't just trust my reading of the code. I wrote a test that reproduces the problem, and it fails with two refunds instead of one. That's what gives the feedback weight. Sam can run it himself, and when it goes green he knows the fix is right.

My approach was to read the diff file by file, smallest first, so I understood the settings before the logic. Then I ran the existing tests to see what they actually cover. They all passed, which was the interesting part. Green tests don't mean the code is safe. They only cover the cases someone thought of.

The hardest part was the tone. Sam is junior, and the change is a reasonable first attempt. Retries are a good idea here. So I led with what's good, kept the must-fix list to two items, and pushed the smaller things to nice to have, so he isn't buried in comments.

If I did this again, I'd look at whether the processor has a status check we could call before retrying, as another safety net. With more time, I'd also pair with Sam for half an hour on the fix instead of only leaving written feedback. Idempotency is one of those things that clicks faster when you talk it through.

And one insight for anyone reviewing payments code. A timeout is not a failure. It's an unknown. Any retry on a call that moves money needs an answer to one question: what happens if the first call actually worked? If there's no good answer, the retry is a bug waiting to happen."

---

## Ending

"I'm closing my Firefox window with the red button at the top left."

*[Click TaskRecorder in the Dock, click Stop Recording, wait for Idle]*
