# Task 1 narration script (simplified): covenant compliance test

**What changed:** the workbook now does the mechanical math for you. The LTM sums, EBITDA bridge, ratios, variance and formats are already built. Your job on screen is the judgment calls, made with the mouse through Yes/No dropdowns, plus 4 short entries. In Skim you only highlight; there are no typed notes. The memo template already has the header and a results table. In total you type about 3 minutes in a roughly 32-minute recording, and the rest is talking and clicking.

**How to read this:**
- **▶** is the action you take. **SAY:** is roughly what you say. Put it in your own words.
- ⌨️ marks typing. Every typing burst here is short.
- **What + Where every time:** name the thing you click and where it is on screen. Keep the cursor on it, and pause about a second after each click.
- **Window rule:** before you switch apps, minimize the one you're leaving with its yellow button and say so.
- Don't stay silent more than about 30 seconds. While something loads, say what you're expecting.

**Before you press Start:**
- Skim is open with both PDFs, at the top of page 1.
- Firefox has the Sheet open on the Financials tab.
- Pages has the memo template open.
- All three are minimized, and only the Goal doc is showing.

---

## 0. Intro [0:00 to 0:20]

▶ Click **Start Recording**, then minimize TaskRecorder.

**SAY:** "I will be beginning my task soon. Here is the workflow I have in mind."

▶ Wait about 3 seconds, then minimize the Goal doc with its yellow button.

---

## 1. Skim: credit agreement [0:20 to 7:00]

▶ Click **Skim** in the Dock and bring up **Credit_Agreement_Excerpt.pdf**.

**SAY:** "I'm on the agent side of Kestrel Ridge's term loan, and their Q3 compliance certificate just came in. Before I look at their numbers I want to go back to the credit agreement, because the certificate is only as good as how they've applied these definitions. I've got the excerpt open in Skim."

▶ Hover over the title block.

**SAY:** "It's a 480 million dollar term loan B plus a 75 million dollar revolver, and we're the administrative agent."

▶ In the toolbar at the top right, click the **Note Tool** dropdown and choose **Highlight**.

**SAY:** "I'll set Skim's note tool to Highlight, from the dropdown in the toolbar at the top right, so I can mark up the language that matters."

▶ Scroll to Consolidated EBITDA and point down the list from clause (i) to (iv).

**SAY:** "This is Consolidated EBITDA. Clauses one to four are the standard add-backs: interest, taxes, D&A and stock comp. Nothing unusual there. Clauses six and seven are where borrowers have room to get creative, so that's where I'll focus."

▶ Highlight "taken or initiated prior to the last day of such Test Period" in clause (vii).

**SAY:** "Clause seven lets them add back cost savings they haven't actually realized yet. The first guardrail is this phrase I'm highlighting: the action has to be taken or initiated before the end of the quarter."

▶ Highlight "within eighteen (18) months after the last day of such Test Period".

**SAY:** "The second guardrail is the eighteen-month window. The quarter ended September 30th, 2026, so anything that lands after March 31st, 2028 doesn't count."

▶ Highlight the bold proviso, "25.0% of Consolidated EBITDA ... (calculated before giving effect to any such add-backs ...)".

**SAY:** "And this proviso is the big one. Clauses six and seven together are capped at 25 percent of EBITDA, and the cap is measured on EBITDA *before* those add-backs. That detail matters a lot. If you measure it off EBITDA after the add-backs, the cap grows with the add-backs themselves. It's one of the most common places a borrower is generous with the math."

▶ Scroll to Consolidated Total Net Debt. Highlight "in an aggregate amount not to exceed $15,000,000".

**SAY:** "Consolidated Total Net Debt is debt minus unrestricted cash, but the cash they can net is capped at fifteen million. Even if they're holding more cash, only fifteen comes off."

▶ Point at "shall exclude undrawn amounts under letters of credit" in Consolidated Total Debt. Then point at Consolidated Cash Interest Expense.

**SAY:** "Two smaller points. Total debt excludes undrawn letters of credit, and interest coverage only uses cash interest, so the fee amortization comes out."

▶ Scroll to Section 7.11. Highlight the row "September 30, 2026 through June 30, 2027 | 5.25 : 1.00".

**SAY:** "Now the covenant itself, in Section 7.11. Max net leverage was 5.75 times, and it steps down to 5.25 starting this quarter. Step-down quarters are exactly when a borrower is tempted to stretch the adjustments, so this is the quarter to be careful on."

▶ Point at 7.11(b).

**SAY:** "Interest coverage has to stay above 2.25 times. That one is rarely the issue."

▶ Scroll to Section 8.04 and point at the heading.

**SAY:** "There's an equity cure in 8.04 as well. I'll come back to it if they turn out to be out of compliance."

---

## 2. Skim: compliance certificate [7:00 to 11:30]

▶ Click Skim's **Window** menu in the menu bar and choose **Q3_2026_Compliance_Certificate.pdf**.

**SAY:** "Now the borrower's certificate. I'll switch to it from Skim's Window menu at the top."

▶ Point at the certification paragraph, then at the sentence about the prior cure.

**SAY:** "Their CFO is certifying compliance as of September 30th. They also mention one cure used so far, for the quarter ended June 2025. I'll keep that in mind for the cure question later."

▶ Scroll to Schedule 1. Point at the 96.0 EBITDA, then at 5.13x in the covenant table.

**SAY:** "On Schedule 1 they show EBITDA of 96 and leverage of 5.13 times against the 5.25 maximum. That's only about a tenth of a turn of headroom on a step-down quarter, so I'm not going to just tick and tie this. I'll rebuild it."

▶ Highlight "Limit: 25.0% × Consolidated EBITDA (96.0)".

**SAY:** "Here's the first problem. They've taken 25 percent of 96, but 96 already includes the add-backs. The agreement says to measure it before the add-backs, which is 72. So their cap is 24 when it should be 18."

▶ Highlight "(28.0)" in the net debt table.

**SAY:** "Second problem: they've netted all 28 million of cash. The cap is 15, so net debt is understated by 13 million."

▶ Scroll to Schedule 2 and point down rows R-1 to S-2.

**SAY:** "Schedule 2 lists the add-backs. R-1 to R-3 are restructuring costs, which are fine under clause six. S-1 and S-2 are cost savings that were already started and land in 2027, so they're inside the window."

▶ Highlight the whole S-3 row.

**SAY:** "But S-3, the Dayton plant closure, fails on both tests. Board approval isn't expected until late 2027, so nothing was started by quarter end. And the savings land in Q2 2028, after the March 2028 cutoff. That three million comes out."

▶ Click **File** in the menu bar, then **Save**.

**SAY:** "I'll save my highlights with File, Save."

---

## 3. Google Sheets: add-back review [11:30 to 14:30]

▶ Minimize Skim with its yellow button. Click **Firefox** in the Dock. The Sheet opens on the **Financials** tab.

**SAY:** "I'll minimize Skim with the yellow button and open Firefox from the Dock. I've got the borrower's financials in Google Sheets, with four quarters of P&L and the September 30th balances."

▶ Point at column F (LTM), then move down F6, F7, F8, F10, F11 and F12, and stop at F13.

**SAY:** "The LTM column on the right sums the last four quarters. Net income is 18, interest 24 with 22 of that in cash, taxes 6, D&A 22, stock comp 2 and restructuring 8. All of those tie to their certificate. So the raw numbers are fine, and the issue is the adjustments."

▶ Point at the balance sheet block, rows 18 to 23.

**SAY:** "Underneath are the balances: 520 of debt across the term loan, revolver and leases, 6 of undrawn LCs that I'll leave out, and 28 of cash."

▶ Click the **Add-back Review** tab at the bottom.

**SAY:** "Now the Add-back Review tab at the bottom. I've put the March 31st, 2028 cutoff at the top so the eighteen-month test is right in front of me."

▶ Click **G5** and pick **Yes** from the dropdown. Do the same for G6, G7, G8 and G9.

**SAY:** "I'll go down the Eligible column using the dropdown in each yellow cell. R-1 severance is completed restructuring, so Yes. R-2 lease exit, Yes. R-3 relocation is still in progress, but clause six doesn't need it to be finished, so Yes. S-1 procurement was started in Q2 and lands in Q2 2027, inside the cutoff, so Yes. S-2 headcount, same reasoning, Yes."

▶ Click **G10** and choose **No**.

**SAY:** "S-3 Dayton is No, for the two reasons I flagged on the certificate."

▶ Point at H11, then F11.

**SAY:** "So 21 of add-backs are eligible, against the 24 they claimed."

---

## 4. Google Sheets: covenant recalculation [14:30 to 21:00]

▶ Click the **Covenant Test** tab.

**SAY:** "On to the Covenant Test tab. Column B has the borrower's numbers from their certificate, column C is my recalculation, and the yellow cells are where the calls get made."

▶ Point down C4 to C12.

**SAY:** "The top of column C pulls straight from the financials and the add-back review. EBITDA before the add-backs is 72, restructuring is 8, and eligible cost savings are 13 rather than their 16. Combined that's 21 before any cap."

▶ Click **C13**. ⌨️ Type `=0.25*`, click **C9**, and press Return.

**SAY:** "In C13 I'll put the cap the way the agreement defines it: equals 0.25 times, click C9, which is the pre-add-back EBITDA, and press Return. That gives 18."

▶ Point at C14, then C15.

**SAY:** "The allowed add-backs are the lower of 21 and 18, so 18. Even after taking Dayton out, the cap still bites. The cap base is really what's driving this. That puts Consolidated EBITDA at 90, against their 96."

▶ Click **C19**. ⌨️ Type `=MIN(`, click **C18**, type `,15)`, and press Return.

**SAY:** "Next, cash netting in C19: the lower of the actual cash in C18 and the 15 million cap. That gives 15."

▶ Point at C20.

**SAY:** "So net debt is 505, against their 492."

▶ Click **C25**. ⌨️ Type `5.25` and press Return.

**SAY:** "In C25 I'll enter the maximum for this quarter from the step-down table, which is 5.25."

▶ Point at C24, then C26.

**SAY:** "Net leverage comes out at 5.61 times, and the compliance check in C26 says No. They're in breach of the leverage covenant, and not by a rounding difference. It's 5.61 against 5.25."

▶ Point at C27 and C29.

**SAY:** "Interest coverage is 4.09 times, comfortably over 2.25, so that one passes."

▶ Point at C31, then go down column D and stop at D15, D19 and D20.

**SAY:** "The cushion row shows they're about 6.2 million of EBITDA short, where they were showing 2.3 million of room. And the variance column tells the whole story in three lines: EBITDA down 6, cash netting down 13, net debt up 13. Everything else ties."

---

## 5. Skim: cure conditions [21:00 to 22:30]

▶ Minimize Firefox. Click **Skim** in the Dock and bring up the credit agreement. Scroll to **Section 8.04**.

**SAY:** "Since they're out, the next question is whether the sponsor can cure. I'll minimize Firefox and go back to Section 8.04 in Skim."

▶ Highlight "solely for the purpose of determining compliance with Section 7.11(a)".

**SAY:** "A cure adds to EBITDA, but only for the leverage test."

▶ Point at (b)(i) and (b)(ii).

**SAY:** "There can be no more than two cures in any four quarters, and none in back-to-back quarters. Their only earlier cure was June 2025, which is outside the current four quarters and not consecutive. So they're fine, and this would be cure two of five."

▶ Highlight (b)(v).

**SAY:** "And this one matters: the cure money can't also be used to pay down debt or be netted as cash. Net debt stays at 505."

---

## 6. Google Sheets: cure sizing and download [22:30 to 25:00]

▶ Minimize Skim. Click **Firefox** in the Dock and go back to the Covenant Test tab.

**SAY:** "Minimizing Skim and going back to the sheet in Firefox."

▶ Point at C34 and C35.

**SAY:** "To get to 5.25 they need EBITDA of 96.19, so the minimum cure is 6.19 million."

▶ Click **C36**. ⌨️ Type `=ROUNDUP(`, click **C35**, type `,1)`, and press Return.

**SAY:** "I'll round it up to the next hundred thousand in C36 with ROUNDUP, which gives 6.2. A 6.19 cure would leave them a hair over 5.25."

▶ Point at C37.

**SAY:** "Checking it: the pro forma ratio is 5.249 times, so it clears."

▶ Click **C38** and choose **Yes** from the dropdown.

**SAY:** "And the cure is permitted, so Yes from the dropdown."

▶ Click **File → Download → Microsoft Excel (.xlsx)**.

**SAY:** "I'll download a copy with File, Download, Microsoft Excel."

---

## 7. Pages: credit memo [25:00 to 30:00]

▶ Minimize Firefox. Click **Pages** in the Dock.

**SAY:** "Last step is the memo to my portfolio manager. I'll minimize Firefox and open the template in Pages. The header's already set up."

▶ Click under **Summary**. ⌨️ Type: `Certificate shows 5.13x but I get 5.61x vs a 5.25x max, so they're in breach of 7.11(a). Coverage still passes.`

**SAY:** "Summary first, with the conclusion up front: they're actually at 5.61 against 5.25, so it's a breach."

▶ Click the bullet under **Findings**. ⌨️ Type these three bullets, pressing Return between them:
- `Add-back cap based on post-add-back EBITDA. Should be 25% of 72 = 18`
- `Dayton savings ($3M) ineligible: not started, lands after 3/31/28`
- `Netted $28M of cash vs $15M cap`

**SAY:** "Three findings: the cap base, Dayton, and the cash netting."

▶ Click into the **Covenant Results** table and fill each cell, using Tab to move between them. Say "Tab" the first time you use it.
- EBITDA row: `96.0` | `90.0` | (n/a) | `Overstated`
- Leverage row: `5.13x` | `5.61x` | (≤ 5.25x) | `Breach`
- Coverage row: `4.36x` | `4.09x` | (≥ 2.25x) | `Pass`

**SAY:** "In the results table I'll fill in their numbers and mine side by side, pressing Tab to move between cells."

▶ Click under **Recommendation**. ⌨️ Type: `Send a reservation of rights letter and ask for a corrected certificate. Sponsor can cure with at least $6.2M by about Dec 7. Move to watch list.`

**SAY:** "Recommendation: reserve our rights, ask for a corrected certificate, note that the sponsor can cure with 6.2 million by early December, and put the name on the watch list."

▶ Scroll through the memo with the cursor for about 10 seconds.

**SAY:** "A quick read-through to check the numbers match the workbook."

▶ Click **File → Export To → PDF…**, click **Next**, and save to Documents as `Kestrel_Ridge_Credit_Memo`.

**SAY:** "File, Export To, PDF, Next, and I'll save it to Documents."

▶ Minimize Pages. Click **Skim** in the Dock. For each of the two PDFs, click **File → Export…**, choose **PDF With Embedded Notes**, and save.

**SAY:** "Last thing. Skim stores highlights separately by default, so I'll export both PDFs with File, Export, PDF With Embedded Notes. That way the markups go with the files."

---

## 8. Epilogue [30:00 to 33:30] (the mouse can rest)

**SAY:**

"To wrap up: I tested Kestrel Ridge's Q3 compliance certificate independently, and it overstates compliance. On the agreement's own definitions, leverage is 5.61 times against a 5.25 max, so they're in breach. The sponsor would need to put in at least 6.2 million to cure.

To check it, every number in my workbook goes back to a definition I highlighted, and the variance column explains the whole gap: 6 million of EBITDA and 13 million of net debt. I also confirmed the cure actually works, at 5.249 times.

My approach was to read the definitions first and only then look at their certificate, and to rebuild the numbers rather than follow their layout. That's how the cap base issue stands out.

The tricky part was that two problems overlap. Dayton is ineligible, but even without it the cap still binds, so the cap base is what really moves the number. I wanted the memo to be clear about which one drives the result.

On quality, I'm comfortable with the conclusion. The limitation is that I'm relying on management's schedules. I don't have support for the S-1 and S-2 savings, so with more time I'd ask for that backup and run a sensitivity on how much more EBITDA could slip.

And one tip for someone newer: always check what a percentage cap is measured against. Pre-add-back versus post-add-back sounds like a detail, but here it's the difference between passing and breaching."

▶ Bring TaskRecorder to the front, click **Stop Recording**, and wait for **Idle**.

---

## Dry-run notes
- Time each section. If you're running long, shorten the SAY lines in sections 1 and 2. If you're short, explain more of the "why" while pointing. Talking is what you want here.
- Before recording, check that the Yes/No dropdowns came through after importing the workbook into Google Sheets. If they didn't, just type Yes or No in those cells.
- Shortcuts: the only one is Tab in the Pages table. Say it out loud.
- If you mistype, say "let me fix that," fix it, and keep going.
