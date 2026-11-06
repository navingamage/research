# Repo vetting log

Checked 2026-10-31 to 2026-11-01. Method:
- **Blobless clone** (`git clone --filter=blob:none --no-checkout`), then `git ls-tree` at HEAD and at the candidate pin.
- **AI files:** `CLAUDE.md`, `AGENTS.md`, `.claude/`, `.cursor`, `copilot-instructions`, `GEMINI.md`.
- **AI commits:** `Co-authored-by:` lines naming Claude, Copilot, Cursor, Codex, Devin or GPT. I recorded the count and the date of the first one.
- **Build sources:** every `<repository>`/`maven { url }` in the build files, with parent and dependency versions checked on repo1.maven.org.
- **Gap:** grep of main code, plus `git log --grep` across all history, at the pin and at upstream HEAD.

## Kept

| Repo | Pin | Language / build | Size | AI at pin | Build sources | Notes |
|---|---|---|---|---|---|---|
| spiral-project/ihatemoney | `e66a7672e8e5c41549bf53c4a824c72c43ab9079` (2026-08-07) | Python, Flask, SQLAlchemy, Alembic | 235 files, 46 .py, ~3k lines of app code, ~130 pytest tests | none ever | PyPI | Free hosted version, no pricing page. Personal-scale. |
| ananthakumaran/paisa | `18ecd7d502c91cd5695d18ee42ac51d35919a6dd` (2026-10-21) | Go 1.24 + Svelte | 93 Go + 159 Svelte/TS files, 6 Go test files + JSON regression suite across ledger/hledger/beancount | none ever | Go modules, npm (+ ledger/hledger/beancount binaries) | No pricing. Personal-scale. |
| apache/fineract | `8a61e30a4521c77a59bc55f598e14c128ffd183a` (2026-02-14) | Java 21, Gradle 8.14 | 4,736 main / 930 test Java files, 266 integration tests | 0 | Maven Central + Gradle plugin portal | AI commits start 2026-02-17, and `AGENTS.md`/`.claude/` appear 2026-07-05, so pin before them. |
| apache/ofbiz-framework | `f05a0c2268f163573a0e0895ec381593de1283db` (2026-05-10) | Java 17 + Groovy, Gradle 8.14 | 1,147 Java + 484 Groovy, 35 test definitions (embedded Derby) | 0 | Central, maven.restlet.talend.com, packages.atlassian.com, clojars, alfresco public | AI commits start 2026-05-16 (21 since). The Gradle wrapper jar isn't committed, so run `gradle/init-gradle-wrapper.sh` first. Only 2 direct dependencies need a non-Central repo, one each from atlassian and alfresco (checked 2026-11-04). |
| traggo/server | `8ca9bb6aa414ae621488a5a158a6235e89cfa57b` (2026-09-27) | Go 1.25 + React 16 / TS 3.7 (react-scripts 3) | 272 files, ~4.5k Go + ~6.8k UI TS lines, 58 Go test files, 1 UI test file | none ever | Go modules, public npm (yarn v1) | Self-hosted time tracker, no pricing. Personal-scale. UI build needs `NODE_OPTIONS=--openssl-legacy-provider` on Node 17+. Frontend candidate, see [ideas/traggo-daily-totals.md](ideas/traggo-daily-totals.md). |
| ggozad/oterm | `de8604c6425f5f4777319d7ac228b6d97e47e8a8` (0.14.7, 2025-12-19) | Python 3.10+, Textual, ollama-python, fastmcp, aiosqlite | 56 source files, 18 test files (most need a live Ollama) | none before 2026-02-25 (the next commit) | PyPI (uv) | Terminal client for Ollama with tools and MCP. No pricing. Personal-scale. See [ideas/oterm-tool-approval.md](ideas/oterm-tool-approval.md). |
| TheR1D/shell_gpt | `4ea2f834cff5dc020dabb25bb02030c630cd2062` (1.5.0, 2026-01-28) | Python CLI, OpenAI or LiteLLM | 42 files, 8 test files | none before 2026-05-06 | PyPI | Small. Backup for the oterm idea. |
| pretix/pretix | `e6f8f178aff537bc32dc7359a5f561b8074a269b` (2026-10-29) | Python, Django 5.2 | 2,747 files, 995 .py source, 182 test modules | none ever | PyPI | It has a hosted product with a pricing page, so it's Enterprise. |
| killbill/killbill | `cb60779c171391be558cd7aebb1eafea60ad2b82` (2026-08-20) | Java 21, Maven | 1,698 Java / 616 test | 0 (2 Copilot commits come later, 2026-10-19) | Central (parent `killbill-oss-parent 0.147.20`) | The last public-build pin. Later commits need parent 0.147.36, which is only on a login-protected Cloudsmith repo. |

## Rejected

| Repo | Reason |
|---|---|
| go-gitea/gitea, go-vikunja/vikunja, netbox-community/netbox, keycloak/keycloak, saleor/saleor, mealie-recipes/mealie, photoprism/photoprism | `AGENTS.md` / `CLAUDE.md` / `.claude` at HEAD |
| getlago/lago-api | `AGENTS.md` + `CLAUDE.md`, 235 of the last 400 commits AI co-authored |
| idempiere/idempiere | `AGENTS.md`, `CLAUDE.md`, `.claude/skills` |
| rundeck/rundeck | `.claude/` docs, and 420 AI co-author trailers across the last 400 commits |
| inventree/InvenTree | `AGENTS.md` + `CLAUDE.md`, 95 AI commits since 2025-07 |
| openmrs/openmrs-core | 148 of the last 400 commits AI co-authored |
| woodpecker-ci/woodpecker | 21 AI co-authored commits |
| listmonk, gotify | Too small or too few tests for Enterprise |
| thingsboard | Clean, but 9k files and slow to work with |
| axelor/axelor-open-suite | Clean, but no root build. It only runs inside the separate axelor-open-webapp repo. |
| adempiere/adempiere | Clean, but mixed ant/gradle/sbt builds, 243 tests across 15k files, and it needs a seeded DB |
| metasfresh/metasfresh | Clean, but 46k files with a docker-based build. Too heavy. |
| BroadleafCommerce | Clean, but it's a framework, not an application |
| frappe/erpnext | Clean. The retention-billing gap exists, but payment terms can pass for a workaround, and setting up the Frappe bench is heavy. Kept as a backup. |
| mayswind/ezbookkeeping | 5 AI commits since 2026-03 (pinnable before that). Not explored further. |
| beancount/fava | Clean, 0 AI commits at `12d6fea6` (2026-11-03). Svelte 5, strict TS, 23 frontend test files. Backup for the frontend slot behind traggo: the maintainer closes gaps within weeks and the typed test suite catches most mistakes. |

## Ideas dropped during gap checks
- **Fineract balloon loans:** partly supported already through fixed-principal-percentage instalments (FINERACT-1348, FINERACT-2789).
- **Kill Bill coupons and discounts:** the catalog already has `DISCOUNT` phases.
- **Kill Bill expiring credits:** a real gap (`CBADao` has no per-credit tracking), but it's the same repo and the same kind of problem as the minimum-invoice idea.
- **Healthchecks maintenance windows, Miniflux WebSub:** too easy. See [past-attempts.md](past-attempts.md).

## Vetted 2026-11-04
Same method (blobless clone since 2025-01-01; AI files at HEAD and AI co-author trailers).
- **Clean:** syncthing, godot, openjdk/jdk, typst, servo, OpenTTD, endless-sky, shattered-pixel-dungeon (no tests), gleam, futhark, purescript, pimutils/khal (184 files), beancount/fava, nkanaev/yarr, iSoron/uhabits, YOURLS, dullage/flatnotes, philc/vimium, taskwarrior, lfos/calcurse, pimutils/todoman, pimutils/vdirsyncer, traggo/server, tryton/tryton (mirror), apache/syncope, moqui/mantle-usl.
- **Pin before AI starts:** etcd-io/etcd (2026-10-09), timewarrior (2025-10-03), akaunting (2026-03-16), sissbruecker/linkding (2026-08-05).
- **AI-heavy, skip:** pandoc, prometheus, vitess, temporal, pebble, seaweedfs, juicefs, rclone, hibernate-orm, scala3 (pin before 2025-05-23), calcite (pin before 2026-06-24), excalidraw, logseq, lichess, penpot, beets, Radicale (2026-01 on), babybuddy, spliit, mealie, Unciv, wger, Tandoor, ezbookkeeping, hledger (`.claude/` from 2026-09), openemr, Dolibarr, frappe/hrms.

## Vetted 2026-11-06 (AI agents)
Same method: AI files in any commit, plus AI co-author trailers, AI bot authors and "Generated with" lines, with the date of the first one.
- **Clean before a date:** ggozad/oterm (2026-02-25), TheR1D/shell_gpt (2026-05-06), sigoden/aichat (2025-08-02, Rust, no test files), khoj-ai/khoj (no AI markers, but it has a pricing page, so it counts as Enterprise), ItzCrazyKns/Perplexica (no tests), n3d1117/chatgpt-telegram-bot (no tests, 32 files), cheshire-cat-ai/core (no AI markers, but it calls itself a framework).
- **AI from early on, skip:** gptme (AI bot commits from 2024-11), plandex (aider commits from 2023), gpt-engineer (sweep bot from 2023), danielmiessler/Fabric (2024), icereed/paperless-gpt (2025-02), charmbracelet/mods (MCP support itself was Copilot co-authored), kardolus/chatgpt-cli (agent mode built with Claude, 2026-01), simonw/llm (`AGENTS.md` from 2025-06), big-AGI (2025-06), gpt-researcher (2025-01), 5ire (2025-07), agent-zero (2025-12), acon96/home-llm (2025-12).
- **Too big:** leon-ai/leon (2.4k files, `CLAUDE.md` at HEAD), py-gpt (5k files). open-interpreter's history was replaced in 2025.

