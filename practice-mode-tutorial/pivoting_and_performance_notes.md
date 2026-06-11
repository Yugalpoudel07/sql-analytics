# Pivoting Data & Performance Tuning — Advanced SQL Notes

**Platform:** Mode Analytics (now part of ThoughtSpot) | mode.com/sql-tutorial  
**Level:** Advanced  
**Dataset used:** `tutorial.billboard_top_100_year_end`, `tutorial.aapl_historical_stock_price`

---

## Pivoting Data — Rows to Columns

A pivot transforms row values into column headers. SQL has no native `PIVOT` keyword in PostgreSQL (unlike SQL Server) — pivoting is done with conditional aggregation using `CASE` inside aggregate functions.

### The pattern: CASE + aggregate function + GROUP BY

```sql
-- Rows-based: one row per (year, rank_tier)
SELECT year,
       CASE WHEN year_rank <= 10 THEN 'top_10'
            WHEN year_rank <= 50 THEN 'rank_11_50'
            ELSE 'rank_51_100'
       END AS rank_tier,
       COUNT(*) AS song_count
FROM tutorial.billboard_top_100_year_end
GROUP BY 1, 2;
```

```sql
-- Pivoted: one row per year, one column per rank_tier
SELECT year,
       COUNT(CASE WHEN year_rank <= 10  THEN 1 END) AS top_10_count,
       COUNT(CASE WHEN year_rank BETWEEN 11 AND 50 THEN 1 END) AS rank_11_50_count,
       COUNT(CASE WHEN year_rank > 50  THEN 1 END) AS rank_51_100_count
FROM tutorial.billboard_top_100_year_end
GROUP BY year
ORDER BY year;
```

The second version is the pivot — what would have been separate rows (one per tier) are now separate columns, one row per year.

### Pivoting with SUM for value totals

```sql
-- Total volume split into up-days vs down-days, per year
SELECT year,
       SUM(CASE WHEN close > open THEN volume ELSE 0 END) AS up_day_volume,
       SUM(CASE WHEN close < open THEN volume ELSE 0 END) AS down_day_volume,
       SUM(CASE WHEN close = open THEN volume ELSE 0 END) AS flat_day_volume
FROM tutorial.aapl_historical_stock_price
GROUP BY year
ORDER BY year;
```

### Pivoting with a fixed, known set of categories

```sql
-- One column per device category
SELECT user_id,
       COUNT(CASE WHEN device_category = 'phone'    THEN 1 END) AS phone_events,
       COUNT(CASE WHEN device_category = 'tablet'   THEN 1 END) AS tablet_events,
       COUNT(CASE WHEN device_category = 'computer' THEN 1 END) AS computer_events
FROM tutorial.yammer_events
GROUP BY user_id;
```

**Limitation:** this approach requires knowing the category values in advance — each category needs its own CASE/column written explicitly. There is no dynamic pivot in standard SQL; if categories are unknown or change frequently, pivoting is usually done in the application layer (pandas, Excel, BI tool) instead.

**When to use pivoting:**
- Turning a long/tidy dataset into a wide summary table for reporting or dashboards
- Building "one row per entity, one column per category" views
- Preparing data for tools that expect wide format (e.g., spreadsheet exports, certain chart types)

---

## Performance Tuning SQL Queries

Performance tuning matters once datasets get large (millions+ rows) or queries run frequently. The goal: reduce the amount of data the database has to scan, sort, or join.

### 1. Filter early — push WHERE conditions as early as possible

```sql
-- LESS EFFICIENT: filters after joining and aggregating everything
SELECT year, AVG(close)
FROM (
    SELECT a.*, b.something
    FROM tutorial.aapl_historical_stock_price a
    JOIN some_other_table b ON a.id = b.id
) joined
WHERE year = 2014
GROUP BY year;

-- MORE EFFICIENT: filter the large table BEFORE the join
SELECT a.year, AVG(a.close)
FROM (
    SELECT * FROM tutorial.aapl_historical_stock_price WHERE year = 2014
) a
JOIN some_other_table b ON a.id = b.id
GROUP BY a.year;
```

Filtering before a join means the join processes fewer rows. The query optimizer often does this automatically for simple cases, but it's not guaranteed — especially with subqueries containing aggregates or window functions, where the optimizer cannot push filters through.

---

### 2. Avoid SELECT * — name only the columns you need

```sql
-- Wasteful: pulls every column, even unused ones
SELECT * FROM tutorial.aapl_historical_stock_price WHERE year = 2014;

-- Efficient: only what's needed
SELECT date, close, volume FROM tutorial.aapl_historical_stock_price WHERE year = 2014;
```

Matters more on wide tables (50+ columns) and when the result feeds into another step (less data to move/serialize).

---

### 3. Use LIMIT while developing/testing queries

```sql
-- Test the query logic on a small sample first
SELECT *
FROM tutorial.aapl_historical_stock_price
WHERE year = 2014
LIMIT 100;
```

Remove the LIMIT only once the query logic is confirmed correct. Running an unfiltered query against a billion-row table while debugging wastes compute and time.

---

### 4. Avoid functions on indexed/filtered columns in WHERE

```sql
-- BAD: wrapping the column in a function prevents index usage
SELECT * FROM tutorial.aapl_historical_stock_price
WHERE EXTRACT(YEAR FROM date) = 2014;

-- GOOD: compare against a date range instead — index-friendly
SELECT * FROM tutorial.aapl_historical_stock_price
WHERE date >= '2014-01-01' AND date < '2015-01-01';
```

Applying a function to a column in WHERE forces the database to compute that function for every row before it can filter — it can't use an index efficiently. Rewriting as a range comparison on the raw column lets the database use an index if one exists.

---

### 5. JOIN order and join type awareness

- `INNER JOIN` is generally cheaper than `LEFT JOIN` because it doesn't have to track unmatched rows
- Join on indexed columns (typically primary keys / foreign keys) — joining on unindexed text columns is significantly slower
- When joining a large table to a small lookup table, most query engines handle this efficiently regardless of order — but for very large tables joined together, putting the more selective filter on the larger table first reduces the rows the join has to process

---

### 6. CTEs vs subqueries — performance is usually equivalent, but materialisation can differ

In some databases, a CTE referenced multiple times may be computed once and reused (materialised), while in others it's recomputed each time it's referenced — equivalent to inlining the subquery. If a CTE is expensive and used multiple times, check whether the specific database materialises it; if not, consider a temporary table instead.

```sql
-- If yearly_stats is expensive and used twice, a temp table guarantees single computation
CREATE TEMP TABLE yearly_stats AS
SELECT year, AVG(close) AS avg_close, SUM(volume) AS total_volume
FROM tutorial.aapl_historical_stock_price
GROUP BY year;

SELECT * FROM yearly_stats WHERE avg_close > 100;
SELECT * FROM yearly_stats WHERE total_volume > 1000000;
```

---

### 7. COUNT(DISTINCT ...) is expensive — use sparingly on large tables

`COUNT(DISTINCT column)` requires the database to track every unique value seen, which is memory-intensive on large datasets. If an approximate count is acceptable, some databases offer approximate distinct functions (e.g., `APPROX_COUNT_DISTINCT` in some warehouses) that trade exactness for speed.

---

## Performance Tuning — Quick Checklist

| Check | Why |
|---|---|
| Are WHERE filters as early/specific as possible? | Reduces rows processed downstream |
| Is SELECT * avoided? | Reduces data volume moved |
| Are functions avoided on filtered columns? | Allows index usage |
| Is LIMIT used during development? | Avoids full-table scans while testing |
| Are joins on indexed columns? | Major speed difference on large tables |
| Is COUNT(DISTINCT) necessary, or would COUNT(*) after pre-filtering work? | DISTINCT is expensive at scale |
| Is an expensive CTE reused — should it be a temp table? | Avoids redundant computation |

**The broader principle:** performance tuning is about minimizing the amount of data the database touches at each step. Every technique above reduces either the number of rows scanned, the number of columns moved, or the number of times a computation is repeated.
