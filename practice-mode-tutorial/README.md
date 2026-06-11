# practice-mode-tutorial

![Status](https://img.shields.io/badge/Status-Complete-green)
![Lessons](https://img.shields.io/badge/Lessons-52%2F52-success)

**Platform:** Mode Analytics (now part of ThoughtSpot) | [mode.com/sql-tutorial](https://mode.com/sql-tutorial)  
**Repo:** sql-analytics | Month 1 — SQL Mastery

---

## What this folder contains

Complete structured notes from the Mode SQL Tutorial — all 4 phases, 52 lessons, 16 files. Each file contains syntax patterns, annotated queries against Mode's live datasets, behavioural rules, and when-to-use guidance. Together these demonstrate the ability to write, debug, and reason about SQL across the full range required for analytics roles: from basic filtering through real-world business case analysis to performance-tuned advanced queries.

---

## Skills demonstrated

`SELECT` | `WHERE` | `LIMIT` | `ORDER BY` | `GROUP BY` | `HAVING` | `AND/OR/NOT` | `BETWEEN` | `IN` | `LIKE/ILIKE` | `IS NULL` | `COUNT/SUM/MIN/MAX/AVG` | `DISTINCT` | `CASE` | `INNER/LEFT/RIGHT/FULL JOIN` | `Self Join` | `UNION/UNION ALL` | `Multi-key joins` | `Subqueries` | `EXISTS/NOT EXISTS` | `CTEs` | `ROW_NUMBER/RANK/DENSE_RANK` | `LAG/LEAD` | `NTILE` | `Window aggregates` | `Running totals` | `Moving averages` | `CAST` | `Date functions` | `String functions` | `Data wrangling` | `Pivoting` | `Performance tuning` | `Cohort analysis` | `Session analysis` | `A/B test validation`

---

## Files — Beginner (15 lessons)

| File | Topics covered |
|---|---|
| `select_basics_notes.md` | SELECT, column aliasing, LIMIT, WHERE introduction |
| `filtering_operators_notes.md` | Comparison operators, BETWEEN, IN, LIKE/ILIKE, IS NULL, arithmetic columns |
| `logical_operators_notes.md` | AND, OR, NOT, operator precedence, ORDER BY, GROUP BY, SQL comments |

---

## Files — Intermediate (20 lessons)

| File | Topics covered |
|---|---|
| `aggregate_functions_notes.md` | COUNT, SUM, MIN, MAX, AVG, HAVING, COUNT(DISTINCT) |
| `joins_notes.md` | INNER/LEFT/RIGHT/FULL OUTER JOIN, Self Join, UNION/UNION ALL, WHERE vs ON, multi-key joins, joins with comparison operators |
| `distinct_and_case_notes.md` | DISTINCT, COUNT(DISTINCT), CASE bucketing, conditional aggregation, CASE in ORDER BY |

---

## Files — SQL Analytics Training (8 lessons)

| File | Case / topic | Business question answered |
|---|---|---|
| `case1_drop_in_engagement_notes.md` | Case 1 | What caused the weekly engagement drop? |
| `case2_search_functionality_notes.md` | Case 2 | Is search worth improving, and what specifically should change? |
| `case3_ab_test_validation_notes.md` | Case 3 | Are these A/B test results valid before we ship? |
| `core_principles_and_conclusion_notes.md` | All cases | The analytical framework connecting all three cases |

---

## Files — Advanced (9 lessons)

| File | Topics covered |
|---|---|
| `data_types_and_dates_notes.md` | Data types, CAST/CONVERT, integer division trap, EXTRACT, DATE_TRUNC, date arithmetic, TO_CHAR |
| `string_functions_and_wrangling_notes.md` | LENGTH, TRIM, UPPER/LOWER, SUBSTRING, CONCAT, REPLACE, SPLIT_PART, data cleaning patterns |
| `subqueries_notes.md` | Subqueries in FROM/WHERE/SELECT, IN vs EXISTS/NOT EXISTS, NULL trap, CTEs, subqueries vs JOINs |
| `window_functions_notes.md` | ROW_NUMBER, RANK, DENSE_RANK, LAG, LEAD, NTILE, window aggregates, running totals, moving averages |
| `pivoting_and_performance_notes.md` | Pivoting via CASE, performance tuning checklist |

---

## Datasets used across all phases

- `tutorial.us_housing_units` — US regional housing construction data
- `tutorial.billboard_top_100_year_end` — Billboard year-end top 100 charts
- `tutorial.aapl_historical_stock_price` — Apple historical stock price data
- `tutorial.crunchbase_companies` / `tutorial.crunchbase_investments` — Startup company and investment records
- `tutorial.yammer_users` / `tutorial.yammer_events` / `tutorial.yammer_emails` / `tutorial.yammer_experiments` — Yammer product analytics dataset (SQL Analytics Training)

---

## Progress

- [x] Basic SQL — 15 lessons complete
- [x] Intermediate SQL — 20 lessons complete
- [x] SQL Analytics Training — 8 lessons complete
- [x] Advanced SQL — 9 lessons complete

**Mode SQL Tutorial: 100% complete — 52 lessons across 16 files.**