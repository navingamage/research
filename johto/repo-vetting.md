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
| beancount/fava | Clean, 0 AI commits. Backup for Personal-scale. Gap not researched yet. |

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


## Vetted 2026-11-04: web frontend (Create × Web frontend × Enterprise)
Same method. Shallow clones were deepened past any merge boundary newer than 2025-01-20, because a shallow cut can hide when an AI file was first added.
- **Best:** element-hq/element-web, pinned at `ee5d2609df1421123e363d79c2e58efa200ad92b` (2026-03-16). Its first AI commit is 2026-03-17 and `AGENTS.md` arrives 2026-09-15. React 19/TS, about 1,380 source files, about 740 Jest test files and 138 Playwright specs; builds from npm plus a public codeload tarball.
- **Near-clean, recent pins:**
  - odoo/odoo `e3e47d97` (2026-04-27): OWL, 3.8k JS files; tests need Postgres and Chrome
  - mastodon `5b395774` (2026-04-29): thin frontend tests, no first-party pricing
  - zulip `715a9f31` (2026-02-06): jQuery + Handlebars
  - nextcloud/server `c8e2bb93` (2026-04-15)
  - nextcloud/calendar `45d2d8aa` (2026-05-20, small)
  - jupyterlab `148597d8` (2026-01-26)
  - joplin `89f9c6a5` (2026-02-18, Electron)
  - OpenSearch-Dashboards `cbcfe2fa` (2026-01-26, needs a backend)
  - nocodb `4f3d3749` (2026-02-24, almost no frontend unit tests)
  - bitwarden/clients `fdf47ffe` (2025-10-02)
- **Clean at HEAD:**
  - standardnotes/app (only 38 web tests)
  - ONLYOFFICE/web-apps (legacy code, needs sdkjs and DocumentServer)
  - cryptpad (no frontend tests)
- **Pins too old or AI-heavy:**
  - grafana (2025-03), superset (2025-07), discourse (2025-04, or 2025-06 if one Copilot commit is excused), gutenberg (2025-08), outline (2025-04, about 22 frontend tests), Rocket.Chat (2025-05), metabase (2025-04), Ghost (2025-06)
  - mattermost (2025-02), openproject (2025-01), appsmith (2025-01)
- **Rejected:**
  - twenty: AI files from before 2025
  - plane: no frontend tests
  - pdf.js: a library
  - etherpad and wekan: small, AI-heavy
  - PeerTube and jitsi: no frontend unit tests
  - bigbluebutton: needs the full stack
  - taiga-front: AngularJS 1.5
- **Sandbox note:** `codeload.github.com` and `cdn.sheetjs.com` are blocked by this session's proxy, so Element Web and Metabase can't install their dependencies here.
