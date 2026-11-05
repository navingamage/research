# Typst: reviewing an open layout PR

- **Repo:** https://github.com/typst/typst/tree/1f0cd84af6bbc4f612b80822f2da04fd4649a707 (head of PR #7931; base is `main` at `c98e910391a8544b28bd5c99a6f3b1ac1ada9a84`)
- **Stack:** Rust (typst-layout, typst-realize, typst-library)
- **Domain:** typesetting compiler, review only (no code changes)
- **Status:** not run yet
- **Full notes:** [codebase notes and facts for the write-ups](../top3/typst-pr7931-review/) (checked against the head and base on 2026-11-04)

**Pre-flight (checked)**
- No AI files and no AI co-authored commits in the repo.
- PR #7931, "Allow equation and tight lists/enums to be part of paragraphs" (closes #3206), is still open. It has 4 commits on top of `c98e9103` (2026-06-15) and changes 110 files: 11 source files (+370/−93) and 99 reference files (images and hashes).
- Workspace: fetch `pull/7931/head`, put `main` at the merge-base, and check out the PR branch.
- Re-checked 2026-11-04: the PR is still open and its head is still `1f0cd84a`.

**What the PR does**
- Adds an internal `inlinable` field on `block`. Block equations and tight lists/enums are marked inlinable.
- In `typst-realize`, inlinable blocks next to paragraph content are grouped into the same paragraph. `typst-layout` (flow collect and distribute, plus inline linebreak, line and finalize) then lays them out inside it.

**Problems reviewers found on the PR thread (the ground truth)**
- **HTML export:** lists end up inside `<p>`, which isn't valid HTML.
- **PDF tags:** a nested list moves under the paragraph in the tag tree instead of being a direct child of its parent list (saecki, citing the Tagged PDF best-practice guide §4.2.5.1; affects the `enum-number-override-nested` and `list-tags-complex` tests).
- **Widow/orphan prevention:** the `need` calculation in `flow/collect.rs` "around line 207" is wrong (lines 237-245 at the head). The author's own review comment: "very wrong currently (but does appear to be benign)".
- **Rendering changes:**
  - The PR re-records 96 reference renders (72 math, 24 list, enum, equation and counter ones) plus 117 PDF, 96 SVG and 8 PDF-tag hashes. HTML hashes are untouched.
  - `issue-6242-tight-list-attach-spacing` is not among them. The author lists it as a regression still to fix (tight-list spacing before the first item). It's the one test expected to fail at the head (not run).
  - A tight list followed by a paragraph turns non-tight.
- **Open design question:** whether a lone equation or two side-by-side equations should form a paragraph (accessibility arguments on the thread).

**Why it's hard**
- The PR re-records its own reference images and hashes, so most changes don't show as test failures. They only show by comparing renders before and after, or by checking HTML and PDF-tag output. The diff doesn't touch either exporter.
- The problems sit in other crates (HTML export, PDF tagging) from the ones the diff changes.

**Verification in the lane:** `cargo test --workspace --test tests` with before/after renders. `cargo testit --base-revision c98e9103` compares the head against the base refs, so every re-recorded change fails again and shows in the report (not run). A review can be checked item by item against the list above.
