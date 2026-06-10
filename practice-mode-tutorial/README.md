# practice-mode-tutorial — Intermediate

![Status](https://img.shields.io/badge/Status-Complete-green)
![Level](https://img.shields.io/badge/Level-Intermediate-orange)

**Platform:** Mode Analytics (now part of ThoughtSpot) | [mode.com/sql-tutorial](https://mode.com/sql-tutorial)  
**Repo:** sql-analytics | Month 1 — SQL Mastery

---

## Skills demonstrated

`COUNT` | `SUM` | `MIN` | `MAX` | `AVG` | `HAVING` | `INNER JOIN` | `LEFT JOIN` | `RIGHT JOIN` | `FULL OUTER JOIN` | `Self Join` | `UNION / UNION ALL` | `WHERE vs ON` | `Multi-key joins` | `Join with comparison operators` | `DISTINCT` | `COUNT(DISTINCT)` | `CASE` | `Conditional aggregation`

---

## What this folder contains

Structured reference notes covering all 20 lessons in the Mode SQL Tutorial Intermediate section. Topics are grouped into three files by concept cluster. Each file contains: syntax patterns, annotated queries against Mode's datasets, behavioural rules, and when-to-use guidance.

These notes demonstrate the ability to combine data across multiple tables, aggregate and filter grouped results, and apply conditional logic inline — the core skills tested in AU/NZ analytics take-home assessments.

---

## Files

| File | Topics covered |
|---|---|
| `aggregate_functions_notes.md` | COUNT, SUM, MIN, MAX, AVG, HAVING, COUNT(DISTINCT) |
| `joins_notes.md` | INNER JOIN, LEFT JOIN, RIGHT JOIN, FULL OUTER JOIN, Self Join, UNION/UNION ALL, WHERE vs ON, multi-key joins, joins with comparison operators |
| `distinct_and_case_notes.md` | DISTINCT, COUNT(DISTINCT), CASE bucketing, conditional aggregation, CASE in ORDER BY |

---

## Datasets used

- `tutorial.aapl_historical_stock_price` — Apple historical stock price data (Mode built-in)
- `tutorial.billboard_top_100_year_end` — Billboard year-end top 100 charts (Mode built-in)
- `tutorial.crunchbase_companies` — Startup company records (Mode built-in)
- `tutorial.crunchbase_investments` — Startup investment records (Mode built-in)

---

## Progress

- [x] Basic SQL — 15 lessons complete
- [x] Intermediate SQL — 20 lessons complete
- [ ] SQL Analytics Training — 8 lessons
- [ ] Advanced SQL — 9 lessons
