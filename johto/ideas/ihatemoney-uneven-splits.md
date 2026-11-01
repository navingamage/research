# ihatemoney: uneven bill splits

- **Repo:** https://github.com/spiral-project/ihatemoney/tree/e66a7672e8e5c41549bf53c4a824c72c43ab9079
- **Stack:** Python, Flask, SQLAlchemy, Alembic
- **Domain:** Splitting shared expenses (personal-scale codebase)
- **Status:** active, next up

## Pre-flight (checked 2026-11-01)
- It's a real application: a self-hostable web app with a REST API for sharing group expenses. It uses Flask, SQLAlchemy, Alembic migrations and sqlalchemy-continuum for history.
- **Personal-scale:** 235 files in total, 46 Python files, about 3k lines of app code. Tests are pytest on SQLite, about 130 test functions in api, budget, history, import and main.
- There's no pricing page, and the hosted ihatemoney.org is free.
- Not one AI co-authored commit in the repo's whole history, and no AI files. It isn't a fork and isn't in SWE-bench.
- Dependencies come from PyPI. The currency rates API (exchangerate.host) gets mocked in tests, so nothing paid or live is needed.
- Pin `e66a767` is from 2026-08-07, the HEAD of main.

## Gap
- Bills are always split **evenly**, weighted only by each member's project-wide `Person.weight`. The `billowers` table holds only bill_id and person_id, with no per-bill share. An old commit even reads "show '... each' only when ... the split is even".
- So there's no way to record who owes what on a single bill, like a dinner where each person had a different meal.
- No branch or commit adds one. `Glandos/issue1054` is empty compared with main.

## Where a solution can go wrong
- **Money is stored as `Float`** (`amount` and `converted_amount`). Exact shares have to add up to the total, and leftover cents need a rule.
- **Currency:** each bill has its own `original_currency` and `converted_amount`, and a project's currency can be switched later (the "Rework currency switching" commit converts every bill). Shares have to convert too and still add up.
- **Every money figure has to agree:**
  - `Project.balance` and the spent/paid/transferred logic around `models.py:136`
  - settle (the minimum set of transfers)
  - statistics, including monthly totals
  - per-member views
- **Weights:** right now changing a member's weight changes old bills after the fact. Per-bill shares shouldn't change that way, and both kinds of bill have to work side by side. `uses_weights` and `get_bill_weights` in `models.py`.
- **Edge cases:**
  - members who have been deactivated
  - reimbursement bills (`BillType`)
  - archives
  - editing a bill after it was saved (including changing who owes)
- **History:** the continuum versioned tables and `history.py` have to show changes to shares.
- **Import and export:** JSON and CSV round-trips (`import_test.py`) and settle exports.
- **API backward compatibility:** the `owers` field and existing clients, including the mobile apps listed in `download_mobile_app.html`.
- **Migration:** an Alembic migration that keeps existing data.

## Verifying in a lane
- Set up with `uv venv && uv pip install -e .[dev]` (Python 3.10+), then run `pytest ihatemoney/tests`.
- A small repo is easier for the model, so try a short check run first. If it finishes cleanly in about 15 minutes, drop the idea.
