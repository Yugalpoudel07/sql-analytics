# Logical Operators & ORDER BY — Basic SQL Notes

**Platform:** Mode Analytics (now part of ThoughtSpot) | mode.com/sql-tutorial  
**Level:** Basic  
**Dataset used:** `tutorial.billboard_top_100_year_end`

---

## AND Operator

All conditions must be true for a row to be included. If any one condition is false, the row is excluded.

```sql
-- Returns top-10 songs from 2012 that include a featured artist
SELECT *
FROM tutorial.billboard_top_100_year_end
WHERE year = 2012
  AND year_rank <= 10
  AND "group_name" ILIKE '%feat%';
```

Each additional AND narrows the result. Use AND when you need all filters to apply simultaneously.

**When to use:**
- Filtering by multiple dimensions at once (year + rank + category)
- Narrowing a dataset to a precise segment

---

## OR Operator

At least one condition must be true. Returns a row if any one condition matches. Broadens results.

```sql
-- Returns songs ranked 5 OR songs by Gotye (regardless of rank)
SELECT *
FROM tutorial.billboard_top_100_year_end
WHERE year_rank = 5 OR artist = 'Gotye';
```

**AND + OR combined — always use parentheses:**

```sql
-- year = 2013 applies to BOTH OR branches because of the parentheses
SELECT *
FROM tutorial.billboard_top_100_year_end
WHERE year = 2013
  AND ("group_name" ILIKE '%macklemore%' OR "group_name" ILIKE '%timberlake%');
```

Without parentheses, SQL evaluates AND before OR — the logic breaks. Always group OR conditions in parentheses when mixing with AND.

**When to use:**
- Matching multiple possible values for one column (use IN instead if the list is 3+)
- Broadening a filter across different columns
- Always wrap OR groups in parentheses when combining with AND

---

## NOT Operator

Reverses a condition. Excludes rows that match.

```sql
-- All 2013 songs except those ranked 2 or 3
SELECT *
FROM tutorial.billboard_top_100_year_end
WHERE year = 2013
  AND year_rank NOT BETWEEN 2 AND 3;

-- All 2013 songs except Macklemore entries
SELECT *
FROM tutorial.billboard_top_100_year_end
WHERE year = 2013
  AND "group_name" NOT ILIKE '%macklemore%';

-- All 2013 songs with a recorded artist
SELECT *
FROM tutorial.billboard_top_100_year_end
WHERE year = 2013
  AND artist IS NOT NULL;
```

**NOT variations:**

| Form | Meaning |
|---|---|
| `NOT BETWEEN x AND y` | outside the range |
| `NOT ILIKE '%text%'` | does not contain pattern |
| `IS NOT NULL` | column has a value |
| `NOT IN (a, b, c)` | not in the list |

**When to use:**
- Excluding specific categories, keywords, or value ranges
- Cleaning data by removing known bad values
- Writing the exclusion is cleaner than listing every included value

---

## Logical Operators — Evaluation Order

SQL evaluates logical operators in this order:
1. NOT
2. AND
3. OR

This means `WHERE a OR b AND c` is read as `WHERE a OR (b AND c)` — not `WHERE (a OR b) AND c`.

**Rule:** always use parentheses when mixing AND and OR. Never rely on implicit precedence.

---

## ORDER BY

Sorts query results by one or more columns. Default is ascending (ASC). Does not filter — only changes the display order.

```sql
-- Sort alphabetically by artist (A → Z)
SELECT *
FROM tutorial.billboard_top_100_year_end
ORDER BY artist;

-- Sort descending: worst rank first
SELECT *
FROM tutorial.billboard_top_100_year_end
WHERE year = 2013
ORDER BY year_rank DESC;

-- Multi-column sort: newest year first, then rank 1→3 within each year
SELECT *
FROM tutorial.billboard_top_100_year_end
WHERE year_rank <= 3
ORDER BY year DESC, year_rank;

-- Column position shortcut (2nd column first, then 1st column descending)
SELECT *
FROM tutorial.billboard_top_100_year_end
WHERE year_rank <= 3
ORDER BY 2, 1 DESC;
```

**When to use:**
- Rankings and leaderboards
- Time-series output (newest or oldest first)
- Reports where row order matters for readability
- Always combine with LIMIT when previewing sorted data on large tables

---

## SQL Comments

Comments are ignored by the database engine. Use them to explain logic.

```sql
-- Single-line comment: explain what this filter does
SELECT *
FROM tutorial.billboard_top_100_year_end
WHERE year = 2013;

/*
  Multi-line comment:
  use for longer explanations
  or temporarily disabling a block
*/
SELECT *
FROM tutorial.billboard_top_100_year_end
WHERE year = 2013;
```

---

## GROUP BY

Groups rows with the same value so aggregate functions run per group instead of across the whole table. Each group produces one output row.

```sql
-- Count entries per year
SELECT year,
       COUNT(*) AS count
FROM tutorial.aapl_historical_stock_price
GROUP BY year;

-- Count entries per year and month
SELECT year,
       month,
       COUNT(*) AS count
FROM tutorial.aapl_historical_stock_price
GROUP BY year, month;

-- Column position shorthand
SELECT year,
       month,
       COUNT(*) AS count
FROM tutorial.aapl_historical_stock_price
GROUP BY 1, 2;

-- GROUP BY + ORDER BY
SELECT year,
       month,
       COUNT(*) AS count
FROM tutorial.aapl_historical_stock_price
GROUP BY year, month
ORDER BY month, year;
```

**Execution order:** GROUP BY runs before LIMIT. LIMIT only restricts the final displayed rows — the aggregation is always computed on the full grouped dataset first.

**Rule:** every column in SELECT that is not inside an aggregate function must appear in GROUP BY.

**When to use:**
- Summarising data by category (totals, counts, averages per year/month/region)
- Building report breakdowns instead of returning raw rows
- Any time you use COUNT, SUM, AVG, MIN, or MAX on a subset of data
