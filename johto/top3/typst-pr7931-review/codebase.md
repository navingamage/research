# Typst: codebase context for reviewing PR #7931

PR https://github.com/typst/typst/pull/7931; head https://github.com/typst/typst/tree/1f0cd84af6bbc4f612b80822f2da04fd4649a707; base https://github.com/typst/typst/tree/c98e910391a8544b28bd5c99a6f3b1ac1ada9a84; stack: Rust (edition 2024, MSRV 1.92, workspace 0.15.0); size: 379 `.rs` files, 146,324 lines under `crates/` at base; PR: 4 commits, 110 files (+591/-314), 11 Rust files (+370/-93), 99 reference files.

**Conventions in this file**
- Paths are repo-relative. "base" = `main` at `c98e9103`; "head" = `pr-7931` at `1f0cd84a`. Line numbers are base unless marked "head".
- "(not run)" = reasoned from code, nothing built or executed. "(unverified)" = could not be confirmed from the repo.
- Problems (a) to (f) come from the PR thread (ground truth). Everything in section 14 is extra: not in the thread, not confirmed by maintainers, not run.

## 1. What Typst is

- **Product:** markup-based typesetting system. Source `.typ` files mix markup and a scripting language. Output is built from elements (text, `par`, `block`, `list`, `math.equation`, ...), styled by set rules and show rules.
- **Compiler:** one Rust workspace; phases syntax, eval, realize, layout, export.
- **Targets and outputs:**
  - Paged: PNG (`typst-render`), PDF (`typst-pdf`, tagged by default), SVG (`typst-svg`).
  - HTML (`typst-html`): experimental. Error "html export is only available when `--features html` is passed" without the feature; warning "html export is under active development and incomplete" with it (`crates/typst/src/lib.rs:105-109`, 246-266; issue #5512).
  - Bundle (`typst-bundle`): feature-gated (`Feature::Bundle`).
- **Paragraph model, as documented** (`crates/typst-library/src/model/par.rs`, unchanged by the PR):
  - inline-level elements: text, `h`, `box`, inline equations (17-19)
  - paragraphs are interrupted by any block-level element, e.g. `block`, `place` (21-23)
  - an explicit `#par[..]` body may not contain block-level content (25-28)
  - `box` puts block content into a paragraph; `block` keeps inline content out of one (30-39)
  - what becomes a paragraph: all root text; container text only if the container holds block-level content (41-55)
  - why it matters: `first-line-indent` and `par` show rules apply only to paragraphs; assistive technology; PDF `P` tag only for paragraphs (67); HTML `<p>` only for paragraphs (69)
  - non-tight lists force item paragraphs (74-77)
  - first-line indent: not for the first paragraph and not for "paragraphs immediately following other block-level elements" (379-381)
  - recommended indent style: `set block(spacing: 1.2em)` with `set par(spacing: 0.65em)` (388-401)

## 2. Crates and the compile pipeline

### Workspace
- `Cargo.toml`: `members = ["crates/*", "docs", "tests", "tests/fuzz", "tests/wrapper"]` (2); `default-members = ["crates/typst-cli"]` (3); version 0.15.0 (7); rust-version 1.92 (8); edition 2024.
- Crates (`.rs` counts from `find <dir> -name '*.rs' | wc -l`; lines counted at base for the crates this PR touches or affects):

| Crate | .rs | Lines | Role |
|---|---|---|---|
| `typst` | 1 | | facade; `compile`, routines wiring |
| `typst-syntax` | 14 | | parser, syntax tree |
| `typst-eval` | 13 | | interpreter |
| `typst-library` | 176 | 69,185 | stdlib: elements, styles, introspection, routines interface |
| `typst-realize` | 2 | 1,656 | realization (show rules, grouping) |
| `typst-layout` | 49 | 21,969 | paged layout + paged show rules |
| `typst-html` | 18 | 7,513 | HTML exporter + HTML show rules |
| `typst-pdf` | 28 | 7,997 (tags 4,902) | PDF exporter incl. tagging |
| `typst-svg` | 7 | | SVG exporter |
| `typst-render` | 5 | | PNG exporter |
| `typst-bundle` | 4 | | bundle exporter |
| `typst-ide` | 10 | | IDE features |
| `typst-kit` | 10 | | tooling utilities |
| `typst-macros` | 8 | | proc-macros (`#[elem]` etc.) |
| `typst-utils` | 14 | | utilities |
| `typst-timing` | 1 | | timing |
| `typst-cli` | 19 | | CLI; binary `typst` (`crates/typst-cli/Cargo.toml:15-17`) |
| `docs` | 7 | | typst-docs |
| `tests` | 19 (14 in `tests/src`) | | typst-tests (`[[test]] name="tests" path="src/tests.rs" harness=false`) |
| `tests/fuzz` | 4 | | fuzz targets parse, paged, html |
| `tests/wrapper` | 1 | | test-wrapper (`cargo testit`) |

- **Dependency oddity:** typst-realize imports `typst_html::HtmlElem` (`crates/typst-realize/src/lib.rs:16`).

### Pipeline
1. `compile` (`crates/typst/src/lib.rs:74`) and `trace` (87) call `compile_impl` (99-194). Pipeline doc comment at 1-31.
2. Target checks (105-109): HTML calls `warn_or_error_for_html` (246-266); bundle calls `warn_or_error_for_bundle`.
3. `TargetElem` style set (111-113).
4. `typst_eval::eval` (123-131; memoized at `crates/typst-eval/src/lib.rs:40`) returns a module with content.
5. Convergence loop (138-185): introspector from the last history entry; `comemo::Constraint` (144); `T::create` (156); converged when `constraint.validate` passes (158-161); full history goes to `introspection::analyze` for warnings (163-182). `MAX_ITERS = 5` (`crates/typst-library/src/introspection/convergence.rs:16`); `ITER_NAMES` (17); `analyze` (25).
6. Delayed errors (187-191).

- **Routines:** `ROUTINES` (`crates/typst/src/lib.rs:311-325`): show rules from `typst_layout::register` and `typst_html::register`; `realize: typst_realize::realize`; `layout_frame: typst_layout::layout_frame`. Signatures: `crates/typst-library/src/routines.rs:81-98`.
- **Paged target:**
  1. `PagedDocument::create` calls `layout_document` (`crates/typst-layout/src/document.rs:63-79`).
  2. `layout_document` (`crates/typst-layout/src/pages/mod.rs:33`; memoized impl 51-53).
  3. `layout_document_common` (128-172) realizes `RealizationKind::Document` (160-167).
  4. `layout_pages` (175-241): page runs laid out in parallel (185-195); leftover tags go to the last page (233-238).
  5. Each page run calls `layout_flow` with `FlowMode::Root` (`crates/typst-layout/src/pages/run.rs:190-198`; `layout_page_run_impl` memoized at 79).
- **HTML target:**
  1. `html_document` (`crates/typst-html/src/document.rs:25`; memoized impl 43-45) realizes Document (163-164).
  2. `convert_to_nodes` builds the node tree.
  3. `html` encodes it (`crates/typst-html/src/encode.rs:23`).
- **Exporters:** `typst_pdf::pdf` (`crates/typst-pdf/src/lib.rs:36`); `PdfOptions` (58-89) with `tagged` (82-86), default `tagged: true` (107), so PDFs carry a tag tree even without PDF/UA; `typst_svg::svg` (`crates/typst-svg/src/lib.rs:32`); `typst_render::render` (`crates/typst-render/src/lib.rs:21`).
- **Show-rule dispatch:** `NativeRuleMap` keyed by exact `(Element, Target)`, no fallback across targets (`crates/typst-library/src/foundations/styles.rs:985-1070`; `get` 1067-1069; duplicate registration panics 1041-1049). `Target` (`crates/typst-library/src/foundations/target.rs:67`); `Output` trait (13).
- **comemo memoization points:**
  - eval: `crates/typst-eval/src/lib.rs:40` (eval), 103 (eval_string); `call.rs:600` (closures)
  - typst-layout: `layout_fragment_impl` (`flow/mod.rs:108-110`); `layout_single_impl` (`flow/collect.rs:413-415`), `layout_multi_impl` (513-515); `layout_par_impl` (`inline/mod.rs:70-72`); `create_shape_plan` (`inline/shaping.rs:1220`); `layout_page_run_impl` (`pages/run.rs:79`)
  - typst-html: `html_block_fragment_impl` (`crates/typst-html/src/fragment.rs:40-42`)
  - not memoized: `layout_inline` (non-paragraph inline content), and the inline collect step inside a paragraph (runs inside `layout_par_impl`)
- **Feature gating:** `Feature` enum Html, Bundle, A11yExtras (`crates/typst-library/src/lib.rs:273-277`); CLI `--features`, env `TYPST_FEATURES` (`crates/typst-cli/src/args.rs:440-443`); the test world enables `Features::all()` (`tests/src/world.rs:174`).

## 3. Realize (grouping, paragraphs, paged vs HTML)

`crates/typst-realize/src/lib.rs` (1534 lines base, 1576 head), `spaces.rs` (122 base, 125 head).

### Entry and state
- `realize` (44-76): rule set picked by kind (57-63); `outside` set only for Document (66).
- `RealizationKind` (`crates/typst-library/src/routines.rs:153-169`): Bundle, Document{info}, Fragment{kind: &mut FragmentKind}, Par, Math. `FragmentKind` Inline | Block (171-180); `Arenas` (182-193); `Pair` (196).
- `State` (88-110): sink, groupings, rules, `may_attach`, `saw_parbreak`, `outside`, ...
- Rule sets: `MAX_GROUP_NESTING = 3` (1005); `BUNDLE_RULES = []` (1008); `FLOW_RULES = [TEXTUAL, PAR, CITES, LIST, ENUM, TERMS]` (1011, used for Document and Fragment, i.e. both paged and HTML); `PAR_RULES` = same without PAR (1014); `MATH_RULES = [CITES, LIST, ENUM, TERMS]` (1017). Head: 1019-1031.

### `visit` order (244-296)
1. Tags are pushed raw to the sink (250-253).
2. Kind rules (`visit_kind_rules` 299-351): mathy element outside math gets wrapped in `EquationElem` (332-336); `SymbolElem` becomes `TextElem`.
3. Show rules (`visit_show_rules` 355-435): start tag (414-417); `outside &=` (420); `visit_styled(realized, map)` (424); end tag (430-432).
   - `verdict` (439-532): user show-set recipes go into `map`; user show rule chosen first; built-in rule picked by target (509-514).
   - `prepare` (535-591): location for Locatable, labelled or Tagged elements; built-in `ShowSet` styles added to the same `map` (562-564); synthesize; materialize; tags (582-584).
4. Sequences, then styled content (`visit_styled` 594-694): PageElem in styles gives a pagebreak plus `outside` (paged) or the warning "page set rule was ignored during HTML export" (642-647); bails inside containers (650-653); `finish_interrupted` runs before and after the styled content (682-684).
5. Grouping (`visit_grouping_rules` 698-751).
6. Filter (`visit_filter_rules` 755-787).
7. Push to the sink.

### Grouping machinery
- `GroupingRule` (113-128): `priority`, `tags`, `effect`, `interrupt` (by style element), `finish`.
- `GroupingEffect` (131-149): Trigger, Inner, Neutral, Interrupt. Neutral exists only for mixed HTML flows.
- `Grouping` (152-163): `start`, `rule`, `interrupted`, `contains_neutral`.
- `visit_grouping_rules` (698-751):
  - a matching Trigger rule with higher priority than the active one starts a nested group (712)
  - the active group accepts content when not `interrupted` and the effect is not Interrupt (717-722)
  - otherwise `finish_innermost_grouping` and loop; "maximum grouping depth exceeded" after 512 iterations (726-734)
  - new group on Trigger (738-748)
  - an interrupting element always closes the PAR group, so inline content before it becomes a `ParElem`
- `finish_interrupted` (815-833): for each style element in the local styles, finish groupings whose `interrupt(elem)` is true; in a fully inline fragment it sets `groupings[0].interrupted = true` instead.
- `finish_grouping_while` (836-852); `finish_innermost_grouping` (855-896), which segments neutral content for HTML.
- `finish_grouping` (900-983): trims trailing non-trigger content (905-909); tag boundary inclusion (917-951); PAR space extraction (924-926); tags pulled out for rules with `tags: false` (958-972); calls the finisher (975); revisits the remainder (978-980).
- `finish` (790-812): a fully inline fragment gets `FragmentKind::Inline`, the PAR group is dropped and spaces collapsed (794-800); Par and Math kinds collapse top-level spaces.
- `is_fully_inline_or_neutral` (1175-1188): Fragment kind, `!saw_parbreak`, exactly one grouping and it is PAR, sink before it holds only tags or Neutral content.
- `in_non_par_grouping` (1165-1170): last grouping not PAR, or interrupted; used in `finish_textual` (1141).

### Grouping rules (base)

| Rule | Lines | Prio | Trigger | Inner | Style interrupt | Finish |
|---|---|---|---|---|---|---|
| TEXTUAL | 1020-1043 | 3 | Text, Linebreak, SmartQuote | Space | any style | `finish_textual` 1132-1163 |
| CITES | 1076-1093 | 2 | Cite | Space | CiteGroup, ParElem, AlignElem (1089-1091) | `finish_cites` |
| LIST/ENUM/TERMS | 1096-1122 | 2 | item element | Space, Parbreak | list element, AlignElem | `finish_list_like` 1222-1244 |
| PAR | 1046-1073 | 1 | Text, HElem, Linebreak, SmartQuote, InlineElem, BoxElem (1051-1058); HtmlElem if `should_group_into_pars` (1061-1066) | Space | ParElem, AlignElem (1071) | `finish_par` 1191-1205 |

- PAR effect for everything else is Interrupt (1067-1069): BlockElem, VElem, ParbreakElem, PlaceElem, the BlockElem from list and equation rules, ... Non-phrasing HtmlElem is Neutral.
- PAR `tags: true`; list-like `tags: false`.
- PR leaves TEXTUAL, CITES and the list-like rules unchanged.

### Paragraph formation
- TEXTUAL finishes into a PAR grouping only if PAR is in the rule set and no PAR group is active (1153-1160).
- `finish_par` (1191-1205): `collapse_spaces` in place, `repack` (1515-1534), `ParElem::new(body)`, then `visit` again.
- Paged: no ParElem show rule in `crates/typst-layout/src/rules.rs` (only style reads); flow collect consumes ParElem (`flow/collect.rs:159-186`).
- HTML: `PAR_RULE` gives `<p>` (`crates/typst-html/src/rules.rs:94-95`).
- Base: block equations interrupt PAR twice:
  - ShowSet `AlignElem::alignment` CENTER (`crates/typst-library/src/math/equation.rs:190`) is a PAR style interrupt (1071)
  - paged and HTML outputs are a `BlockElem`, an element interrupt

### Spaces, parbreaks, attach
- `visit_filter_rules` (755-787):
  - Par/Math kinds: no filtering (760-762)
  - stray `SpaceElem` dropped (764-767)
  - `ParbreakElem` dropped; sets `may_attach = false`, `saw_parbreak = true` (768-773)
  - `VElem` with `attach` dropped unless `may_attach` (774-780)
  - after any kept element: `may_attach = content.is::<ParElem>()` (784)
  - so an attach `VElem` survives only directly after a ParElem
- `spaces.rs`: `SpaceState` (12-21); `collapse_spaces` (30-75); `collapse_state` (78-100): Tag Invisible; HElem fractional or weak Destructive, else Invisible; Linebreak or whitespace-collapsing (block-display) HtmlElem Destructive (87-94); Space Space; anything else Supportive (BlockElem included).

### List item grouping
- `list_like_grouping` (1105-1122): Trigger = item element; Inner = Space or Parbreak; interrupt = list element or AlignElem.
- `finish_list_like` (1222-1244): `let tight = !elems.iter().any(|(c, _)| c.is::<ParbreakElem>());` (1226). A trailing parbreak is trimmed first (905-909), so it does not make the list wide. Trunk styles (1227-1229); item-local styles (1230-1238); `T::create(children, tight)` (1242).
- `ListLike::create` builds `ListElem::new(children).with_tight(tight)` (`crates/typst-library/src/model/list.rs:240-246`).
- Item bodies are realized later, as Fragment, inside list layout (section 6).
- Head: `finish_list_like` byte-identical at 1264-1286 (tight at 1268), so the tight flag logic is unchanged.

### Paged vs HTML in realize
- Same realize code for both targets. Differences come from target-specific show rules (`NativeRuleMap`), the HtmlElem branch in PAR (Trigger or Neutral), and PageElem handling in `visit_styled`.
- HTML fragments ignore `FragmentKind` (`crates/typst-html/src/fragment.rs:150-168`); MathML bodies use Math realization, so no paragraphs (117-147).

### Head changes in realize (detail in section 12)
- PAR trigger: `|| is_inlinable_block(content)` (head 1071). PAR now spans 1060-1088; interrupt still ParElem/AlignElem (head 1086).
- `finish_interrupted(s, content, local)` returns early when `content` is an inlinable block (head 815-845; comment 820-822; `if` at 823-825).
- `finish_grouping` skips the finisher for a tentative PAR (head 985-989).
- `is_fully_inline_or_neutral` gains `&& !s.sink[grouping.start..].iter().any(|(c, _)| is_inlinable_block(c))` (head 1198).
- `is_tentative_par` (head 1206-1221): PAR group whose only trigger is one inlinable block. Doc comment: it "only becomes a paragraph once a second trigger joins it (e.g. adjacent text or another inlinable block)".
- `is_inlinable_block` (head 1223-1230): `to_packed::<BlockElem>()` and `inlinable.as_option() == Some(true)`; explicit field only, no style chain.
- `spaces.rs`: inlinable block is `Destructive` (head 89-90).
- A tentative group's block stays in the sink without a pass through `visit_filter_rules` (the grouping step pushed it, head 738-748, before any filtering; the finisher is skipped at head 985-989), so `may_attach` is not updated by it.

## 4. Flow layout

`crates/typst-layout/src/flow/`: block 419, collect 718 (head 841), compose 965, distribute 639 (head 660), mod 435 lines.

### mod.rs
- `layout_frame` (42-51); `layout_fragment` (56-77); `layout_columns` (84-105).
- `layout_fragment_impl` (108-166): realizes Fragment (145-154), then `layout_flow(kind.into())`, so FragmentKind Inline gives `FlowMode::Inline`.
- `FlowMode` Root | Block | Inline (170-178); `layout_flow` (191-235): collect (207-215), compose loop (221-232).
- `configuration` (238-286): line numbers only for Root (267). `Work` (294-311); `Config` (356-369); `Stop` (421-429).

### collect.rs (base)
- `run_block` (77-109): Tag, `VElem`, `ParElem`, `BlockElem`, `PlaceElem`, Flush; Colbreak resets `par_situation` to First (91-93); Pagebreak bails (94-98); anything else warns "... was ignored during paged export" (99-104).
- `run_inline` (112-145): FlowMode::Inline path; splits off leading and trailing tags, calls `layout_inline`, then `lines()` with `ParElem::leading`.
- `v` (148-155): `Rel(amount, weak as u8)`, so weak `v` has weakness 1 and strong 0.
- `par` (159-186): `layout_par` gives one frame per line; pushes `Rel(par.spacing, 4)`, the lines, `Rel(par.spacing, 4)`; sets `par_situation = Consecutive`.
- `lines` (189-231):
  - `prevent_orphans` / `prevent_widows` / `prevent_all` (194-199) from `TextElem::costs` (default widow and orphan cost 100%, `crates/typst-library/src/text/mod.rs:1526-1533`)
  - edge heights front_1, front_2, back_2, back_1 (203-207)
  - `Rel(leading, 5)` between lines (210-212)
  - `need` (218-226): prevent_all and i == 0, whole three lines; prevent_orphans and i == 0, first two lines; prevent_widows and second-to-last line, last two lines; else the frame height
  - `Child::Line(LineChild{frame, align, need})`
  - assumes the lines are contiguous: uses neighbour heights plus `leading`
- `block` (235-280): `alone` (238), `sticky` (239), `breakable` (240), fr height; spacing closure (246-251): `Auto` gives `Rel(par.spacing, 4)`, `Custom(Rel)` weakness 3, `Custom(Fr)` `Fr(.., 2)`; `!breakable || fr` gives `Child::Single`, else `Child::Multi` (255-276); then `par_situation = Other` (279).
- `place` (283-335).
- Child types: `Child` (348-367): Tag, Rel, Fr, Line, Single, Multi, Placed, Flush, Break; `LineChild` (371-375); `SingleChild` (379-410, expand.y only when alone); `MultiChild` (446-510); `MultiSpill` (546-619); `PlacedChild` (623-670); `CachedCell` (678-718).

### Spacing weakness (lower number wins; equal weakness takes the larger amount)

| Source | Weakness |
|---|---|
| strong `v()` | 0 (never collapses) |
| weak `v()`, incl. the list attach `VElem` | 1 |
| block custom Fr above/below | 2 (Fr) |
| block custom Rel above/below | 3 |
| block auto above/below, paragraph spacing | 4 |
| line leading | 5 |

### distribute.rs (base)
- `Item` (74-85); `migratable` (90-102).
- `flush_tags` (151-157): pending flow tags wait in `work.tags` and are flushed right before the next frame is pushed (`frame()` at 430).
- `rel` (160-168) calls `keep_weak_rel_spacing` (185-210), a backward scan: a previous weak `Abs` is replaced when `weakness <= prev && (weakness < prev || amount > prev_amount)`, and the new item is always dropped (191-199); Tag, strong Abs and Placed are skipped (201); `Fr(None)` drops it (204); a Frame or `Fr(Some)` keeps it (206); region start drops it (209).
- `trim_spacing` (243-259).
- `line` (274-296): line height doesn't fit and the region may progress, Finish; `need` doesn't fit here but fits the next region, Finish (head 285-293). `need` is read nowhere else.
- `single` (299-322); `multi` (325-355); `multi_spill` (358-378); `frame` (381-433): sticky snapshot (388-417), footnotes (420-426); `placed` (436-460); `finalize` (487-624): restores the sticky snapshot (503-505), baseline from the first frame (594-604).

### compose.rs and block.rs
- `compose` (35-54); `page` (82-106); `column` (174-225, line numbers at 212-222); `float` (266-346).
- `footnotes` (357-421): Root only; migratable only when `!breakable` (388).
- `layout_line_numbers` (767-866): `find_in_frame::<ParLineMarker>` (784), one marker per paragraph line.
- `block.rs`: `layout_single_block` (19-~104) calls a MultiLayouter body with `pod.into()` then `.into_frame()` (54-58); `layout_multi_block` (107-254).

### Block vs inline in flow (base)
- Top-level inline content never reaches flow raw; realize groups it into ParElem. Exception: a fully inline fragment uses `run_inline`.
- Only ParElem and `run_inline` produce `Child::Line`.
- BlockElem gives Single/Multi with above/below spacing children.
- Line-level logic (widow/orphan `need`, line-number markers, leading) applies only to Line children.

### Head changes in flow
- `par()` (head 160-186) calls the new `par_children()` (head 189-292); `par_situation = Consecutive` after every paragraph (head 183), including one that ends in a block.
- `par_children`: `Child::Rel(leading, 5)` before every child after the first (head 227-229); lines become `Child::Line` with a widow/orphan `need` (head 198-245); blocks become `SingleChild`/`MultiChild` with `sticky: false`, `alone: false` and (Single) `fr: None` hard-coded, plus `inline: Some((width, tags, tags_before))` (head 254-288). Only `breakable` and alignment are read from the block's styles (head 260-261).
- No above/below for inline blocks. `block()` (head 340-387) still reads them: auto `Rel(par.spacing, 4)`, custom `Rel(.., 3)`, Fr `Fr(.., 2)` (head 351-356); `par_situation = Other` (head 386).
- `SingleChild`/`MultiChild` get an `inline` field (head 494, 568); layout sets `region.size.x = line_width` (head 508-509, 616-617).
- Old `lines()` kept (head 294) for `run_inline`, so widow/orphan code now exists twice.
- distribute: `push_tags` (head 652-660) prepends the "before" tags and appends the "after" tags to the block's frame; called in `single()` (head 321-324) and `multi()` (head 349-352), i.e. on the first region only; `multi_spill` (head 368-388) pushes none.

## 5. Inline layout

`crates/typst-layout/src/inline/`: box 93, collect 338 (head 348), deco 213, finalize 35 (head 33), line 852 (head 918), linebreak 1039 (head 1040), mod 323 (head 337), prepare 175, shaping 1511 lines.

### mod.rs
- `layout_par` (44-67) calls memoized `layout_par_impl` (70-123), which realizes Par on `elem.body` (98-106).
- `layout_inline` (126-149): non-paragraph inline content, `par: None`. Callers: `run_inline` (`flow/collect.rs:121` head) and math text (`crates/typst-layout/src/math/text.rs:31-38`, a single `TextElem`).
- `layout_inline_impl` (153-178): collect, prepare, `linebreak(region.x - hanging)`, finalize.
- `configuration` (181-242):
  - linebreaks Optimized if justify, else Simple (194-196)
  - first-line indent by situation (197-215; head 211-229): First `all && !in_list`; Consecutive yes; Other `all`; None no; only when the paragraph is start-aligned (head 223)
  - hanging indent only for paragraphs (216-220); line-number marker (221-231)
- `ParSituation` First | Consecutive | Other (248-258).
- `in_list` (319-323): `ListElem::depth > 0`, `EnumElem::parents` non-empty, or `TermsElem::within`.

### collect.rs
- `OBJ_REPLACE` U+FFFC (21). `Item` (32-46): Text, Absolute(amount, weak), Fractional, Frame, Tag, Skip; `textual` (72-80).
- `collect` (125-251): first-line and hanging indent pushed first as strong `Item::Absolute` (head 141-149, rendered as `" "`); then SpaceElem, TextElem (bidi embedding, case), HElem, LinebreakElem (U+2028 if justify, else `\n`), SmartQuoteElem, InlineElem (206-222: `LTR_ISOLATE`/`POP_ISOLATE`, `InlineItem::Space` gives `Absolute`, `InlineItem::Frame` gives `Frame` after `apply_shift`), BoxElem (223-233: fr width Fractional, else `layout_box` frame), TagElem (234-235).
- Anything else: warning "{} may not occur inside of a paragraph and was ignored" (236-243; head 246-253). Tests: `par-contains-block` (`tests/suite/model/par.typ:487-493`, `#block[]` inside `#par[..]`); `par-contains-parbreak` (479-485).
- Adjacent weak `Absolute` items merged (292-307).

### prepare, shaping, linebreak
- `prepare.rs`: `Preparation::get` (head 34-37) maps a byte offset to the item covering it; `indices` covers only non-empty item ranges (103-107); `slice` (41-59) skips leading empty-range items unless the line starts at offset 0, and keeps trailing empty-range items. So a zero-width tag at a line boundary belongs to the earlier line.
- `prepare` (66-121): bidi, shaping, CJK-Latin spacing; shape plan memoized (`shaping.rs:1220`).
- `linebreak.rs`: costs (29-30); `Breakpoint` (66-74); segmenter: default LSTM one, no tailoring (38-39), `icu_segmenter` 2.2.0 (`Cargo.lock:1290-1291`); `linebreak` dispatch (152-161); `linebreak_simple` (167-208, first-fit); `linebreak_optimized` (227-241: approximate pass at 384 bounds the Knuth-Plass pass at 246); `raw_ratio` (571-621); `raw_cost` (627-682); `breakpoints` (693-782): iterates UAX #14 opportunities only, with a special case around `OBJ_REPLACE` (736-762).

### line and finalize
- `line.rs`: `Line` (31-41); `line()` (126-200; head 143-208); `apply_shift` (457-483).
- Repeated hard hyphen: after a line ending in `-`, `line()` pushes a hyphen `Item::Text` at the start of the next line (head 175-184) when `should_repeat_hyphen` says so (head 447-471): Lower Sorbian, Czech, Croatian, Polish, Portuguese, Slovak always; Spanish when the next char is not uppercase. `trailing_text` skips tags (head 803-805).
- `commit` (487-636): line height = max top + max bottom over items (558-569, 615-617), so the tallest object sets the line box; par line marker (619-621; head 685-687); logical-order sort (626); positions (629-633). `add_par_line_marker` (645-673).
- Fallback text (gives a text-less line a non-zero height) only when a text item in the line's range was trimmed to empty (head 239-244, set in `collect_range` 324-330); a line holding only indent spacing gets none, so its height is 0.
- `finalize.rs` (8-35): width = region.x, unless `!expand` and no fr items, then max line width + hanging.

### Paragraph contents and inline equations (base)
- Paragraph contents: text, spaces, `h`, linebreak, smartquote, box (incl. fr), `InlineElem` (only inline equations), tags.
- `InlineElem` (`crates/typst-library/src/layout/container.rs:159-204`): not user-constructible (170-174); `layouter` (179-190); `InlineItem` Space | Frame (207-213); only use: `InlineElem::layouter` in `crates/typst-layout/src/rules.rs:810`.
- Inline equations: `layout_equation_inline` (`crates/typst-layout/src/math/mod.rs:51-102`) returns `InlineItem`s; `into_par_items` (`math/run.rs` ~170-250) splits at Binary/Relation classes for break opportunities; slack = `ParElem::leading * 0.7` (`math/mod.rs:86`).
- Box: `layout_box` (`inline/box.rs:16`); `BoxElem` (`container.rs:30-157`).

### Head changes in inline
- `BLOCK_REPLACE = "\u{FFFC}\n"` (head `collect.rs:23`); `Item::Block(&Packed<BlockElem>, StyleChain)` (head 48-49), width 0; collected when `elem.inlinable.get(styles)` (head 240-244).
- `breakpoints`: `point == text.len() || matches!(p.get(point).1, Item::Block(_, _))` gives `Breakpoint::Mandatory` (head 733-735). Upgrades an existing UAX #14 opportunity at the block start; adds none. The break after the block comes from the `"\n"`.
- `pub enum ParChild { Frame(Frame), Block { elem, styles: Styles, width, tags, tags_before } }` (head `line.rs:17-31`).
- `commit` (head 504-) returns `ParChild`: a line holding a Block becomes `ParChild::Block` with the line's tags split into before and after; comment "There will only ever be a single Block and possibly Tag(s)" (head 514) backed by `_ => unreachable!()` (head 533); style snapshot `styles.to_map()` (head 527); `Item::Block(_, _) => unreachable!()` in the frame path (head 672).
- `finalize` returns `Vec<ParChild>`, passing `region.x` as block width (head `finalize.rs`).
- `layout_par` returns `Vec<ParChild>` (head `mod.rs:45`, 73); `layout_inline` maps `ParChild::Block` to `unreachable!()` (head 159), relying on the realize invariant.

## 6. Lists and enums

### Elements (`crates/typst-library/src/model/`)
- `ListElem` (`list.rs`, 253 lines): `#[elem(scope, title = "Bullet List", Locatable, Tagged)]` (44); `tight` default true (66-67), docs (46-65), markup tightness not overridable by a set rule (52-56); `spacing` (105-109): auto means leading if tight, else par.spacing; `children` (150-151); internal `depth` (153-157); `ListItem` Tagged (167-172); `ListLike` trait (226-232).
- `EnumElem` (`enum.rs`, 283 lines): Locatable, Tagged (70); `tight` (93-94); `numbering` (115-116); `start` (127); `full` (139-140); `reversed` (150-151); `spacing` (160-165); `number_align` (201-202); `parents` (233-236); `EnumItem` (246-254); ListLike impl (270-276).
- `TermsElem` (`terms.rs`): struct (26); `tight` (50).
- `BlockElem` (`crates/typst-library/src/layout/container.rs:250-393` base): docs (215-249); `breakable` default true (291-292); `spacing` 1.2em (341-343), doc (321-323: larger of below/above wins; block spacing beats par spacing); docs example (333-340); `above` (345-350); `below` (352-354); `sticky` (387-388); head adds `inlinable` (390-394); `body`; `packed` (398-400); `single_layouter` (406-421, sets breakable false); `multi_layouter` (424-437); `BlockBody` (442-451).
- `VElem` (`crates/typst-library/src/layout/spacing.rs:96`): `weak` (114); `attach` (116-120): "Whether the spacing collapses if not immediately preceded by a paragraph".
- `ParElem` (`model/par.rs`): `#[elem(scope, title = "Paragraph", Locatable, Tagged)]` (98); `leading` 0.65em (210-211); `spacing` 1.2em (224-225).

### Paged show rules (`crates/typst-layout/src/rules.rs`)
- Base `LIST_RULE` (124-141) and `ENUM_RULE` (143-160): `BlockElem::multi_layouter(elem, layout_list | layout_enum)`; if tight, prepend `VElem::new(spacing).with_weak(true).with_attach(true)` with spacing = `list.spacing`, else `par.leading` (131-138, 150-157).
- `TERMS_RULE` (base 162-218; head 154-): StackElem-based; tight prepends the same attach `VElem` (base 204-215; head `with_attach(true)` at 203). Unchanged by the PR.
- Base rules.rs has 23 `BlockElem::multi_layouter`/`single_layouter` uses.
- `ALIGN_RULE` (head 663-664): `elem.body.clone().aligned(..)`, i.e. styles the body with `AlignElem::alignment`.

### Base spacing around a tight list (paged)
- After a paragraph, flow children in order: `[Rel(par.spacing,4), lines, Rel(par.spacing,4)]`, then `Rel(attach, 1)`, then block above `Rel(par.spacing,4)` or custom `Rel(x,3)`, then Multi, then block below `Rel(par.spacing,4)`.
- Above gap: attach (weakness 1) beats both 4s and a custom above (3), so the gap = `list.spacing` or leading. Test `list-attached-above-spacing` (`tests/suite/model/list.typ:275-281`, `#show list: set block(above: 100pt)`) pins this.
- Below gap: block below (4) vs next paragraph's spacing (4), larger wins, so par.spacing; a custom below (3) wins.
- Not directly after a paragraph (after a parbreak, heading, block or at the start): attach `VElem` dropped in realize (`crates/typst-realize/src/lib.rs:774-780`), so block above spacing applies.
- Wide list: no attach; block spacing only.
- Item gutter (`crates/typst-layout/src/lists.rs:29-31`): `spacing`, else leading if tight, else par.spacing. Unchanged by the PR.

### List layout (`crates/typst-layout/src/lists.rs`, 645 lines, unchanged by the PR)
- `layout_list` (19-74): gutter (29-31); marker by depth; wide appends `ParbreakElem` to each item body (49-52); `ListElem::depth` style (53); bodies wrapped in `PdfMarkerTag::ListItemLabel` / `ListItemBody` (55-58).
- `layout_enum` (78-169): gutter (90-92); numbering; parbreak (140-144); PdfMarkerTag (148-151).
- `layout_items` (246-304): `layout_stack_internal`, TTB with gutter (293-303); a stack, not a grid.
- `measure_markers` (307-337); `measure_bodies` (343-370); `layout_item` (374-403).
- `ItemLayouter` (409-628): `layout_body` via `layout_fragment` (484-505), so item bodies are realized as Fragment; `baseline_align` (511-555); `finish` (584-627).
- Item body realization:
  - fully inline body: FragmentKind Inline, `run_inline`, no ParElem
  - body with a nested list (base): not fully inline, item text becomes a ParElem, nested tight list's attach `VElem` survives after it (gap = nested `spacing` or leading). Setting of `issue-6242-tight-list-attach-spacing` (`tests/suite/model/list.typ:604-610`).

### Tests pinning list behaviour
- `tests/suite/model/list.typ`: `list-indent-specifics` 31 (pdftags); `list-attached` 267-273; `list-attached-above-spacing` 275-281; `list-non-attached-followed-by-attached` 283-291; `list-tight-non-attached-tight` 293-303; `list-wide-cannot-attach` 305-312; `list-wide-really-cannot-attach` 314-318; `list-par` 440-465 (paged html; lists only alone in `#block`); `issue-1850-list-attach-spacing` 568-586; `issue-5503-list-in-align` 588-594; `issue-6242-tight-list-attach-spacing` 604-610; `issue-529` 612; `issue-1204` 618.
- All `list-attached*` tests put plain text before the list.
- `tests/suite/model/enum.typ`: `enum-number-override-nested` 6-11 (paged pdftags pdfstandard(ua-1)); `list-mix` 19-23; `enum-start` 122 (html); `enum-par` 222 (paged html).
- `tests/suite/model/par.typ`: `par-first-line-indent-all-list` 410; `-enum` 422; `-terms` 434 (pdftags).

## 7. Equations

- **Markup:** `$ x $` is a block equation when the 2nd and 2nd-to-last syntax children are spaces (`crates/typst-syntax/src/ast.rs:873-878`); eval: `EquationElem::new(body).with_block(block)` (`crates/typst-eval/src/math.rs:18`).
- **`EquationElem`** (`crates/typst-library/src/math/equation.rs`, 252 lines):
  - docs (20-46): block equations unbreakable by default (39-40)
  - `#[elem(Locatable, Tagged, Synthesize, ShowSet, Count, LocalName, Refable, Outlinable)]` (47)
  - `block` (49-51); `numbering` (53-65); `number_align` (81-82); `alt` (116); `body` (119-120)
  - `ShowSet` (186-204), block case: `AlignElem::alignment` CENTER (190); `BlockElem::breakable` false (191); `ParLine::numbering` None (192); `EquationElem::size` Display (193); inline case: size Text; both: weight 450 and "New Computer Modern Math" (197-201)
  - `Count` (206-211): steps only when block and numbered; `Outlinable` (235-252)
- **Paged `EQUATION_RULE`** (`crates/typst-layout/src/rules.rs:805-812` base; head 797-806 with `.with_inlinable(true)` at 800): block gives `BlockElem::multi_layouter(elem, layout_equation_block)`; inline gives `InlineElem::layouter(elem, layout_equation_inline)`.
  - ShowSet breakable false makes flow use `SingleChild` (`flow/collect.rs:255-266`); `layout_single_block` calls the multi layouter with `pod.into()` (`flow/block.rs:54-58`)
  - breakable only with `show math.equation: set block(breakable: true)`
- **`layout_equation_block`** (`crates/typst-layout/src/math/mod.rs:106-249`): region splitting when breakable (138-202); unnumbered uses `build_aligned` (204-210); counter (212-217); `NUMBER_GUTTER` 0.5em (219); number per region from `regions.size.x` (228-246); `add_equation_number` (251-315), width (272-276).
- **Realize (base):** a block equation always ends any open paragraph (section 3), so a lone `$ x $` gets no ParElem, `P` or `<p>`.
- **HTML rule** (`crates/typst-html/src/rules.rs:822-841`): `<math>`, `display="block"` when block; block wraps it in `BlockElem::packed(math)`; inline gives bare `<math>` (840). `math` is phrasing content (`crates/typst-html/src/tag.rs:322`).
- **PDF tags:** EquationElem gives a `Formula` group (`crates/typst-pdf/src/tags/tree/build.rs:411-414`); Formula breakable Never (`tags/groups.rs:113-185`); `Placement::Block` if block (`tags/resolve/mod.rs:303-307`); `element_kind` Formula Inline (363-408).
- **Tests:** `tests/suite/math/equation.typ` 24 tests, mostly `paged html` (`math-equation-numbering` 4; number-align variants 60-221); `tests/suite/html/mathml.typ` (`mathml-box-equation` 106-107, `mathml-block-equation` 109-110, `mathml-box-block-equation-nested` 112-113); `issue-4278-par-trim-before-equation` (`tests/suite/model/par.typ:549-551`); `math-pagebreaking-single-line` and `-numbered` (`tests/suite/math/multiline.typ:182-197`: page height 4em, `set block(breakable: true)` show-set, "Shouldn't overflow:" then `$ a + b $` with no blank line).

## 8. HTML export

- **Gating:** see section 1 (`crates/typst/src/lib.rs:246-266`).
- **Show rules** (`crates/typst-html/src/rules.rs`, `register` 39-92; FrameElem rule is a no-op on Paged, 91):
  - `PAR_RULE` (94-95): `HtmlElem::new(tag::p).with_body(Some(elem.body.clone()))`
  - `LIST_RULE` (103-120): `BlockElem::packed(ul(li...))`; wide appends `ParbreakElem` to each `li` body; no attach `VElem`
  - `ENUM_RULE` (122-147): `ol` with reversed/start, `li value`; wide appends parbreak; `BlockElem::packed` (146)
  - `TERMS_RULE` (149-173): `dl/dt/dd` in BlockElem
  - `IMAGE_RULE` (774-820): BlockElem(img)
  - `EQUATION_RULE` (822-841): see section 7
  - none sets an inlinable flag; HTML list, enum and block-equation output is a plain `BlockElem`, a PAR Interrupt. The flag is set only in the paged rules (head `crates/typst-layout/src/rules.rs:129`, 144, 800).
- **Conversion** (`crates/typst-html/src/convert.rs`, 697 lines):
  - `convert_to_nodes` (60-90); `handle` (93-163): unknown element warns "{} was ignored during HTML export" (155-160)
  - `html.frame` content is laid out with the paged target (140-154), so paged layout changes can reach HTML output inside a frame (as SVG)
  - `handle_html_elem` (166-248): default display Block calls `html_block_fragment` (197-205); MathML `html_math_fragment`; else `html_inline_fragment` (220-228); comment (206-210): inline-level content next to block-level content always gets a paragraph
  - `handle_box` (337-369); `make_inline_level` (371-391: "Paragraphs only allow 'Phrasing content'")
  - `handle_block` (394-438): layouter bodies bail with "blocks with layout routines should not occur in HTML export" plus "this is a bug" (402-409); a single child is made block-level, else wrapped in `<div>`
  - `make_block_level` (444-486)
- **Fragments:** `html_block_fragment` checks `check_html_depth` (`crates/typst-html/src/fragment.rs:66`) and realizes a Fragment with FLOW_RULES (66-69). Depth limits: `MAX_SHOW_RULE_DEPTH` 64, `MAX_HTML_DEPTH` 72 (`crates/typst-library/src/engine.rs:342`, 348; checks 354-385; route extension 294-300).
- **Display** (`crates/typst-html/src/property.rs`, `Display::default_for` 62+): `p` Block (94), `div` Block (85), `ol`/`ul` Block (121-122), `li` ListItem (123).
- **`<li>` bodies:** ListItem display goes through `html_inline_fragment`, i.e. realize Fragment; text + nested `<ul>`: the BlockElem interrupt closes PAR, giving `<p>` around the item text inside `<li>` (derived from code, `crates/typst-realize/src/lib.rs:698-751`, 790-812; not confirmed from a ref, HTML refs are hashes).
- **Phrasing content:** `is_phrasing_content` (`crates/typst-html/src/tag.rs:291-350`) includes `mathml::math` (322) and `span` (338), excludes ul, ol, div, p; `is_whitespace_collapsing` (498-502); `should_group_into_pars` doc (509-542, "phrasing content ... is the content model of `<p>`") and fn (543-546).
- **Validation:** no check of `<p>` children. "phrasing" appears only in docs, plus `crates/typst-html/src/mathml.rs:610` "TODO: enforce phrasing content".
- **Tests:** `par-semantic-html*` (`tests/suite/model/par.typ:67-267`); `par-semantic-html-blocky` (216-257): "elements with blocky show rules force neighbour inline elements into paragraphs"; `list-par`, `enum-par`, `terms-par` (paged html); `tests/suite/html/` (5 files); mathml tests (section 7).
- **Line counts (typst-html):** convert 697, document 384, fragment 168, rules 851, tag 629, mathml 1346, property 275, encode 402.
- **PR:** `git diff main pr-7931 -- crates/typst-html crates/typst-pdf` is empty.

## 9. PDF tagging

`crates/typst-pdf/src/tags/` (4,902 lines; unchanged by the PR).

### Entry
- `tags/mod.rs`: `init` (28-39: `page_ranges` + tagged bails); `handle_start` (41-47); `handle_end` (49-55); `group` (57-75).
- Tree built from introspection start/end tag order in frames, plus `Frame::set_parent` (`tags/tree/build.rs:1-13`); `build` (180-239); `visit_frame` (241-262, only tags with `flags.tagged`); `visit_group_frame` (274-293).
- Element to group (`progress_tree_start` 337-535):
  - PdfMarkerTag kinds (349-377): `ListItemLabel` (364-366); `ListItemBody` (367-369); `Label` gives Lbl (376)
  - `ListElem` `List(Circle)` (384-388); `EnumElem` `List(Decimal)` (389-393); `TermsElem` `List(None)` (394-397)
  - `EquationElem` Formula (411-414)
  - `ParElem`: `push_weak(tree, elem, GroupKind::Par(None))` (512-513)
  - `push_group` (554); `push_located` (562); `push_weak` (570)
- Markers from `PdfMarkerTag` (`crates/typst-library/src/pdf/accessibility.rs:317-325`, `#[elem(Construct, Tagged)]`; kinds 359-376), wrapped in `crates/typst-layout/src/lists.rs:55-58`.

### Ends, splitting, PDF/UA
- `progress_tree_end` (589-672): overlapping start/end tags split groups.
- PDF/UA errors (652-661): "invalid document structure, this element's PDF tag would be split up"; hints "this is probably caused by paragraph grouping", "maybe you've used a `parbreak`, `colbreak`, or `pagebreak`".
- `split_inner_groups` (691-721); `split_outer_group` (740-811).
- `tags/groups.rs`: `new_located` (73-95); `new_weak` (108-110); `breakable` (113-185): List, LI-related and Formula Never; Par `NoPdfUa(BreakPriority::Par)` (140); P NoPdfUa (153); L/LI/Lbl/LBody Never (155-158); `break_group` (188-227); `BreakPriority` (252-261); empty weak group omitted (375-377); `GroupKind` (422-459; Par docs 452-455); `is_semantic` (634-647): Par, TextAttr, Transparent, LogicalParent, LogicalChild non-semantic, so list items find their List through a Par.

### Closing and list context
- `tags/tree/mod.rs` `close_group` (391-599): direct vs semantic parent (392-394); List pushed to its direct parent (492-494); ListItemLabel `ListCtx::push_label` (495-503); ListItemBody `push_body` (504-512); Formula (567-569); Par (584-586); `move_into` (601-612); `semantic_parent` (614-624).
- `tags/context/list.rs`: `push_label` (22-31); `ensure_within_item` (33-41); `push_body` (43-87):
  - expected structure (47-79): nested `L` is a sibling of the `LI`s inside the outer `L`
  - move-out (80-86) only if the last direct node of LBody is a List group:
    ```rust
    if let &[.., TagNode::Group(id)] = groups.get(body).nodes()
        && let GroupKind::List(..) = groups.get_mut(id).kind
    ```
  - a nested List wrapped in a Par group (P) is not matched and stays inside LBody/P

### Resolve
- `tags/resolve/mod.rs`: `resolve` (56-95); `resolve_group_node` (121-200): weak empty group omitted (179-183); validation only with an accessibility validator (195-197); `build_group_tag` (278+): L (290), Lbl (291), LBody (292), Formula `Placement::Block` if block (303-307), Par gives `P` (313); `element_kind` (363-408): P and L Block, Formula Inline; `validate_children` (410-451): rules only for TOC, TOCI, L, LI, Table, THead/TBody/TFoot, TR; `_ => ()` for the rest, including P and LBody.
- `tags/resolve/accumulator.rs` (1-83): leaf content at grouping level wrapped in Span; inline tags inside grouping elements get `Placement::Block` (48-51); `element_kind` used only for placement (43-51).
- Checks inside krilla (external dependency) cannot be seen from the repo (unverified).

### Tests
- `tests/suite/pdftags/list.typ`: `list-tags-basic` 1-8; `list-tags-mixed-with-enum` 10-17; `list-tags-wide-with-nested-list` 19-32; `list-tags-complex-item-with-sub-list` 34-39; `list-tags-complex-item-with-nested-list` 41-49; terms tests 51-86 (expected PDF/UA error 73-79); `issue-7789-list-tags-breaking` 89-93 (marker with a 100em-tall box; `pdftags` only, no UA-1).
- Others: `enum-number-override-nested` (`tests/suite/model/enum.typ:6-11`); `par-basic` (`tests/suite/model/par.typ:3`); `par-first-line-indent-all-terms` (434); `list-indent-specifics` (`tests/suite/model/list.typ:31`); `lang-tags-propagation` (`tests/suite/pdftags/lang.typ:11-20`).
- No pdftags test has a display equation next to text; `figure-tags-block-equation-with-caption` (`tests/suite/pdftags/figure.typ:24`) has a lone equation in a figure.
- pdftags refs are hashes only (`tests/ref/pdftags/hashes.txt`, 105 lines). Readable YAML is produced by `tests/src/pdftags.rs` (567 lines, hayro_syntax) at test time, stored at `tests/store/by-hash/{hash}_{name}.yml` with a symlink `tests/store/pdftags/{name}.yml` (`tests/src/collect.rs:360`, 368, 342).
- Line counts: tags/context/list 102, groups 649, tags/mod 322, resolve/mod 559, resolve/accumulator 83, tree/build 811, tree/mod 624.

## 10. Tests and reference outputs

### Layout
- Suite: `tests/suite/**/*.typ`, 222 files: root (`playground.typ`) 1, foundations 23, html 5, introspection 8, layout 47, loading 8, math 26, model 18, pdf 2, pdftags 16, scripting 24, styling 6, symbols 1, syntax 10, text 14, visualize 13.
- 3677 tests at base and head (`git grep -h '^--- ' main -- tests/suite | wc -l`); `walk_files` parses every `.typ` not starting with `// SKIP` (`tests/src/collect.rs:459-474`).
- Attribute counts: paged 2109, eval 1267, html 497, pdftags 126, pdfstandard 126 (116 exactly `pdfstandard(ua-1)`), empty 78, bundle 38, pdf 15, large 2, tolerance 1.
- Refs under `tests/ref/`: render `*.png` 1915; `html/hashes.txt` 479 lines; `pdf/hashes.txt` 1922; `pdftags/hashes.txt` 105; `svg/hashes.txt` 1915; bundle `*.txt` 27.
- Store: `tests/store/` (git-ignored, `.gitignore:9`); report `tests/store/report.html`; `tests/skip.txt` empty.
- Test page: 120pt wide, 10pt margins (100pt inner width), auto height, 10pt text (`tests/src/world.rs:170-196`; helpers `test`, `test_repr`, `print`, `lines`, `bounds`). Auto height means only tests that set a page height paginate.

### Runner
- `tests/src/tests.rs` (harness=false): `SUITE_PATH` 34; `STORE_PATH` 37; `REF_PATH` 43; `SKIP_PATH` 46; `REF_LIMIT` 20 KiB (49); `main` 51; store dirs created at 71-72.
- Header syntax `--- name attr+ ---` (`tests/README.md:128-148`); attribute parser `tests/src/collect.rs:690-760`: eval, paged, pdf, pdftags, `pdfstandard(..)`, `tolerance(u8)`, html, bundle, large, empty.
- Annotations: `// Error: 2-7 msg`, `// Warning: ...`, `// Hint: ...`; ranges like `3:2-3:7` (`tests/README.md:171-187`).
- Output types (`tests/src/output.rs`):
  - `HASH_OUTPUTS = [Pdf, Pdftags, Svg, Html]` (177-178)
  - Render is a file output: `make_live` returns `render(doc, 1.0)` (192-194), `fn render(document, pixel_per_pt)` (586), so 1 px per pt; oxipng; compare with `approx_equal` (638-644), which requires equal width and height before the per-pixel tolerance
  - Pdf: always runs the default export and a PDF/UA-1 export; returns the default one unless the test sets a standard (247-257), so UA-1 errors count only for tests that ask for UA-1; test PDFs use default options, i.e. tagged (`pdf_options` 349-355)
  - Pdftags formats the PDF's tag tree (370-372)
  - hash indices: Pdf 340-341, Pdftags 391-392, Svg 429-430, Html 478-479; Bundle file output (536); bundle PNG option `pixel_per_pt: 1.0` (500)
- Args (`tests/src/args.rs`): pattern (17-19); `-p` (20-22); `--exact` (23-25); `--list`; `--update/-u` (29-31); `--stages` (32-36); `--scale` (37-41); `--base-revision` (62-66, "If none is specified, it's compared against the current working tree", conflicts with `--update`); `--use-github-annotations` / env `USE_GITHUB_ANNOTATIONS` (67-71), the only env var; subcommands from 140 (Clean, Undangle, ...).
- `--base-revision <rev>`: hash refs read from git at `<rev>` (`tests/src/tests.rs:96-110`); file refs (PNGs) too (`read_ref_data`, `tests/src/run.rs:810-815`). Run at head against `c98e9103`, every self-recorded change fails again and shows in the report (not run).

### Commands (`tests/README.md`)
- `cargo test --workspace` (16); `cargo test --workspace --test tests` (21).
- `cargo testit` alias (24) = `run --quiet --package=test-wrapper --` (`.cargo/config.toml`); the wrapper forwards to `cargo test --workspace --test=tests -- <args>` (`tests/wrapper/src/main.rs:25-31`). `docit` = `run --profile dev-fast --package=typst-docs --`.
- Filters (27-34); `-p` (36-40); `--exact` (47-50); stages (57-84): eval; paged (render, pdf, pdftags, svg); html.
- Report written only when tests fail (86-90).
- Update refs: `--update`, `--update --stages=render` (221-238).
- Hashed refs (92-126): only hashes committed; live output in `store/by-hash/{hash}_{name}.{ext}` with symlinks in `store/{format}/`; missing old live output is regenerated by checking out the revision that committed the hash.
- `cargo testit regen` (`tests/wrapper/src/main.rs`, 280 lines): needs a clean work tree (`git diff-index --quiet HEAD`, 93-98); with `--base-revision` (60) it reruns the old revision with `--stages=pdf,pdftags,svg,html` (136-141); switches the work tree with `git read-tree -um <rev>` and back to HEAD (215-221, 156); `--rerun` reruns the suite at HEAD with `--base-revision` (158-167); prompt hint "run `cargo testit regen` to generate missing old live output" (42).
- Policy: 20 KiB ref limit and the `large` attribute (202-207); prefer assertions over images (217-219).
- README out of date: says html and pdftags compare against "reference files" (209-215), but they are hashed; no mention of `tolerance` or `bundle`.

### CI (`.github/workflows/ci.yml`)
- Triggers: push, pull_request, merge_group; `RUSTFLAGS=-Dwarnings`.
- Test matrix (28-88): ubuntu + windows 64-bit, ubuntu 32-bit; fetch-depth 100; toolchain 1.95.0; `cargo test --workspace --no-run` (58), then `cargo test --workspace --no-fail-fast` (59-62).
- "Generate missing live output and regenerate the test report" (64-66): `cargo testit regen --rerun`, plus `--base-revision=<PR base sha>` only when `job.status == 'success'` on a pull_request. On a red run the regen runs without it, so the report compares against the PR's own refs (inferred from the expression, not run).
- Store uploaded on failure (68-79: by-hash, render, pdftags, html).
- Report uploaded on failure, or on PR success from ubuntu 64 (81-88).
- checks (90-103): clippy x2, fmt, doc, `git diff --exit-code`.
- docs (105-113): `cargo docit compile --format website|pdf --deny-warnings` (112-113); compiles docs only, no visual comparison of examples.
- min-version 1.92 (115-122); fuzz (124-134); miri (136-146). Other workflows: `docker-image.yml`, `release.yml`.

### Test source line counts (`tests/src`)
- args 182, collect 884, custom 95, git 107, logger 297, notes 656, output 705, pdftags 567, report/diff 361, report/html 1384, report/mod 117, report.css 687, report.js 725, run 877, tests 278, world 275.

## 11. Contribution conventions

- `CONTRIBUTING.md` (108 lines):
  - start small (13-18); discuss design first in an issue or chat (20-24); read `docs/dev/architecture.md` (27-31)
  - no AI-implemented changes (33-39); PR description in own words with technical rationale and how users will use it, ideally linking an issue; no AI-written descriptions (41-48)
  - CI must pass; first maintainer pass usually only once CI is green (50-53)
  - maintainers check code quality, bugs, and alignment with prior discussion (55-58)
  - signs of a good PR (64-78): single discussed feature or fix; as little code and interface as possible; tests with reference output for visual/HTML tests (71-72); doc comments on all new Rust types (73); brief docs for new Typst definitions (74-78)
  - review cycle (80-86)
- `.github/pull_request_template.md`: comment pointing to the guidelines and the no-AI rule.
- `docs/dev/architecture.md` (199 lines): phases (37-50); introspection loop gives up after five attempts (134-140); layout caching per element (142-145). Out of date: "HTML export does not exist yet" (155-157); "PDF output is sadly untested so far" (192-196); typst-bundle missing from the directory list (10-34).
- No dedicated layout or reference-update policy beyond CONTRIBUTING, architecture.md and `tests/README.md`.
- Toolchains: MSRV 1.92 (`Cargo.toml:8`, mirrored in ci.yml); CI toolchain 1.95.0. Warnings are errors; clippy, fmt and doc gate.

## 12. The PR

### Identity and size
- Title "Allow equation and tight lists/enums to be part of paragraphs", closes #3206; author mkorje; still open. Text of #3206 and of the PR description: not in the repo.
- 4 commits on merge-base `c98e9103` ("Bump version on main", 2026-06-15), all committed 2026-06-29 (rebased); author dates out of order:
  - `e60dfa1d` "Create inlinable blocks" (2026-01-23)
  - `6e6a5211` "amend" (2026-06-29)
  - `77c83e25` "Make tight lists and enums inlinable" (2026-02-19)
  - `1f0cd84a` "Make block equations inlinable" (2026-02-19)
- `git diff --stat main...pr-7931`: 110 files, +591/-314.
  - 11 Rust files, +370/-93: typst-layout 8 files (+313/-87), typst-realize 2 (+51/-6), typst-library 1 (+6/-0)
  - 99 reference files: 96 PNGs in `tests/ref/render/` + 3 hash files (`tests/ref/pdf/hashes.txt` 234 diff lines, `tests/ref/svg/hashes.txt` 192, `tests/ref/pdftags/hashes.txt` 16)
  - 0 files under `tests/suite/` (no new or changed tests); 0 docs files; `tests/ref/html/` untouched; no change in typst-html, typst-pdf, typst, typst-cli, `crates/typst-layout/src/lists.rs`, `crates/typst-layout/src/math/`
- Whitespace-insensitive (`git diff -w --numstat`): `inline/linebreak.rs` +2/-1 (rest is rustfmt re-indent); `rules.rs` +13/-20; realize `lib.rs` +47/-5.

### Commit by commit
- `e60dfa1d`: 10 Rust files, +369/-73, 0 refs. Infrastructure: field, realize grouping, inline and flow layout. Nothing sets the flag yet. Its realize design used a `tentative: bool` on `Grouping` (lines 164-168, 737-740, 763-769, 882, 914 of that commit's `lib.rs`).
- `6e6a5211` "amend": `crates/typst-realize/src/lib.rs` only, +35/-47. Removes every `tentative` line, adds `is_tentative_par()` computed when the grouping finishes, moves the show-set skip from `visit_styled` into `finish_interrupted`. A commit-by-commit read critiques dead code; `git diff main...pr-7931` shows the final design.
- `77c83e25`: `rules.rs` +14/-22. Refs: 9 PNG, 29 PDF, 8 PDF-tag, 9 SVG hashes.
- `1f0cd84a`: `rules.rs` +1. Refs: 87 PNG, 89 PDF, 87 SVG hashes, 0 PDF-tag.
- Per-commit PDF counts add to 118, net 117: `list-baseline-text-with-math` changed in both.

### Mechanism, one line per layer
- typst-library: internal `block.inlinable` flag.
- typst-realize: inlinable block is a PAR trigger; a lone one stays a block ("tentative" paragraph); spaces next to it are destroyed; its show-set styles don't interrupt the surrounding paragraph.
- typst-layout rules: tight list, tight enum and block equation get `.with_inlinable(true)`; the tight-list attach `v` is deleted.
- typst-layout inline: block becomes `Item::Block` with text `"\u{FFFC}\n"`; mandatory break before it; its line is committed as `ParChild::Block`; layout deferred.
- typst-layout flow: a paragraph's children are lines plus inline blocks; blocks laid out by flow at line width with `par.leading` above and below; tags from the block's line pushed into the block's frame.

### The 11 source files (head line numbers unless marked base)
1. **`crates/typst-library/src/layout/container.rs`** +6/-0: new `BlockElem` field (head 390-394):
   ```rust
   /// Whether this block can be inline when next to other inline content.
   #[internal]
   #[default(false)]
   #[parse(None)]
   pub inlinable: bool,
   ```
   Not user-settable. The `block.spacing` docs example in the same file (333-340) is untouched.
2. **`crates/typst-realize/src/lib.rs`** +48/-6:
   - PAR trigger `|| is_inlinable_block(content)` (head 1071), effect Trigger
   - `finish_interrupted` takes `content` and returns early for an inlinable block (head 815-845). Code comment (820-822): "Show-set styles from an inlinable block's show rule are internal to the block and should not interrupt the surrounding paragraph grouping. E.g. `AlignElem` from `#show math.equation: set align(left)`." The built-in ShowSet's `AlignElem` CENTER (`crates/typst-library/src/math/equation.rs:190`) lands in the same `map` (`prepare`, lib.rs:562-564; passed to `visit_styled` at 424), so it is skipped too (code reading; the comment names only the user rule).
   - `finish_grouping`: `if !is_tentative_par(rule, &s.sink[start..]) { (rule.finish)(..) }` (head 985-989)
   - `is_tentative_par` (head 1206-1221): exactly one PAR trigger and it is an inlinable block means no `ParElem`; the block stays in the sink as block-level
   - `is_fully_inline_or_neutral` (head 1190-1204): a fragment holding an inlinable block is never `FragmentKind::Inline` (1198)
   - `is_inlinable_block` (head 1223-1230): explicit field only
   - target-agnostic: `FLOW_RULES` (head 1025) serve paged and HTML realization
3. **`crates/typst-realize/src/spaces.rs`** +3/-0: inlinable block is `SpaceState::Destructive` (head 89-90), same class as a linebreak, so spaces on both sides are dropped and text touches the block's U+FFFC.
4. **`crates/typst-layout/src/rules.rs`** +15/-22: base `LIST_RULE` (124-141; `ENUM_RULE` 143-160 same):
   ```rust
   if tight {
       let spacing = elem.spacing.get(styles)
           .unwrap_or_else(|| styles.get(ParElem::leading));
       let v = VElem::new(spacing.into()).with_weak(true).with_attach(true).pack();
       realized = v + realized;
   }
   ```
   Head `LIST_RULE` (124-137) and `ENUM_RULE` (139-152): attach `v` gone; tight lists get `.with_inlinable(true)` (129, 144). Head `EQUATION_RULE` (797-806): block equations get `.with_inlinable(true)` (800). `TERMS_RULE` untouched (attach `v` at head 203; not inlinable).
5. **`crates/typst-layout/src/inline/collect.rs`** +12/-2: `BLOCK_REPLACE` (23); `Item::Block` (48-49), width 0; collected when `elem.inlinable.get(styles)` (240-244); other blocks still warn "may not occur inside of a paragraph" (246-253); indents still pushed first (141-149).
6. **`crates/typst-layout/src/inline/linebreak.rs`** +34/-33 (+2/-1 ignoring whitespace): Mandatory break at a block start (head 733-735), only where UAX #14 already offers one.
7. **`crates/typst-layout/src/inline/line.rs`** +74/-8: `ParChild` (17-31); `commit` block path (504-560): single Block plus Tags assumed, `_ => unreachable!()` (533); `styles.to_map()` (527); `Item::Block` unreachable in the frame path (672).
8. **`crates/typst-layout/src/inline/finalize.rs`** +4/-6: returns `Vec<ParChild>`; passes `region.x` as block width.
9. **`crates/typst-layout/src/inline/mod.rs`** +20/-6: `layout_par` returns `Vec<ParChild>` (memoized `layout_par_impl`, 73); `layout_inline` maps `ParChild::Block` to `unreachable!()` (159).
10. **`crates/typst-layout/src/flow/collect.rs`** +131/-8: `par_children` (189-292), see section 4.
11. **`crates/typst-layout/src/flow/distribute.rs`** +23/-2: `push_tags` (652-660), first region only.

### Net layout effect with default settings
- Display equation or tight list next to text, no blank line: one paragraph; gap `par.leading` (0.65em) instead of 1.2em.
- Consecutive display equations or tight lists (or a mix), no blank line: one paragraph; gaps are leading.
- A lone equation or list: unchanged (still a block).
- Custom `block(above/below/spacing)` is never read for an inlined block; `list.spacing` no longer sets the gap above an attached tight list.

### Reference files changed, by kind

| Kind | File(s) | Entries changed | Of total |
|---|---|---|---|
| PNG renders | `tests/ref/render/*.png` | 96 files | 1915 |
| PDF hashes | `tests/ref/pdf/hashes.txt` | 117 | 1922 |
| SVG hashes | `tests/ref/svg/hashes.txt` | 96 (same names as the PNGs) | 1915 |
| PDF-tag hashes | `tests/ref/pdftags/hashes.txt` | 8 | 105 |
| HTML hashes | `tests/ref/html/hashes.txt` | 0 (file untouched) | 479 |

- PDF 117 = the 96 PNG names + 21 PDF-only names. None of the 8 PDF-tag names is in the PNG set.
- Only render PNGs are reference files; HTML, PDF-tag, PDF and SVG refs are hashes. Readable HTML and YAML exist only in `tests/store/` after a run.
- 69 of the 96 re-recorded tests also run `html` (all `math-`/`issue-` names); their HTML hashes are unchanged. No re-recorded list/enum test has an `html` attribute.

### Size of the visual change
- 1 px per pt (`tests/src/output.rs:193`, 586); 10pt text; one gap from 1.2em to 0.65em = 0.55em = 5.5 px.
- Widths: 85 of 96 changed PNGs are 120 px (default page); 11 use a custom page width (122-200 px); no width changed.
- Heights: 95 shorter, 1 same (`issue-1850-list-attach-spacing`, 149x49); total -1016 px; largest `math-mat-delims` -61 px (120x695 to 120x634).
- Gaps lost per test (px / 5.5, rounded): 1 gap 57 tests; 2 gaps 14; 3 gaps 14; 4 gaps 4; 5 gaps 3; 6 gaps 1; 8 gaps 1; 11 gaps 1 (`math-mat-delims`).
- By commit: all 9 PNGs from `77c83e25` shrank (text after a tight list joins the paragraph at leading); 86 of 87 from `1f0cd84a` shrank (display math in running text at leading).
- Examples: `math-mat-align` -11 px (3 equations); `list-mix` 120x64 to 120x53; `list-attached-above-spacing` 120x71 to 120x66.
- Spot checks, base vs head PNG viewed (from the notes): `math-equation-number-align-left` equations (2)-(4) and (5)-(6) closer; `list-tight-non-attached-tight` "World" pulled up under the list; `list-mix` list, enum and terms at leading; `issue-1850-list-attach-spacing` boxes 1-3 tight, 4-6 unchanged; `set-instantiation-site-markup` 4 separate lists look like one.

### Changed PNG names by suite file
- Total by file: math/mat 12, math/equation 11, math/text 10, model/list 8, math/delimited 7, math/attach 7, math/interactions 6, math/cancel 5; 3 each math/primes, math/multiline, math/call, math/accent; 2 each model/enum, math/stretch, math/root; 1 each styling/set, math/vec, math/underover, math/size, math/op, math/frac, math/class, math/cases, math/alignment, layout/line-numbers, introspection/counter, foundations/content.
- 72 names start with `math-`; 82 come from `tests/suite/math/` (adds 10 `issue-*` math tests). The thread's "72 math, 24 list, enum, equation and counter ones" is loose: the 24 non-`math-` names include the 10 math-suite `issue-*` tests plus `content-fields-complex`, `set-instantiation-site-markup`, `line-numbers-equation-number`.
- math detail: `mat` 12 (`math-mat-align`, `-align-complex`, `-align-explicit-mixed`, `-align-signed-numbers`, `-delim-class`, `-delim-set`, `-delims`, `-delims-inverted`, `-delims-pair`, `-gaps`, `-spread-2d`, `issue-852-mat-type`); `text` 10 (`math-font-fallback-class`, `math-font-features(-switch)`, `math-optical-size-*` x4, `math-text-single-grapheme-cluster`, `math-text-styled-num`, `issue-6090-math-overhang`); `attach` 7; `delimited` 7 (`math-lr-*`); `cancel` 5; `interactions` 6 (`issue-math-realize-hide/-scripting/-show`, `math-glyph-show-rule`, `math-size-math-content-1/2`); `accent` 3 (incl. `issue-7437-...`); `call` 3 (incl. `issue-3774-...`); `multiline` 3; `primes` 3; `root` 2; `stretch` 2; 1 each `alignment` (`issue-3973-math-equation-align`), `cases`, `class`, `frac`, `op`, `size` (`issue-3658-math-size`), `underover`, `vec`.
- equation (`math/equation.typ`, 11): `math-equation-align-(un)numbered`, `math-equation-number-align*` x7, `math-equation-numbering`, `issue-6170-equation-stroke`; elsewhere `line-numbers-equation-number`.
- list (9): `list-attached-above-spacing`, `list-baseline-text-with-math`, `list-big-marker-full-width`, `list-mix` (in `enum.typ`), `list-negative-indent-block`, `list-negative-indent-full-width`, `list-syntax-edge-cases`, `list-tight-non-attached-tight`, `issue-1850-list-attach-spacing`.
- enum (1) `enum-syntax-edge-cases`; counter (1) `counter-display-matching-numbering-wrong`; other (2) `content-fields-complex` (equation followed by text), `set-instantiation-site-markup` (adjacent lists).
- `77c83e25` (9 PNG): `enum-syntax-edge-cases`, `list-attached-above-spacing`, `list-big-marker-full-width`, `list-mix`, `list-negative-indent-block`, `list-negative-indent-full-width`, `list-syntax-edge-cases`, `list-tight-non-attached-tight`, `set-instantiation-site-markup`. `1f0cd84a`: the other 87.

### PDF-tag hash changes (8, all in `77c83e25`)
- `enum-number-override-nested` (`tests/suite/model/enum.typ:6`); `list-indent-specifics` (`tests/suite/model/list.typ:31`); `list-tags-basic` (1), `list-tags-mixed-with-enum` (10), `list-tags-complex-item-with-sub-list` (34), `list-tags-complex-item-with-nested-list` (41), `issue-7789-list-tags-breaking` (89) in `tests/suite/pdftags/list.typ`; `lang-tags-propagation` (`tests/suite/pdftags/lang.typ:11`).
- Each has a tight nested list after item text. Unchanged: `list-tags-wide-with-nested-list` (wide) and the terms tag tests (not inlinable).

### PDF-only hash changes (21, render and SVG unchanged)
- enum: `enum-number-align-unaffected`, `-unfolded`, `enum-number-override-nested`, `enum-numbering-closure-nested(-complex)`, `-full`, `-pattern`, `enum-syntax-number-length`
- list: `list-attached`, `list-basic`, `list-content-block`, `list-indent-specifics`, `list-marker-closure`, `list-marker-cycle`, `list-non-attached-followed-by-attached`, `list-tabs`
- grid: `issue-grid-skip-list`, `issue-grid-double-skip`
- other: `show-text-in-other-show`
- equation: `math-pagebreaking-single-line(-numbered)`
- Reading: the tag tree is embedded in the default PDF, so a structure change shows in PDF bytes with no visual change (hash only, not inspected).

### Base facts the PR changes (index)

| Topic | Base fact | Evidence |
|---|---|---|
| Block in paragraph | BlockElem = PAR Interrupt; closes the paragraph, even mid-line | `crates/typst-realize/src/lib.rs:1067-1069`, 712-734 |
| Block in `#par[..]` | warned and dropped by inline collect | `crates/typst-layout/src/inline/collect.rs:236-243`; `par-contains-block` |
| Fragment kind | item bodies with any block are not fully inline; text becomes ParElem | `crates/typst-realize/src/lib.rs:790-812`, 1175-1188 |
| Space collapse | BlockElem is Supportive | `crates/typst-realize/src/spaces.rs:78-100` |
| Tight list spacing | attach VElem (weakness 1), only after ParElem; beats block above | `crates/typst-layout/src/rules.rs:131-138`, 150-157; `crates/typst-realize/src/lib.rs:774-784`; `crates/typst-layout/src/flow/distribute.rs:185-210` |
| Terms | same attach mechanism | `crates/typst-layout/src/rules.rs:204-215` |
| Nested tight list | item text becomes ParElem, nested attach applies; issue-6242 | `crates/typst-layout/src/lists.rs:484-505`; `tests/suite/model/list.typ:604-610` |
| Widows/orphans | `need` from neighbour line heights + leading | `crates/typst-layout/src/flow/collect.rs:189-231` |
| Line box | height = max ascent + max descent of items | `crates/typst-layout/src/inline/line.rs:558-569`, 615-617 |
| Line breaking | objects are U+FFFC; special rules around them | `crates/typst-layout/src/inline/linebreak.rs:736-762` |
| Line numbers | one ParLineMarker per line; block equations switch numbering off | `crates/typst-layout/src/inline/line.rs:619-621`, 645-673; `crates/typst-layout/src/flow/compose.rs:784`; `crates/typst-library/src/math/equation.rs:192` |
| First-line indent | none after a block (`Other`); first para in a list item not indented | `crates/typst-layout/src/inline/mod.rs:197-215`, 319-323; `crates/typst-layout/src/flow/collect.rs:279` |
| Equation number | positioned with `regions.size.x` | `crates/typst-layout/src/math/mod.rs:228-246` |
| Equation block props | centred via AlignElem (a PAR style interrupt), unbreakable | `crates/typst-library/src/math/equation.rs:186-204` |
| Sticky / alone | read in flow `block()`; `alone` = single child | `crates/typst-layout/src/flow/collect.rs:238-240` |
| Footnotes | migratable only for unbreakable blocks | `crates/typst-layout/src/flow/compose.rs:388` |
| HTML | list/enum/block-eq rules emit plain `BlockElem`; `<ul>` not phrasing; no `<p>` validation | `crates/typst-html/src/rules.rs:103-147`, 822-841; `crates/typst-html/src/tag.rs:291-350`, 509-546 |
| PDF tags | P weak, non-semantic; nested list moved out only if last direct LBody child; no P child rules | `crates/typst-pdf/src/tags/tree/build.rs:512-513`; `crates/typst-pdf/src/tags/context/list.rs:80-86`; `crates/typst-pdf/src/tags/groups.rs:634-647`; `crates/typst-pdf/src/tags/resolve/mod.rs:410-451` |
| Refs | PNG refs are files; pdf/pdftags/svg/html are hashes | `tests/src/output.rs:177-178`; `tests/README.md:92-126` |

## 13. Known problems from the thread, at code level

### (a) HTML export: lists end up inside `<p>`
- **Thread:** lists end up inside `<p>`, which is not valid HTML.
- **State at head:** HTML output unchanged. HTML `LIST_RULE`, `ENUM_RULE`, `EQUATION_RULE` (`crates/typst-html/src/rules.rs:103-120`, 122-147, 822-841) build plain `BlockElem::packed(..)`; rule lookup is an exact `(Element, Target)` key with no fallback (`crates/typst-library/src/foundations/styles.rs:1067-1068`); `tests/ref/html/hashes.txt` untouched.
- **Where it comes from:** the realize change is target-agnostic, but only the paged rules set the flag, so the paragraph model now differs by target.
- **Code path, paged (head):**
  1. `LIST_RULE` returns `BlockElem::multi_layouter(..).with_inlinable(true)` for a tight list (`crates/typst-layout/src/rules.rs:124-137`).
  2. PAR treats it as a Trigger (`crates/typst-realize/src/lib.rs:1071`).
  3. `finish_par` wraps text + list in one `ParElem`.
- **Code path, HTML (head):**
  1. HTML `LIST_RULE` returns plain `BlockElem::packed(ul)`.
  2. PAR treats it as an Interrupt.
  3. Output `<p>text</p><ul>...</ul>`.
- **If HTML followed paged** (flag added to the HTML list rule; not run): `<p>` has display Block (`crates/typst-html/src/property.rs:94`), so its body is realized again through `html_block_fragment` (`crates/typst-html/src/convert.rs:197-205`; `fragment.rs:66-69`, FLOW_RULES incl. PAR); with an inlinable block in the group `is_fully_inline_or_neutral` is false (head `lib.rs:1198`), so [text, block, text] forms another ParElem and another `<p>`, level after level, until `check_show_depth` (64) or `check_html_depth` (72) fails (`crates/typst-library/src/engine.rs:342-385`). Likely a "maximum ... depth exceeded" error, not a clean `<p>` holding `<ul>`.
- **Content model:** `<p>` takes phrasing content only (`crates/typst-html/src/tag.rs:509-546` doc; `is_phrasing_content` 291); `ul`/`ol` are not phrasing, `mathml::math` is (322), so the invalid-HTML side concerns lists and enums. Browser `<p>` auto-close at `<ul>`: general HTML knowledge (unverified).
- **Visible split today:** a `show par` rule reaches lists and equations in paged output only; `query(par)` counts differ between targets (not run). `html.frame` content uses paged layout (`crates/typst-html/src/convert.rs:140-154`), so the change reaches HTML there; no test does that with text + equation or list.
- **Test that would show it:** none exists. A `paged html` test with text + tight list + text, compared base vs head (section 15).
- **Why the suite stays green:** neither HTML code nor HTML refs change; `list-par`, `enum-par`, `terms-par` only have lists alone in `#block`.

### (b) PDF tags: nested list moves under the paragraph
- **Thread:** nested list sits under the paragraph instead of being a direct child of its parent list (saecki; Tagged PDF best-practice guide §4.2.5.1, not checked here; affects `enum-number-override-nested` and `list-tags-complex*`).
- **Where it comes from:** list hoisting in typst-pdf checks only the last direct node of `LBody`; the PR wraps item text and nested list in one paragraph.
- **Code path (head):**
  1. Item body realized as Fragment by list layout (`crates/typst-layout/src/lists.rs:484-505`): [text, nested tight list] gives one `ParElem` (head `crates/typst-realize/src/lib.rs:1071`, 1213-1221).
  2. ParElem is Locatable + Tagged (`crates/typst-library/src/model/par.rs:98`) and becomes a weak `GroupKind::Par` (`crates/typst-pdf/src/tags/tree/build.rs:512-513`).
  3. The nested list's frame is laid out between the paragraph's tags (head `crates/typst-layout/src/flow/collect.rs:160-186`, 254-288), so its List group's direct parent is the Par (`crates/typst-pdf/src/tags/tree/mod.rs:492-494`).
  4. On LBody close, `push_body` (`crates/typst-pdf/src/tags/context/list.rs:43-87`) moves a nested list out only if the last node of LBody is a List group (80-86); the last node is the Par, so nothing moves.
  5. Resolve turns Par into `P` (`crates/typst-pdf/src/tags/resolve/mod.rs:313`).
- **Result:** base `L[LI[Lbl, LBody[P[a]]], L[..], LI[..]]` (base realized `[Par(a), v(attach), Block(list)]`); head `L[LI[Lbl, LBody[P[a, L[..]]]], LI[..]]`.
- **Tests that show it:** the 8 re-recorded pdftags hashes (section 12); likely also inside the list/enum PDF-only hash changes (hash only, not inspected). `list-attached` (`tests/suite/model/list.typ:267`) is the top-level case (P holding L) with an unchanged render.
- **Why the suite stays green:** hashes re-recorded in `77c83e25`; hashes show no YAML diff; 7 of 8 tests use `pdfstandard(ua-1)` with no error annotations and `validate_children` has no rule for P or LBody (`crates/typst-pdf/src/tags/resolve/mod.rs:410-451`, run only with a validator 195-197); `issue-7789-list-tags-breaking` has no UA-1 attribute, and UA-1 errors count only for tests that ask for UA-1 (`tests/src/output.rs:247-257`). Checks inside krilla: unverified.

### (c) Widow/orphan `need` in `flow/collect.rs`
- **Thread:** `need` around line 207 is wrong; author: "very wrong currently (but does appear to be benign)".
- **Where it comes from:** `par_children` (head `crates/typst-layout/src/flow/collect.rs:189-292`) copies base `lines()` (base 189-231, still at head 294 for `run_inline`) and swaps the line index for `frame_idx`.
  - TODO (head 198-200): "Should frames only (not blocks) participate in widow/orphan prevention? ... a little weird"
  - `frames` = text lines only, blocks filtered out (head 201-207); `len`, `front_1/2`, `back_1/2` over those (head 210-223); `need` (head 237-245)
  - leading added as if paired lines were adjacent; block heights never enter
- **Wrong cases** (L = text line, B = inline block; not run):
  - `[L0, B, L1]`: L0 `need` = h0 + leading + h1, pairing L0 with L1 across B.
  - `[L0, L1, B, L2]`: len 3, `prevent_all`, so L0 needs 3 lines; base needed 2 (separate 2-line paragraph). When exactly 2 lines fit, head moves L0 and L1 to the next page; base kept them.
  - `[L0, B, L1, L2]`: L0 needs L0 + L1 + L2, ignoring B; base: L0 alone, no prevention.
  - `[L0, L1, B, L2, L3]`: correct by accident.
  - `[L0, L1, L2, B, L3]`: L2 (`frame_idx` 2 = len - 2) pairs with L3 across B.
- **Why "benign":** `need` is read only in distribute `line()` (head `crates/typst-layout/src/flow/distribute.rs:285-293`): finish the region if `need` doesn't fit here but fits the next region. A wrong `need` means a missed widow/orphan or extra white space at a page bottom, never broken output. Default widow/orphan cost 100% (`crates/typst-library/src/text/mod.rs:1526-1533`).
- **Fix direction:** run `lines()`-style prevention per maximal run of lines between blocks (equals base behaviour, each run used to be its own paragraph), or define a cross-block rule explicitly; block height is unknown at collect time.
- **Test that would show it:** a short page with 2+ text lines and a display equation, sweeping the space above (section 15).
- **Why the suite stays green:** no coverage of the wrong shapes. Some tests do combine a short page with an inline block in a paragraph: `issue-grid-skip-list` (`tests/suite/layout/grid/grid.typ:248-251`, `#set page(height: 60pt)`, `#lines(2)`, `- #lines(2)`), `issue-grid-double-skip` (253-259), `math-pagebreaking-single-line` and `-numbered` (`tests/suite/math/multiline.typ:182-197`); all four PDF hashes changed. Each has at most two text lines, all before the block ([L, L, B] or [L, B]), so head `need` equals base. `tests/suite/layout/flow/orphan.typ` and `costs-widow-orphan` (`tests/suite/layout/inline/hyphenate.typ:121-130`) contain no lists or equations. Default test page height is auto (`tests/src/world.rs:191`).

### (d) `issue-6242-tight-list-attach-spacing`: regression, ref not re-recorded
- **Thread:** the author lists it as a regression still to fix (tight-list spacing before the first item).
- **Test** (`tests/suite/model/list.typ:604-610`):
  ```typ
  // Nested tight lists should be uniformly spaced when list spacing is set.
  #set list(spacing: 1.2em)
  - A
    - B
    - C
  - C
  ```
- **Base path:**
  1. Item A's body realizes to `[Par(A), v(1.2em, weak, attach), Block(B, C)]` (`crates/typst-layout/src/rules.rs:131-138`).
  2. The attach `v` survives because it follows a ParElem (`crates/typst-realize/src/lib.rs:774-784`).
  3. Weakness 1 wins (`crates/typst-layout/src/flow/distribute.rs:191-198`), so A to B = 1.2em = `list.spacing`; spacing is uniform.
- **Head path:**
  1. The attach `v` is deleted (head `rules.rs:124-137`); the body becomes `Par(A + Block(B, C))` (head `lib.rs:1071`, 1213-1221).
  2. `par_children` inserts `Child::Rel(leading, 5)` (head `crates/typst-layout/src/flow/collect.rs:227-229`), so A to B = `par.leading` = 0.65em.
  3. B to C and C to C stay 1.2em (list gutter, `crates/typst-layout/src/lists.rs:29-31`).
- **Root cause, broader than this test:** `set list(spacing: ..)` / `set enum(spacing: ..)` no longer set the gap above the first item of an attached tight list; only item-to-item gaps still follow it. The only base code that applied it above the list was the attach `VElem` (base `rules.rs:131-138`, 150-157).
- **Ref state:** `tests/ref/render/issue-6242-tight-list-attach-spacing.png` unchanged (120x82, 410 B, last changed `f312df7b`, 2025-10-24); PDF hash (`tests/ref/pdf/hashes.txt:810`) and SVG hash (`tests/ref/svg/hashes.txt:808`) unchanged.
- **Expected at head (not run):** render about 5-6 px shorter; `approx_equal` needs equal width and height (`tests/src/output.rs:638-644`), so the render stage fails; PDF and SVG hashes mismatch. One red test, which the author already lists.
- **CI consequence (not run):** `cargo test --workspace --no-fail-fast` (`.github/workflows/ci.yml:59`) goes red; CONTRIBUTING says the first maintainer pass usually waits for green CI (`CONTRIBUTING.md:50-53`); the base-revision report is generated only on green runs (ci.yml:66).
- **Why the rest stays green:** every other affected ref was re-recorded; this one was left alone.

### (e) Tight list followed by a paragraph
- **Thread:** "a tight list followed by a paragraph turns non-tight". Which configuration the thread meant cannot be recovered from the code.
- **Tight flag unchanged:** `finish_list_like` byte-identical (base `crates/typst-realize/src/lib.rs:1222-1244`, tight at 1226; head 1264-1286, tight at 1268); trailing Parbreak and Space trimmed before finishing (head 916-920). Traced cases keep `tight: true`: list then text; list, blank line, text; list nested after item text. So the effect is spacing, not the flag.
- **Base mechanics:** attach `v` above the list only; below the list `block.below` applies (auto = `par.spacing`; a custom value wins at weakness 3).
- **Head mechanics:** list inside a paragraph with `par.leading` above and below (head `crates/typst-layout/src/flow/collect.rs:227-229`) and `par.spacing` at the paragraph end (178, 182); `block.above/below` never read for an inlined block.
- **Concrete changes (not run):**
  1. Tighter: text right after a tight list with no blank line was a new paragraph at 1.2em; now the same paragraph at 0.65em. Re-recorded: `list-attached-above-spacing`, `list-tight-non-attached-tight`, `list-syntax-edge-cases`, `enum-syntax-edge-cases`.
  2. Looser: blank line after an attached tight list with a custom `block` below/spacing on lists: base used `block.below` (e.g. `#show list: set block(below: 0.65em)` gives 0.65em), head uses `par.spacing` (1.2em). A standalone tight list (tentative, no text before it) still honours `block.below`, so the result depends on whether text precedes the list. This is the direction that matches "turns non-tight".
  3. A paragraph that starts with a list (list then text) plus first-line indent: an indent-only line plus leading above the list (extra E3).
- **Test that would show it:** cases 2 and 3 untested; case 1 refs re-recorded (section 15 has inputs).
- **Why the suite stays green:** case 1 refs re-recorded; cases 2 and 3 have no test.

### (f) Open design question: lone vs side-by-side equations as a paragraph
- **Thread:** whether a lone equation or two side-by-side equations should form a paragraph (accessibility arguments).
- **Rule in code:** `is_tentative_par` (head `crates/typst-realize/src/lib.rs:1206-1221`): a group whose only PAR trigger is one inlinable block gets no paragraph; a second trigger makes one. Its doc comment states this policy.
- **Results:**
  - `$ a $` alone: plain block
  - `$ a $ $ b $`, or two lines with no blank line: one `ParElem` holding 2 blocks
  - any two inlinable blocks with no text form a paragraph: equation + list (`issue-1850-list-attach-spacing`, `tests/suite/model/list.typ:568-586`, boxes 1-3); list + enum (`list-mix`, `tests/suite/model/enum.typ:19-23`); four lists (`set-instantiation-site-markup`, `tests/suite/styling/set.typ:8-19`)
  - `#h(1em)` or `#box` + equation: one paragraph (HElem/BoxElem are triggers)
- **Semantics and accessibility:** PDF tags `P[Formula, Formula]` for a pair, a bare `Formula` alone (Formula group `crates/typst-pdf/src/tags/tree/build.rs:411`); HTML no `<p>` in either case, so targets disagree.
- **Visual:** gap between consecutive display equations 1.2em to 0.65em (`math-mat-align` 3 equations, -11 px; `math-mat-delims` -61 px). Many math refs now encode "two equations = one paragraph" (`math-equation-number-align*`, `math-mat-*`).
- **Why the suite stays green:** a policy choice; refs re-recorded to match it.

## 14. Extra problems not raised in the thread

All: not in the thread, not confirmed by maintainers, not run.

### E1. Block spacing ignored for inlined blocks; default display-math gap shrinks
- `par_children` emits only `Child::Rel(leading, 5)` around inline blocks (head `crates/typst-layout/src/flow/collect.rs:227-229`); `block()` (head 340-387, spacing 351-358), which reads `above`/`below` at weakness 3 or 4, is bypassed. Equation show-set styles don't end the paragraph (head `crates/typst-realize/src/lib.rs:823-825`).
- So `#show math.equation: set block(above: .., below: ..)`, `#set block(spacing: ..)`, `#show list: set block(..)` do nothing when the block sits in a paragraph.
- The `block.spacing` docs example (`crates/typst-library/src/layout/container.rs:333-340`, same at base and head) breaks; at head both gaps become leading and "A second paragraph." is no longer a paragraph:
  ```typ
  #set align(center)
  #show math.equation: set block(above: 8pt, below: 16pt)

  This sum of $x$ and $y$:
  $ x + y = z $
  A second paragraph.
  ```
- `par.first-line-indent` docs (`crates/typst-library/src/model/par.rs:388-401`) recommend `set block(spacing: 1.2em)` with `set par(spacing: 0.65em)`; under that setup equations inside paragraphs lose the 1.2em entirely.
- Default gap around display math in running text: 1.2em to 0.65em (86 of the 87 PNGs from `1f0cd84a`).

### E2. Panic paths through `unreachable!()` in `commit`
- `commit` assumes a block's line holds only the block plus tags (head `crates/typst-layout/src/inline/line.rs:514`), else `_ => unreachable!()` (533). Base: these inputs compile (the block was its own flow child).
- **No UAX #14 break before the block's U+FFFC:** the forced break only upgrades opportunities the segmenter yields (head `crates/typst-layout/src/inline/linebreak.rs:723-735`); segmenter default, untailored (38-39), `icu_segmenter` 2.2.0 (`Cargo.lock:1290-1291`); spaces next to the block are removed (head `crates/typst-realize/src/spaces.rs:89-90`), so text touches U+FFFC. No break after NBSP `~` (LB12), WJ (LB11), ZWJ (LB8a), opening `(`/`[` (LB14), or a non-closing quote (LB19); these rank above LB20 (CB) in the spec (ICU rule order unverified). Inputs: `(` then `$ x $` then `)` on separate lines; `see~` then `$ x $`; `(` then `- a`.
- **Repeated hard hyphen:** after a line ending in `-`, `line()` puts a hyphen `Item::Text` at the start of the block's line (head `line.rs:175-184`) for cs, hr, pl, pt, sk, dsb always and es when the next char isn't uppercase (U+FFFC isn't) (`should_repeat_hyphen`, head 447-471); `trailing_text` skips tags (head 803-805). Input: `#set text(lang: "pl")`, `Wzór -`, `$ x $` on separate lines gives a block line `[Text(hyphen), Block]`.

### E3. Indent-only empty line before a paragraph-initial block
- First-line and hanging indent are pushed as strong `Item::Absolute` before any child (head `crates/typst-layout/src/inline/collect.rs:141-149`), rendered as `" "` (head collect.rs:79).
- The break after that space is upgraded to Mandatory because the next item is the block (head `linebreak.rs:733-735`); the line has no Text, so no fallback and zero height (head `line.rs:239-244`, 623-683); then `par_children` adds `Rel(leading, 5)` (head `flow/collect.rs:227-229`). Extra space above the equation or list.
- Triggers: a paragraph that starts with a block and is `ParSituation::Consecutive` (e.g. paragraph, blank line, `$ x $`, then "where ..."), or `first-line-indent: (all: true)` (common in CJK templates); indent applies only for start alignment (head `inline/mod.rs:223`).
- With `par.line(numbering)`, the empty line also gets a line-number marker (head `line.rs:685-687`); visible effect unverified.
- Not covered: `par-first-line-indent*` tests (`tests/suite/model/par.typ:323-445`) have no paragraph-initial block.

### E4. Introspection, locations and tags
- The block's start tag lands on the preceding text line: tags are pushed raw into the paragraph (`crates/typst-realize/src/lib.rs:249-253`; PAR `tags: true`, head 1062); `indices` covers only non-empty items (`crates/typst-layout/src/inline/prepare.rs:103-107`); `slice` skips leading empty items unless the line starts at offset 0 and keeps trailing ones (45-58). So the start tag goes into the previous line's frame (head `line.rs:666-669`) with `tags_before = 0`.
- Base: a flow `Child::Tag` waits in `work.tags` and is flushed right before the next frame (`crates/typst-layout/src/flow/distribute.rs:151-157`, 430), so it travelled with the block.
- So `locate(<eq>)` position and page = end of the preceding line, possibly the previous page when the equation moves; affects references and links to equations, `outline(target: math.equation)` page numbers, `counter(page).at(<eq>)` (not run).
- Breakable inline list across regions: "after" tags (the list's end tag) go onto the first region's frame only (head `distribute.rs:349-352`; `multi_spill` 368-388 adds none); later-page items then sit after the `L` end tag. Possible tag-tree damage or PDF/UA nesting error; fallbacks in `crates/typst-pdf/src/tags/tree/mod.rs:492-510` (unverified).

### E5. Re-recorded ref contradicts its own test
- `issue-1850-list-attach-spacing` (`tests/suite/model/list.typ:568-586`) is commented "List attachment should only work with paragraphs, not other blocks" (569).
- Boxes 1-3 (equation + list, no parbreak) now form one ParElem (head `crates/typst-realize/src/lib.rs:1213-1221`); the new ref shows the list pulled up to the equation in boxes 1-3; boxes 4-6 unchanged; size 149x49 unchanged.

### E6. Term lists left out
- `TERMS_RULE` keeps the attach `v` (head `crates/typst-layout/src/rules.rs:196-205`) and is not inlinable.
- `list-mix` (`tests/suite/model/enum.typ:19-23`): list + enum are two inlinable triggers, so one ParElem forms; the terms attach `v` then survives because it follows a ParElem (`crates/typst-realize/src/lib.rs:774-784`). PNG 120x64 to 120x53 (-11 px).

### E7. First-line indent now applied after equations and lists
- A paragraph after `Par(text + $eq$)` and a blank line is `Consecutive` (head `crates/typst-layout/src/flow/collect.rs:183`), so it is indented; base: it followed a block, `Other` (base 279; head 386), no indent unless `all` (head `crates/typst-layout/src/inline/mod.rs:218-220`).
- Contradicts `crates/typst-library/src/model/par.rs:379-381` ("not paragraphs immediately following other block-level elements") for paged output.

### E8. `show par` rules reach lists and equations (paged only)
- `#show par: set text(red)` now colours a list or equation that shares a paragraph with text; HTML unchanged. Another paged/HTML split.

### E9. Explicit `#par[..]` accepts inlinable blocks
- Base: a block equation or tight list in the body was dropped with "may not occur inside of a paragraph" (base `crates/typst-layout/src/inline/collect.rs:236-243`); head: laid out (head 240-244). `par-contains-block` (`tests/suite/model/par.typ:487-493`) uses a plain `#block[]`, which still warns at head, so it never covered inlinable blocks. No test.

### E10. Tight list loses attachment when the paragraph before it is closed first
- Cases: explicit `#par[..]`; `#align(..)[..]` (head `crates/typst-layout/src/rules.rs:663-664` styles the body with `AlignElem`, a PAR interrupt); a `#set par(..)` or `#set align(..)` between text and list.
- Base: attach `v` kept after any ParElem (`crates/typst-layout/src/rules.rs:131-138`; `crates/typst-realize/src/lib.rs:784`), gap = leading or `list.spacing`.
- Head: `v` deleted (head `rules.rs:124-152`); the lone list in a fresh PAR group stays a block (head `lib.rs:1213-1221`) with auto above = `par.spacing` (head `flow/collect.rs:351-358`). Gap 0.65em to 1.2em.
- `list-attached*` tests (`tests/suite/model/list.typ:267-303`) only put plain text before the list.

### E11. Paragraph membership depends on how a show rule is written
- `is_inlinable_block` matches only the built-in BlockElem (head `crates/typst-realize/src/lib.rs:1223-1230`), and the interrupt skip is keyed on that content (820-825).
- `#show math.equation: set align(left)`: show-set, block still inline. `#show math.equation: it => { set align(left); it }`, `it => block(..)`, even `it => it`: the outer `visit_styled` gets the closure's output with the built-in ShowSet map (CENTER), so `finish_interrupted` closes the paragraph before and after (lib.rs:424, 682-684; `prepare` 562-564). Equation stays a separate block, base behaviour (code reading).
- No user switch: the field is `#[internal]` and `#[parse(None)]` (head `crates/typst-library/src/layout/container.rs:390-394`); the ways out are a blank line or a non-set show rule.

### E12. Other block properties dropped inside a paragraph
- `par_children` hard-codes `sticky: false`, `alone: false`, `fr: None` (head `crates/typst-layout/src/flow/collect.rs:254-288`), while `block()` reads them (head 340-386). So `block(sticky: ..)` or `block(height: 1fr)` show-set rules on lists and equations are ignored inside a paragraph.
- The inline block gets `region.x` as width (head `finalize.rs`, `line.rs` block path), so hanging indent is ignored for it.

### E13. Docs contract changes with 0 docs files
- Paged output no longer matches `crates/typst-library/src/model/par.rs:17-28` (inline-level list; interruption by block-level elements; `#par` body without block-level content), `par.rs:379-381` (no indent after block-level elements) and the `crates/typst-library/src/layout/container.rs:333-340` example.
- `CONTRIBUTING.md:64-78`: tests with reference output and documentation are signs of a good PR. The docs CI job only compiles (`.github/workflows/ci.yml:112-113`), so a changed example renders without failing.

### E14. Tag-tree coverage gap for equations
- `1f0cd84a` changed 0 pdftags hashes; no `tests/suite/pdftags` test has a display equation next to text (`figure-tags-block-equation-with-caption`, `tests/suite/pdftags/figure.typ:24`, is a lone equation).
- Only signs of `P` holding a `Formula`: the PDF-only hash changes of `math-pagebreaking-single-line` and `-numbered` (`tests/suite/math/multiline.typ:182-197`), renders identical. For lists, `list-attached` (`tests/suite/model/list.typ:267`) is the matching case.

### E15. Smaller items
- No new tests (0 suite files); `par_children` duplicates `lines()`.
- `styles.to_map()` clones the style chain per inline block inside memoized `layout_par` (head `line.rs:527`); perf impact unverified.
- Adjacent separately-styled lists merge visually into one (`list-big-marker-full-width`, `set-instantiation-site-markup`).
- Commit named "amend" left in history; `ParChild` variants have no doc comments (CONTRIBUTING asks for doc comments on new Rust types, 73).

### Checked, no problem found
- Equation numbering: `Count` impl (`crates/typst-library/src/math/equation.rs:206-211`) is order-based.
- Justification and runt cost: a line before a block ends in a Mandatory break, same as a paragraph's last line at base.
- `layout_inline`'s `unreachable!()` (head `crates/typst-layout/src/inline/mod.rs:159`): `FlowMode::Inline` needs `FragmentKind::Inline` (`crates/typst-realize/src/lib.rs:790-800`), which is false whenever an inlinable block is present (head 1198); a lone inlinable block keeps `FragmentKind::Block`; the math caller passes a single `TextElem` (`crates/typst-layout/src/math/text.rs:31-38`).
- `v(weak)` docs example (`crates/typst-library/src/layout/spacing.rs:105-113`): `v` still interrupts the paragraph.
- List tightness: realize logic identical, so a "tight flag flips" claim is wrong.

## 15. Minimal inputs that show each problem without the re-recorded refs (inputs only, not run)

**Method**
- Same input on base and head; compare live output in `tests/store/{render,html,pdftags}/<name>.*` between the two, not against refs. Commands: `cargo testit --exact <name>` with `--stages` render, html, pdftags.
- Or at head: `cargo testit --base-revision c98e9103` (reads refs from base via git; `tests/src/tests.rs:96-110`, `tests/src/run.rs:810-815`); hashed outputs need old live output, regenerated by `cargo testit regen --base-revision c98e9103`, which rewrites the work tree (`git read-tree -um`) and refuses a dirty tree (`tests/wrapper/src/main.rs:93-98`, 215-221).
- Without running: `git show main:tests/ref/render/<name>.png` vs `git show pr-7931:...` for re-recorded PNGs.

**(a) HTML vs paged split**
```typ
--- probe-a-html-split paged html ---
#show par: it => if target() == "html" { it } else { highlight(it) }
Intro text
- one
- two
Outro text
```
- HTML base vs head: identical, `<p>Intro text</p><ul>...</ul><p>Outro text</p>`. PNG: head highlights text, list and "Outro text" as one paragraph.

**(b) PDF tags**
```typ
--- probe-b-nested-list pdftags pdfstandard(ua-1) ---
- a
  - b
- c
```
- YAML (`tests/store/pdftags/<name>.yml`): base nested `L` beside the `LI`s of the outer `L`; head `LI[Lbl, LBody[P[a, L]]]`. Existing tests: `enum-number-override-nested`, `list-tags-complex-item-with-nested-list`.

**(c) Widow/orphan**
```typ
--- probe-c-need paged ---
#set page(height: 80pt)
#v(40pt) // sweep 30–50pt so exactly two text lines fit at the bottom
#lorem(10) // adjust so it wraps to exactly 2 lines
$ x = 1 $
End.
```
- Expected head: the 2 lines jump to the next page while base keeps them. Mirror case `[L0, B, L1, L2]`: one short line, `$ x $`, then 2 lines.

**(d) issue-6242**
- `cargo testit --exact issue-6242-tight-list-attach-spacing` at head; expected: render fails, report shows A to B smaller than B to C. Minimal form: the test text itself.

**(e) Tight list then paragraph (two directions)**
```typ
--- probe-e-tighter paged ---
Hello
- A
- B
World
```
- B to World: base 1.2em, head 0.65em.
```typ
--- probe-e-looser paged ---
#show list: set block(below: 0.65em)
Hello
- A
- B

World
```
- B to World: base 0.65em, head 1.2em. Without "Hello", head keeps 0.65em (lone list honours `block.below`).
- Indent case: `#set par(first-line-indent: (amount: 1em, all: true))`, then `- A`, `- B`, `World`.

**(f) Lone vs paired equations**
```typ
--- probe-f-pairs paged pdftags pdfstandard(ua-1) ---
$ a = b $

$ c = d $
$ e = f $
```
- PNG: head c to e gap smaller than a to c; base equal. Tags: head wraps the 2nd and 3rd equations in `P`.

**Extras**
- E1: the `block.spacing` docs example (section 14) as a `paged` test.
- E2 (expect a panic at head, normal output at base):
  ```typ
  (
  $ x $
  )
  ```
  ```typ
  see~
  $ x $
  ```
  ```typ
  #set text(lang: "pl")
  Wzór -
  $ x $
  ```
- E3: `#set par(first-line-indent: 1em)`, "First.", blank line, `$ x $`, "where x" on the next line; extra space above the equation.
- E4: `#set page(height: 5em)`, text filling page 1, then `$ x $ <eq>`, then `#context locate(<eq>).page()`.
- E5: `issue-1850-list-attach-spacing` PNGs base vs head.
- E6: `Text` then `/ T: d` vs `Text` then `- d`; PNG and tags.
- E7: `#set par(first-line-indent: 1em)`, "Intro", `$ x $`, blank line, "Next."; indent of "Next.".
- E10: `#par[Hello]` then `- A`; `#align(center)[Hello]` then `- A`; `Hello`, `#set par(justify: true)`, `- A`. Gap Hello to A: base 0.65em, head 1.2em.
- E11: "Text", `$ x $`, "more" under `#show math.equation: set align(left)` vs `#show math.equation: it => { set align(left); it }`.

## 16. Unverified items

- Text of issue #3206, the PR description, and the exact thread wording behind (e); which thread message discloses (d) (description or comment). Not in the repo.
- Tagged PDF best-practice guide §4.2.5.1 content (outside the repo).
- Browser `<p>` auto-closing at `<ul>` (HTML spec, outside the repo).
- Checks inside krilla for `L` inside `P` under PDF/UA-1.
- ICU4X rule order for the no-break cases in E2 (LB8a, LB11, LB12, LB14, LB19 vs LB20).
- HTML `<li>` text next to a nested list getting `<p>` (derived from code; HTML refs are hashes).
- Visible effect of the line-number marker on an indent-only line (E3).
- Tag-tree damage or PDF/UA error for breakable inline lists across pages (E4).
- Perf impact of `styles.to_map()` per inline block (E15).
- Every "(not run)" consequence: render sizes, failing stages, panics, page moves.
