# SQL Subqueries — Advanced SQL Notes

**Platform:** Mode Analytics (now part of ThoughtSpot) | mode.com/sql-tutorial  
**Level:** Advanced  
**Dataset used:** `tutorial.aapl_historical_stock_price`, `tutorial.billboard_top_100_year_end`, `tutorial.crunchbase_companies`, `tutorial.crunchbase_investments`

---

## What a Subquery Is

A query nested inside another query. The inner query (subquery) runs first, and its result is used by the outer query. Subqueries let you use an aggregated or filtered result as if it were a table or a single value.

A subquery can appear in four places:
1. In the `FROM` clause (acts as a temporary table)
2. In the `WHERE` clause (filters using a computed value or set)
3. In the `SELECT` clause (returns a single value per row)
4. In a `JOIN` (joins against a derived table)

---

## Subquery in FROM — Derived Tables

The subquery produces a result set that the outer query treats as a table. Must be given an alias.

```sql
-- Step 1 (inner): get yearly average close price
-- Step 2 (outer): find years where avg close was above 100
SELECT yearly_avg.year,
       yearly_avg.avg_close
FROM (
    SELECT year,
           AVG(close) AS avg_close
    FROM tutorial.aapl_historical_stock_price
    GROUP BY year
) AS yearly_avg
WHERE yearly_avg.avg_close > 100
ORDER BY yearly_avg.year;
```

**When to use:** when you need to filter or join on an aggregated result. WHERE cannot filter on aggregates directly (that's what HAVING is for) — but if the logic is complex, computing the aggregate in a subquery first and filtering in the outer query is often clearer than a single query with HAVING.

---

## Subquery in WHERE — Filtering with a Computed Value or Set

### Single value comparison

```sql
-- Find all stock days where close was above the all-time average
SELECT date, close
FROM tutorial.aapl_historical_stock_price
WHERE close > (
    SELECT AVG(close)
    FROM tutorial.aapl_historical_stock_price
);
```

The subquery returns exactly one value (the overall average), so it can be compared with `>`, `<`, `=`, etc.

### Filtering with IN — subquery returns a list

```sql
-- Find all companies that have at least one investment record
SELECT name
FROM tutorial.crunchbase_companies
WHERE permalink IN (
    SELECT DISTINCT company_permalink
    FROM tutorial.crunchbase_investments
);
```

### Filtering with NOT IN — find rows with no match

```sql
-- Find companies with NO investment records
SELECT name
FROM tutorial.crunchbase_companies
WHERE permalink NOT IN (
    SELECT DISTINCT company_permalink
    FROM tutorial.crunchbase_investments
    WHERE company_permalink IS NOT NULL  -- critical: see warning below
);
```

**NOT IN + NULL trap:** if the subquery's result set contains even one NULL, `NOT IN` returns zero rows for the entire query — NULL comparisons are unknown, not false, and this poisons the whole NOT IN check. **Always filter out NULLs in the subquery when using NOT IN**, or use `NOT EXISTS` instead (see below).

---

## EXISTS and NOT EXISTS

Tests whether the subquery returns any rows at all. Does not care about the actual values returned — only whether rows exist. Safer than IN/NOT IN because it isn't affected by NULLs.

```sql
-- Companies that have at least one investment (equivalent to the IN example above)
SELECT c.name
FROM tutorial.crunchbase_companies AS c
WHERE EXISTS (
    SELECT 1
    FROM tutorial.crunchbase_investments AS i
    WHERE i.company_permalink = c.permalink
);

-- Companies with NO investments — safe version, no NULL trap
SELECT c.name
FROM tutorial.crunchbase_companies AS c
WHERE NOT EXISTS (
    SELECT 1
    FROM tutorial.crunchbase_investments AS i
    WHERE i.company_permalink = c.permalink
);
```

This is a **correlated subquery** — the inner query references a column from the outer query (`c.permalink`). It runs once per row of the outer query conceptually (the optimizer usually rewrites this more efficiently, but logically that's the model).

**When to use EXISTS vs IN:**
- `EXISTS` / `NOT EXISTS` for existence checks, especially when NULLs might be present — always safe
- `IN` / `NOT IN` for short, known, NULL-free lists (e.g., `WHERE year IN (2012, 2013, 2014)`)
- Default to `NOT EXISTS` over `NOT IN` for "find unmatched rows" — it is the safer pattern

---

## Subquery in SELECT — Scalar Subqueries

Returns a single value per outer row. Must return exactly one row and one column, or it errors.

```sql
-- Show each year's avg close alongside the all-time average
SELECT year,
       AVG(close) AS yearly_avg_close,
       (SELECT AVG(close) FROM tutorial.aapl_historical_stock_price) AS all_time_avg
FROM tutorial.aapl_historical_stock_price
GROUP BY year;
```

This is useful for comparing each row/group against a single overall benchmark. Note: for comparing each row to its **own group's** average, a window function (`AVG() OVER (PARTITION BY ...)`) is usually cleaner — see Window Functions notes.

---

## Common Table Expressions (CTEs) — WITH clause

A CTE names a subquery so it can be referenced (and reused) in the main query. Improves readability over deeply nested subqueries.

```sql
WITH yearly_stats AS (
    SELECT year,
           AVG(close)  AS avg_close,
           SUM(volume) AS total_volume
    FROM tutorial.aapl_historical_stock_price
    GROUP BY year
)
SELECT year,
       avg_close,
       total_volume
FROM yearly_stats
WHERE avg_close > 100
ORDER BY year;
```

### Multiple CTEs, chained

```sql
WITH yearly_stats AS (
    SELECT year,
           AVG(close) AS avg_close
    FROM tutorial.aapl_historical_stock_price
    GROUP BY year
),
above_average_years AS (
    SELECT year
    FROM yearly_stats
    WHERE avg_close > (SELECT AVG(avg_close) FROM yearly_stats)
)
SELECT *
FROM above_average_years
ORDER BY year;
```

**When to use a CTE over a nested subquery:**
- When the same derived result is needed more than once in the query
- When nesting would go 2+ levels deep — CTEs read top-to-bottom like a script, nested subqueries read inside-out
- For any query you'll need to revisit or hand off — CTEs are dramatically easier to debug

---

## Subqueries vs JOINs — Choosing the Right Tool

| Goal | Use |
|---|---|
| Combine columns from two tables row-by-row | JOIN |
| Check if a row exists in another table | EXISTS / NOT EXISTS |
| Filter against an aggregate (single value) | Subquery in WHERE, or CTE |
| Compare each row to its group's aggregate | Window function (preferred) or correlated subquery |
| Reuse a derived dataset multiple times | CTE |
| Build a multi-step transformation | Chained CTEs |

**General rule:** if a JOIN can express the logic, prefer it — JOINs are usually more efficient and more familiar to read. Reach for subqueries/CTEs when the logic genuinely requires a result computed first and used as input to a second step.
