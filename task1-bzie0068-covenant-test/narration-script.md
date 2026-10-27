# Task 1 script: covenant compliance test (simple version)

**Length target:** about 20 to 25 minutes, speaking slowly. The hard limits are 15:00 to 50:59 in total, so you have lots of room. Don't rush.

**Flow:** 3 apps, one visit each. Skim, then Google Sheets, then Pages.

**Typing:** three short formulas and a few short sentences. That's it.

**How to use it:**
- ▶ = do this. Then say the line underneath it, in your own words.
- Name what you click and where it is on screen.
- Pause about a second after each click.
- When you leave an app, minimize it with its yellow button and say so.

**Before Start:** Skim has both PDFs open, Firefox has the Sheet open, and Pages has the memo template open. All three are minimized. Only the Goal doc is showing.

---

## Intro (under 20 seconds)

▶ Click Start Recording. Minimize TaskRecorder.

"I will be beginning my task soon. Here is the workflow I have in mind."

▶ Wait 3 seconds, then minimize the Goal doc.

---

## Part 1: Skim, the credit agreement (about 6 min)

▶ Click Skim in the Dock. Open the credit agreement.

"I'm on the agent bank for Kestrel Ridge's term loan. Their Q3 compliance certificate just came in. First I'll check the credit agreement so I know the rules."

▶ In the toolbar at the top right, click the Note Tool dropdown and pick Highlight.

"I'm picking Highlight from the note tool dropdown, at the top right of the toolbar."

▶ Scroll to Consolidated EBITDA. Highlight the 25% cap sentence, the one ending "calculated before giving effect to any such add-backs".

"This is the add-back cap. Extra add-backs can't be more than 25 percent of EBITDA. And it says before the add-backs. That detail is important."

▶ Highlight "within eighteen (18) months" in clause (vii).

"Cost savings only count if they land within 18 months. For this quarter, that's March 31st, 2028."

▶ Scroll to Consolidated Total Net Debt. Highlight "not to exceed $15,000,000".

"They can only subtract up to 15 million of cash from their debt."

▶ Scroll to Section 7.11. Highlight the 5.25 row.

"This quarter the max leverage steps down to 5.25 times."

▶ Scroll to Section 8.04. Highlight the first two lines of 8.04(a).

"And if they fail, the sponsor can fix it by putting in equity. That's the cure."

---

## Part 2: Skim, the borrower's certificate (about 4 min)

▶ Click the Window menu at the top and choose the compliance certificate.

"Now the borrower's certificate, from the Window menu."

▶ Scroll to Schedule 1. Point at 5.13x.

"They say leverage is 5.13 times, just under 5.25. That's very tight, so I'll check it."

▶ Highlight "Limit: 25.0% × Consolidated EBITDA (96.0)".

"First problem. They used 96, which already includes the add-backs. It should be 72. So their cap is too big."

▶ Highlight "(28.0)" in the net debt table.

"Second problem. They subtracted all 28 million of cash. The limit is 15."

▶ Scroll to Schedule 2. Highlight the S-3 Dayton row.

"Third. The Dayton savings land in 2028, after the March 2028 cutoff. So that 3 million doesn't count."

▶ Click File, then Export. Choose "PDF With Embedded Notes" and save. Do the same for the credit agreement.

"I'll export both files with File, Export, PDF With Embedded Notes, so my highlights are saved inside them."

▶ Minimize Skim.

"Minimizing Skim with the yellow button."

---

## Part 3: Google Sheets (about 6 min)

▶ Click Firefox in the Dock. Click the Add-back Review tab at the bottom.

"Now I'll redo the numbers in Google Sheets, starting with the Add-back Review tab at the bottom."

▶ In column G, pick Yes from the dropdown for rows R-1 to S-2, and No for S-3.

"I'll mark each add-back using the dropdowns in column G. The first five are fine, so Yes. Dayton is No."

▶ Click the Covenant Test tab. Point at column B, then column C.

"On the Covenant Test tab, column B is the borrower's numbers and column C is mine."

▶ Click C13 and type `=0.25*C9`, then press Return.

"In C13 I'll set the cap the right way: 25 percent of 72. That's 18."

▶ Click C19 and type `=MIN(C18,15)`, then press Return.

"In C19, cash is capped at 15."

▶ Point at C24, then C26.

"Now leverage is 5.61 times. The max is 5.25, so they fail the covenant."

▶ Point at C27.

"Interest coverage is still fine, at 4.09."

▶ Click C36 and type `=ROUNDUP(C35,1)`, then press Return.

"To get back to 5.25, they need a cure of about 6.2 million. I'll round it up in C36."

▶ Point at C37.

"With the cure, leverage is 5.249. So it works."

▶ Click File, then Download, then Microsoft Excel.

"I'll download the workbook with File, Download, Microsoft Excel."

▶ Minimize Firefox.

"Minimizing Firefox."

---

## Part 4: Pages, the memo (about 4 min)

▶ Click Pages in the Dock.

"Last step: a short memo to my portfolio manager, in Pages."

▶ Click under Summary and type:
`They report 5.13x but it's really 5.61x vs a 5.25x max, so they're in breach.`

"In the summary I'll put the main result first."

▶ Click under Findings and type 3 short bullets, pressing Return after each:
`Add-back cap used the wrong base`
`Dayton savings don't qualify`
`Cash netting over the $15M limit`

"These are the three problems I found."

▶ Click under Recommendation and type:
`Ask for a corrected certificate. Sponsor can cure with $6.2M.`

"And my recommendation."

▶ Click File, then Export To, then PDF. Click Next and save.

"I'll export it as a PDF with File, Export To, PDF."

---

## Epilogue (about 3 min, the mouse can rest)

"So, the result. The borrower said they passed at 5.13 times. When I followed the agreement's rules, it's 5.61 times, so they're in breach. The sponsor can fix it with about 6.2 million of equity.

I checked my work by tying every number back to the rule I highlighted. The difference comes from three things: the cap was measured on the wrong number, the Dayton savings were too late, and they subtracted too much cash.

My approach was to read the rules first and then check their numbers, instead of just trusting their math.

The tricky part was that the problems overlap. Even after removing Dayton, the cap still limited the add-backs.

The weak spot is that I'm trusting the company's savings estimates. With more time, I'd ask for proof of those.

My tip for someone newer: always check what a percentage cap is based on. Here, that one detail decided pass or fail."

▶ Bring TaskRecorder forward and click Stop Recording. Wait for it to say Idle.
