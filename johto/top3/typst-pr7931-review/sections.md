# Typst PR #7931 review: facts for the four write-ups

Facts, numbers and open choices only, as fragments. The write-ups themselves get written separately.

## 1. The ask

### Domain context
- **Typst:** markup-based typesetting compiler; Rust workspace 0.15.0, edition 2024, MSRV 1.92; 379 `.rs` files, 146,324 lines under `crates/` at base.
- **Pipeline:** syntax, eval, realize (show rules, grouping into paragraphs and lists), layout (flow + inline), export.
- **Outputs:**
  - paged: PNG (`typst-render`), PDF (`typst-pdf`, tagged by default, `crates/typst-pdf/src/lib.rs:107`), SVG (`typst-svg`)
  - HTML (`typst-html`): experimental, error without `--features html`, warning with it (`crates/typst/src/lib.rs:246-266`)
  - bundle (`typst-bundle`): feature-gated
- **Paragraph model as documented** (`crates/typst-library/src/model/par.rs`): inline-level = text, `h`, `box`, inline equations (17-19); interrupted by any block-level element (22-23); `#par[..]` body without block-level content (26-28); PDF `P` and HTML `<p>` only for paragraphs (67, 69); no first-line indent right after block-level elements (379-381).
- **Target:** typst/typst PR #7931 "Allow equation and tight lists/enums to be part of paragraphs", closes #3206, author mkorje, open; head `1f0cd84a`, base `c98e9103` ("Bump version on main", 2026-06-15).

### Starting facts about the PR
- **Commits (4):** `e60dfa1d` "Create inlinable blocks"; `6e6a5211` "amend"; `77c83e25` "Make tight lists and enums inlinable"; `1f0cd84a` "Make block equations inlinable". All committed 2026-06-29 (rebased); author dates 2026-01-23, 2026-06-29, 2026-02-19, 2026-02-19.
- **Size:** 110 files, +591/-314.
  - 11 source files, +370/-93: typst-layout 8 (+313/-87), typst-realize 2 (+51/-6), typst-library 1 (+6/-0)
  - 99 reference files: 96 PNGs + 3 hash files (PDF 117 entries, SVG 96, PDF-tag 8)
  - 0 test files, 0 docs files, 0 HTML refs; no typst-html or typst-pdf change
- **What the PR does (title + diff):**
  - internal `inlinable` field on `block` (head `crates/typst-library/src/layout/container.rs:390-394`, `#[internal]`, `#[parse(None)]`)
  - block equations, tight lists and tight enums marked inlinable in the paged show rules only (head `crates/typst-layout/src/rules.rs:129`, 144, 800)
  - realize: inlinable block = paragraph trigger (head `crates/typst-realize/src/lib.rs:1071`); a lone one stays a block (`is_tentative_par`, 1206-1221)
  - flow and inline layout place the block inside the paragraph, `par.leading` above and below (head `crates/typst-layout/src/flow/collect.rs:227-229`)
- **Unknown (not in the repo):** PR description text; issue #3206 text; exact thread wording behind (e); whether (d) is listed in the description or a comment.

### Requirement topics
- paragraph membership rules in realize; target independence of the change
- layout of inline blocks: spacing, line breaking, widow/orphan `need`, tag placement
- spacing semantics users rely on: `list.spacing`, `block.above/below/spacing`, list attachment, first-line indent
- export consistency: HTML `<p>` content model; PDF tag tree (`P`, `L`, `LI`, `LBody`, `Formula`)
- crash safety: `unreachable!()` assumption in `commit` (head `crates/typst-layout/src/inline/line.rs:533`)
- test hygiene: 96 PNG + 117 PDF + 96 SVG + 8 PDF-tag refs self-recorded; 1 affected ref left untouched (`issue-6242`); 0 new tests
- docs contract: paragraph and block-spacing docs vs new paged behaviour
- open policy question: lone vs paired display equations as paragraphs

### Open choices
- **[decide] outputs the brief names** (one ladder; each rung's giveaway):
  - PR link only
  - "paged output": still contains tag changes (default PDF tagged, `crates/typst-pdf/src/lib.rs:107`; test PDFs use default options, `tests/src/output.rs:349-355`; 21 of the 117 PDF hash changes are tag-only with unchanged renders, e.g. `list-attached`, `tests/suite/model/list.typ:267`)
  - "PDF tags" / "pdftags": gives away (b)
  - "HTML": gives away (a)
  - crates outside the diff (`typst-html`, `typst-pdf`): gives away (a) and (b)
  - test names (`issue-6242-tight-list-attach-spacing`, `enum-number-override-nested`): give away (d) and (b)
- **Fact for the ladder:** 2 of 6 thread problems ((a), (b)) visible only in exporters outside the diff; (a): no HTML output change at head, cause in typst-realize (inside the diff).
- **[decide] execution:** read-only review / build and run tests with before/after outputs / plus new probe tests.
  - `cargo testit --base-revision c98e9103`: refs read from git at base, no work-tree change (`tests/src/args.rs:62-66`; `tests/src/tests.rs:96-110`; `tests/src/run.rs:810-815`)
  - `cargo testit regen`: work tree rewritten with `git read-tree -um` (`tests/wrapper/src/main.rs:215-221`), refuses a dirty tree (93-98)
  - no execution: `git show main:<png>` vs `git show pr-7931:<png>` for re-recorded PNGs only
- **[decide] sources shown, with what each discloses:**
  - PR description: content unknown; may hold the `issue-6242` regression note (d)
  - author's inline comments: (c) ("very wrong currently (but does appear to be benign)"); possibly (d)
  - PR discussion thread: (a), (b) (saecki, Tagged PDF guide §4.2.5.1), (e), (f)
  - issue #3206: the PR's goal; text unknown; `CONTRIBUTING.md:55-58` (alignment with prior discussion)
- **[decide] PR's public page (description, thread) off-limits:** yes / no; if no: (a) to (f) readable there.
- **[decide] #3206 text in the brief:** yes / no.
- **[decide] (a) accepted forms:** literal thread claim (lists inside `<p>`) / conditional form (HTML would be invalid or diverge if it followed paged) / both.
- **[decide] (e) counted as:** thread wording / code-level spacing cases / left out.
- **[decide] (f) counted as:** raised as a question / left out (policy, no single right answer).
- **[decide] extras (not in the thread):** in scope or not; a wrong extra counted as a failure or not.
- **[decide] (d) "test is red":** required only if execution is allowed; otherwise "inferred, not run".
- **[decide] deliverable:** findings list / inline comments; kept local, never posted to the live PR; merge verdict (approve / request changes) required yes / no; `path:line` anchors required yes / no; severity labels yes / no.

## 2. Goal and success checks

### Context for a domain expert new to this repo
- **Realize:** content grouped by `GroupingRule`s with effects Trigger / Inner / Neutral / Interrupt; priorities TEXTUAL 3, CITES/LIST/ENUM/TERMS 2, PAR 1; PAR collects inline-level content into `ParElem`; PAR style interrupts `ParElem`, `AlignElem` (base `crates/typst-realize/src/lib.rs:1046-1073`).
- **Elements:** `ParElem` = paragraph; `BlockElem` = block-level container; lists and display equations become `BlockElem`s through show rules.
- **Show rules per target:** `NativeRuleMap` keyed by `(Element, Target)`, no fallback (`crates/typst-library/src/foundations/styles.rs:985-1070`); paged rules `crates/typst-layout/src/rules.rs`; HTML rules `crates/typst-html/src/rules.rs`; realize itself is shared (`FLOW_RULES`, head `lib.rs:1025`).
- **Show-set styles:** user show-set and built-in `ShowSet` styles collected into one map (`lib.rs:562-564`); equations set `AlignElem` CENTER (`crates/typst-library/src/math/equation.rs:190`), a PAR interrupt at base.
- **Tight list:** `tight` = no parbreak between items (base `lib.rs:1226`); base paged rule prepended a weak "attach" `v` (spacing = `list.spacing` or leading) that survives only right after a `ParElem` (base `crates/typst-layout/src/rules.rs:131-138`; `lib.rs:774-784`).
- **Spacing ladder (flow):** lower weakness wins, ties take the larger amount: strong `v` 0; weak `v`/attach 1; block Fr 2; block custom 3; block auto + paragraph spacing 4; leading 5 (`crates/typst-layout/src/flow/distribute.rs:185-210`). Defaults: `par.leading` 0.65em, `par.spacing` 1.2em, block spacing 1.2em.
- **`ParSituation`:** First / Consecutive / Other; drives first-line indent (`crates/typst-layout/src/inline/mod.rs:197-215`).
- **Inline layout:** objects as U+FFFC in the paragraph text; UAX #14 breaks via `icu_segmenter` 2.2.0; line box = tallest item.
- **Introspection tags:** start/end tags around Locatable/Tagged elements, placed in frames; PDF tag tree built from their order (`crates/typst-pdf/src/tags/tree/build.rs:1-13`).
- **PDF list structure:** `L[LI[Lbl, LBody]]`; nested `L` expected beside the `LI`s (`crates/typst-pdf/src/tags/context/list.rs:47-79`); `ParElem` becomes `P` (`crates/typst-pdf/src/tags/resolve/mod.rs:313`).
- **HTML:** `ParElem` becomes `<p>` (`crates/typst-html/src/rules.rs:94-95`); `<p>` takes phrasing content only (`crates/typst-html/src/tag.rs:509-546`).
- **Tests:** `tests/suite/**/*.typ`, `--- name attrs ---` headers, 3677 tests; PNG refs are files (1915), PDF/SVG/PDF-tag/HTML refs are hashes in `tests/ref/*/hashes.txt`; readable output only in `tests/store/` after a run; `approx_equal` needs equal width and height (`tests/src/output.rs:638-644`); test page 120pt wide, auto height, 10pt text, 1 px per pt (`tests/src/world.rs:190-196`; `tests/src/output.rs:193`).
- **Runner:** `cargo testit` alias; default comparison against working-tree refs; `--base-revision <rev>` against refs at a git revision; `regen` for missing old live output of hashed refs.

### Success checks (one outcome each, pass or fail)
- **(a) HTML, pass:** paged/HTML split named: tight lists (and display equations) inside paragraphs in paged output; HTML unable to follow (`<ul>`/`<ol>` not phrasing content, `crates/typst-html/src/tag.rs:509-546`); consequence: target divergence or invalid HTML. Accepted forms per [decide].
  - bonus: target-agnostic PAR trigger (head `crates/typst-realize/src/lib.rs:1071`); unflagged HTML rules (`crates/typst-html/src/rules.rs:103-147`, 822-841); paged-only `show par` reach
- **(b) PDF tags, pass:** structure change for a nested tight list after item text named: nested `L` under `P` inside `LBody` instead of beside the `LI`s in the outer `L`.
  - bonus: `push_body` last-node check (`crates/typst-pdf/src/tags/context/list.rs:80-86`); ParElem as weak Par group (`crates/typst-pdf/src/tags/tree/build.rs:512-513`); no `P` child validation (`crates/typst-pdf/src/tags/resolve/mod.rs:410-451`)
  - the 8 re-recorded PDF-tag hashes: supporting evidence only
- **(c) Widow/orphan, pass:** `par_children` (head `crates/typst-layout/src/flow/collect.rs:189-292`): line indexing across inline blocks, block height ignored; plus one concrete wrong shape (`[L0, B, L1]`, `[L0, L1, B, L2]`, `[L0, L1, L2, B, L3]`) or the per-run fix.
  - bonus: why benign (`need` read only at head `crates/typst-layout/src/flow/distribute.rs:285-293`)
- **(d) issue-6242, pass:** A-to-B gap = `par.leading` via `Child::Rel(leading, 5)` (head `crates/typst-layout/src/flow/collect.rs:227-229`) instead of `list.spacing`; cause: deleted attach `v` (base `crates/typst-layout/src/rules.rs:131-138`, head 124-152); refs untouched (PNG last changed `f312df7b`; `tests/ref/pdf/hashes.txt:810`, `tests/ref/svg/hashes.txt:808`).
  - "red test": only if execution allowed, else tagged inferred
  - bonus: generalized to any attached tight list under `set list(spacing: ..)` / `set enum(spacing: ..)`
- **(e) Tight list + paragraph, pass (code-level reading):** at least one spacing direction with its mechanism, tight flag left unchanged:
  - tighter: text right after a tight list joins the paragraph at leading (head `flow/collect.rs:227-229`); base 1.2em
  - looser: custom `block` below on lists ignored after an attached list; `par.spacing` at the paragraph end (head `flow/collect.rs:178-182`) instead of base `block()` below (base 246-251)
  - a ref name counts only with a before/after judgement
- **(f) Design question, pass:** lone-vs-paired rule raised (one display equation a block; two adjacent ones, or any block pair, a paragraph) + semantics/accessibility stakes + one consequence:
  - PDF `P[Formula, Formula]` vs a bare `Formula`; HTML no `<p>` either way
  - 1.2em to 0.65em between equations
  - block + block pairs already re-recorded: equation + list (`issue-1850-list-attach-spacing`, `tests/suite/model/list.typ:568-586`); list + enum (`list-mix`, `tests/suite/model/enum.typ:19-23`); four lists (`set-instantiation-site-markup`, `tests/suite/styling/set.typ:8-19`)
  - bonus: names `is_tentative_par` (head `crates/typst-realize/src/lib.rs:1206-1221`)
- **Extras, only if [decide] puts them in scope** (each: not in the thread, not confirmed by maintainers, not run):
  - E2 crash: input with no UAX #14 break before U+FFFC (`~`, `(`) or a repeated hard hyphen (lang pl/pt/cs/hr/sk/dsb, es) reaching `_ => unreachable!()` (head `crates/typst-layout/src/inline/line.rs:533`; 175-184, 447-471; head `crates/typst-layout/src/inline/linebreak.rs:733-735`)
  - E1 block spacing ignored: `crates/typst-library/src/layout/container.rs:333-340` docs example; head `flow/collect.rs:227-229` vs 351-358
  - E10 attachment lost after `#par[..]`, `#align(..)[..]` or a `set par`/`set align` (head `crates/typst-layout/src/rules.rs:663-664`; head `flow/collect.rs:351-358`)
  - E11 paragraph membership depends on show-rule form (head `crates/typst-realize/src/lib.rs:820-825`, 1223-1230)
  - E3 indent-only empty line (head `crates/typst-layout/src/inline/collect.rs:141-149`)
  - E4 start tag on the previous line; end tag on the first region only (`crates/typst-layout/src/inline/prepare.rs:45-58`; head `distribute.rs:349-352`)
  - E5 `issue-1850` ref contradicts its comment (`tests/suite/model/list.typ:569`)
  - E6 terms left out (head `rules.rs:196-205`; `list-mix`)
  - E7 indent after equations/lists (head `flow/collect.rs:183` vs base 279; `par.rs:379-381`)
  - E8 `show par` reaches lists/equations, paged only
  - E9 `#par[..]` now lays out inlinable blocks (head `inline/collect.rs:240-244` vs base 236-243)
  - E12 `sticky`, `alone`, `fr` hard-coded (head `flow/collect.rs:254-288`)
  - E13 docs contract (`par.rs:22-28`, 379-381; `container.rs:333-340`; 0 docs files)
  - E14 no PDF-tag test with a display equation next to text
- **False-positive guards:** no "tight flag flips" (`finish_list_like` identical, base `lib.rs:1222-1244`, head 1264-1286); no "suite green"; no "HTML already emits `<ul>` inside `<p>`" as a head fact unless [decide] (a) accepts the literal form (no HTML diff; HTML hashes unchanged).

### Shortcuts that must not count
- "suite/CI green" (refs self-recorded; `issue-6242` expected red, not run)
- "refs updated, so fine"
- listing the 96 PNGs, or any ref name, with no before/after judgement
- style or naming nits only ("amend", duplicated `lines()`)
- approval based on the PR description
- generic "add tests", "check HTML", "pagination may break", "accessibility concern" with no mechanism
- quoting the TODO at head `flow/collect.rs:198-200` with no defect
- "PDF-tag hashes changed" with no defect named
- restating the PR title for (f)
- critique of `e60dfa1d`'s `tentative: bool` design (removed by `6e6a5211`)

## 3. Decision points

### DP1. Re-recorded refs
- **Fork:** treatment of the 96 PNG / 117 PDF / 96 SVG / 8 PDF-tag refs re-recorded in the PR.
- **Paths:** before/after comparison against base / acceptance as the new truth.
- **Code-level reason:** runner compares against working-tree refs by default (`tests/src/args.rs:62-66`); `--base-revision` reads base refs from git (`tests/src/tests.rs:96-110`, `tests/src/run.rs:810-815`); CI adds `--base-revision` only on green PR runs (`.github/workflows/ci.yml:64-66`, 81-88); `approx_equal` needs equal dimensions (`tests/src/output.rs:638-644`); 95 of 96 PNGs shorter, -1016 px total, one gap = 5.5 px. If execution is not allowed: `git show` base vs head PNGs, plus code reasoning.
- **Second path breaks:** every spacing change invisible; `issue-6242` looks like the only symptom.
- **Behaviour:** verification.
- **Findings it gates:** (d), (e), (f), E1, E5, E6.

### DP2. Reading scope beyond the 11 files
- **Fork:** whether the changed paragraph model is followed into its consumers outside the diff.
- **Paths:** consumers traced (typst-html rules and `<p>` model, typst-pdf tag tree) / diff-only reading.
- **Code-level reason:** PAR trigger target-agnostic (head `crates/typst-realize/src/lib.rs:1071`; `FLOW_RULES` 1025 for both targets); HTML rules unflagged (`crates/typst-html/src/rules.rs:103-147`, 822-841); `<p>` phrasing only (`crates/typst-html/src/tag.rs:509-546`); list hoist checks only the last `LBody` node (`crates/typst-pdf/src/tags/context/list.rs:80-86`); `git diff main pr-7931 -- crates/typst-html crates/typst-pdf` empty.
- **Second path breaks:** (a) and (b) missed; paged/HTML split (E8) and tag coverage gap (E14) missed.
- **Behaviour:** scope.
- **Findings it gates:** (a), (b), E8, E14.

### DP3. Spacing: per-ref symptoms vs one mechanism
- **Fork:** how the 95 shrunk refs and the spacing complaints are explained.
- **Paths:** one mechanism traced / per-ref observations.
- **Code-level reason:** single cause: `Child::Rel(leading, 5)` around inline blocks (head `crates/typst-layout/src/flow/collect.rs:227-229`) with `block()` spacing bypassed (head 340-387), plus the deleted attach `v` (base `crates/typst-layout/src/rules.rs:131-138`, 150-157); paragraph membership decided in realize (head `crates/typst-realize/src/lib.rs:1071`, 1206-1221).
- **Second path breaks:** dozens of separate complaints; misses that `block.above/below/spacing` and `list.spacing` lose effect inside paragraphs.
- **Behaviour:** root cause vs symptom.
- **Findings it gates:** (d), (e), E1, E10, E12.

### DP4. Under-specified thread items (e) and (f)
- **Fork:** handling of the two items whose exact meaning the code can't settle.
- **Paths:** ambiguity named, cases separated or a question asked / one interpretation asserted.
- **Code-level reason:** tight flag logic byte-identical (base `crates/typst-realize/src/lib.rs:1222-1244`, head 1264-1286); spacing moves in two directions (leading 0.65em vs `par.spacing` 1.2em; head `crates/typst-layout/src/flow/collect.rs:227-229`, 178-182); (f) is a policy stated in the `is_tentative_par` doc comment (head `lib.rs:1206-1212`), with `P[Formula, Formula]` in PDF and no `<p>` in HTML.
- **Second path breaks:** false positive "tight flag flips"; a policy question presented as a bug with one right answer.
- **Behaviour:** clarifying vs guessing.
- **Findings it gates:** (e), (f).

### DP5. Observed vs inferred, and tools in a read-only checkout
- **Fork:** reporting results that were not executed; choice of comparison tools when the work tree must stay untouched. Conditional on [decide] execution.
- **Paths:** labelled "inferred, not run" plus read-only tools (`git show`, `--base-revision`) / stated as observed plus `cargo testit regen`.
- **Code-level reason:** `issue-6242` failure is inferred (PNG 120x82 last changed `f312df7b`; hash lines `tests/ref/pdf/hashes.txt:810`, `tests/ref/svg/hashes.txt:808` untouched); E2 panic is a code path only (head `crates/typst-layout/src/inline/line.rs:533`); `regen` runs `git read-tree -um` (`tests/wrapper/src/main.rs:215-221`) and exits on a dirty tree (93-98).
- **Second path breaks:** unverifiable "fails"/"panics" claims; a modified work tree in a read-only checkout.
- **Behaviour:** honesty; following instructions.
- **Findings it gates:** (d), E2, E4.

### DP6. Repo conventions: tests, docs, CI
- **Fork:** whether the PR is checked against the repo's own contribution rules, beyond code correctness.
- **Paths:** conventions checked (tests, docs, CI) / code-only verdict.
- **Code-level reason:** 0 `tests/suite` files changed; `CONTRIBUTING.md:64-78` (tests with reference output, docs) and 50-53 (green CI before merge, first maintainer look usually after green); paged behaviour now contradicts `crates/typst-library/src/model/par.rs:22-28`, 379-381 and the `crates/typst-library/src/layout/container.rs:333-340` example; docs CI only compiles (`.github/workflows/ci.yml:112-113`); `issue-1850` ref contradicts its comment (`tests/suite/model/list.typ:569`); `par-contains-block` (`tests/suite/model/par.typ:487-493`) never covered inlinable blocks.
- **Second path breaks:** misses E5, E9, E13, the no-new-tests gap and the red-CI blocker.
- **Behaviour:** repo conventions.
- **Findings it gates:** E5, E9, E13, (d) CI side.

## 4. Difficulty and codebase understanding

### What the system is
- Typst compiler: Rust workspace 0.15.0; 17 crates under `crates/` plus `docs`, `tests`, `tests/fuzz`, `tests/wrapper`; 379 `.rs` files, 146,324 lines under `crates/`.
- Crate sizes (lines, base): typst-library 69,185 (176 files); typst-layout 21,969 (49); typst-pdf 7,997 (28; tags 4,902); typst-html 7,513 (18); typst-realize 1,656 (2).
- Tests: 222 suite files, 3677 tests; 1915 PNG refs; hash files pdf 1922, svg 1915, html 479, pdftags 105 lines; attributes paged 2109, eval 1267, html 497, pdftags 126.
- CI: test matrix ubuntu/windows 64-bit + ubuntu 32-bit, toolchain 1.95.0, MSRV 1.92, warnings as errors; `cargo test --workspace --no-fail-fast` (`.github/workflows/ci.yml:59`).

### What the task touches
- **In the diff (lines base to head):** `crates/typst-realize/src/lib.rs` 1534 to 1576; `spaces.rs` 122 to 125; `crates/typst-layout/src/flow/collect.rs` 718 to 841; `flow/distribute.rs` 639 to 660; `inline/line.rs` 852 to 918; `inline/linebreak.rs` 1039 to 1040 (+2/-1 ignoring whitespace); `inline/collect.rs` 338 to 348; `inline/mod.rs` 323 to 337; `inline/finalize.rs` 35 to 33; `rules.rs` 818 to 811; `crates/typst-library/src/layout/container.rs` 686 to 692.
- **Affected, outside the diff:** typst-html (`rules.rs` 851, `tag.rs` 629, `convert.rs` 697, `fragment.rs` 168 lines); typst-pdf tags (`context/list.rs` 102, `tree/build.rs` 811, `tree/mod.rs` 624, `resolve/mod.rs` 559, `groups.rs` 649); `crates/typst-layout/src/lists.rs` 645; docs inside `par.rs` and the `container.rs` example; tests and refs.
- **Flows:**
  1. realize: PAR trigger, tentative paragraph, show-set interrupt skip, space collapse
  2. inline: collect `Item::Block`, forced break, `commit` to `ParChild::Block`
  3. flow: `par_children`, inline `SingleChild`/`MultiChild`, `push_tags`
  4. export: HTML `<p>`, PDF list hoist and `P`
  5. tests: PNG files, hashes, runner, CI report

### Why it's hard (mechanisms)
- **Self-recorded refs:** 96 PNG + 117 PDF + 96 SVG + 8 PDF-tag refs re-recorded in the PR; default comparison against working-tree refs (`tests/src/args.rs:62-66`); report only on failures (`tests/README.md:86-90`); near-green run with 1 expected red (`issue-6242`, not run).
- **Dimension check:** `approx_equal` equal width and height (`tests/src/output.rs:638-644`); the 95 shortened PNGs = exactly the changes the re-recording hid.
- **Hashes, not files:** PDF, SVG, PDF-tag and HTML refs are hashes (`tests/src/output.rs:177-178`); no readable diff in the PR; YAML/HTML only in `tests/store/` after a run.
- **Tags inside default PDFs:** `tagged: true` (`crates/typst-pdf/src/lib.rs:107`); 21 PDF-only hash changes with identical renders.
- **HTML untouched by construction:** 0 of 479 HTML hashes changed; 69 of the 96 re-recorded tests also run `html` and pass.
- **Cross-crate causes:** (a) realize (`crates/typst-realize/src/lib.rs:1071`, in the diff) meets HTML rules and content model (`crates/typst-html/src/rules.rs:103-147`, `tag.rs:509-546`, outside); (b) flow tag placement meets PDF hoisting (`crates/typst-pdf/src/tags/context/list.rs:80-86`, outside); (c) inline + flow + distribute; (d) rules + realize filter + flow spacing; (e) realize list grouping + flow spacing; (f) realize policy + PDF/HTML semantics.
- **Spacing semantics spread over 4 places:** weakness ladder (`crates/typst-layout/src/flow/distribute.rs:185-210`); `block()` spacing (head `flow/collect.rs:351-358`); attach filter (`crates/typst-realize/src/lib.rs:774-784`); deleted attach `v` (base `crates/typst-layout/src/rules.rs:131-138`).
- **Pagination unexercised:** default test page height auto (`tests/src/world.rs:191`); paginating tests with an inline block (`issue-grid-skip-list`, `issue-grid-double-skip`, `math-pagebreaking-single-line[-numbered]`) have at most 2 text lines, all before the block, so `need` equals base.
- **Defaults hide extras:** block spacing, first-line indent, `list.spacing`, hyphen repetition (pl/pt/cs/hr/sk/dsb, es), `~` or `(` before a block, closure show rules: all need non-default input.
- **Commit history:** `6e6a5211` "amend" (+35/-47) replaces `e60dfa1d`'s realize design; per-commit reading: dead code.
- **CI signal:** expected red run; red-run CI report against the PR's own refs (`--base-revision` only on green runs, `.github/workflows/ci.yml:66`; inferred, not run).
- **Scale of the visual change:** 72 `math-*` + 24 other PNG names; gaps lost per PNG: 1 gap 57 tests, 2 gaps 14, 3 gaps 14, 4+ gaps 10; largest `math-mat-delims` -61 px; 1 unchanged size (`issue-1850`, 149x49).
- **Knowledge needed:** realize grouping priorities and interrupts; spacing weakness ladder 0-5; UAX #14 line breaking (LB12, LB14, LB20); introspection tags; HTML phrasing content; Tagged PDF list structure.
