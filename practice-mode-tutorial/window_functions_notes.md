# SQL Window Functions — Advanced SQL Notes

**Platform:** Mode Analytics (now part of ThoughtSpot) | mode.com/sql-tutorial  
**Level:** Advanced  
**Dataset used:** `tutorial.aapl_historical_stock_price`, `tutorial.billboard_top_100_year_end`

---

## What a Window Function Is

A window function performs a calculation across a set of rows related to the current row — without collapsing those rows into a single output row (unlike GROUP BY, which reduces multiple rows to one).

**The core difference from GROUP BY:** GROUP BY produces one row per group. A window function keeps every original row and adds a calculated column based on a "window" of related rows.

---

## General Syntax

```sql
function_name(column) OVER (
    [PARTITION BY column1, column2, ...]
    [ORDER BY column3, column4, ...]
    [ROWS/RANGE frame_specification]
)
```

- `PARTITION BY` — divides rows into groups (like GROUP BY, but doesn't collapse rows). Optional — without it, the window is the entire result set.
- `ORDER BY` — defines the order rows are processed in within each partition. Required for ranking and running-total functions.
- Frame specification — defines exactly which rows within the partition are included (covered below).

---

## ROW_NUMBER

Assigns a unique sequential integer to each row within a partition, based on the ORDER BY. No ties — every row gets a different number even if values are equal.

```sql
-- Number each song's rank position within its year
SELECT year,
       song_name,
       year_rank,
       ROW_NUMBER() OVER (PARTITION BY year ORDER BY year_rank) AS row_num
FROM tutorial.billboard_top_100_year_end;
```

### Common use: deduplication

```sql
-- Keep only the most recent record per user_id
WITH ranked AS (
    SELECT *,
           ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY occurred_at DESC) AS rn
    FROM some_table
)
SELECT *
FROM ranked
WHERE rn = 1;
```

This is the standard pattern for "give me the latest row per group" — extremely common in real analytics work where a table has multiple snapshot rows per entity.

---

## RANK and DENSE_RANK

Both assign a rank based on ORDER BY, but handle ties differently from ROW_NUMBER and from each other.

```sql
SELECT year,
       song_name,
       year_rank,
       ROW_NUMBER() OVER (PARTITION BY year ORDER BY year_rank) AS row_num,
       RANK()       OVER (PARTITION BY year ORDER BY year_rank) AS rank_val,
       DENSE_RANK() OVER (PARTITION BY year ORDER BY year_rank) AS dense_rank_val
FROM tutorial.billboard_top_100_year_end;
```

**Behaviour with ties** (e.g., two songs both ranked 5):

| Position | ROW_NUMBER | RANK | DENSE_RANK |
|---|---|---|---|
| 1st | 1 | 1 | 1 |
| 2nd (tie) | 2 | 2 | 2 |
| 3rd (tie) | 3 | 2 | 2 |
| 4th | 4 | 4 | 3 |

- `ROW_NUMBER` always increments by 1 — ignores ties, never repeats
- `RANK` repeats the rank for ties, then **skips** the next number(s) (2, 2, 4)
- `DENSE_RANK` repeats the rank for ties but does **not** skip (2, 2, 3)

**When to use which:**
- `ROW_NUMBER` for deduplication or when you need a strictly unique sequence
- `RANK` for leaderboard-style rankings where ties should share a position and skip the next (standard sports ranking behaviour)
- `DENSE_RANK` when you want consecutive rank values with no gaps, e.g., for paging through distinct rank tiers

---

## LAG and LEAD

Access a value from a different row in the same result set — without a self-join.

```sql
-- Compare each day's close price to the previous day's close
SELECT date,
       close,
       LAG(close)  OVER (ORDER BY date)  AS previous_close,
       LEAD(close) OVER (ORDER BY date)  AS next_close,
       close - LAG(close) OVER (ORDER BY date) AS day_over_day_change
FROM tutorial.aapl_historical_stock_price
ORDER BY date;
```

- `LAG(column, offset, default)` — looks backward `offset` rows (default 1)
- `LEAD(column, offset, default)` — looks forward `offset` rows (default 1)

```sql
-- Look back 7 rows, default to 0 if no row exists
SELECT date,
       close,
       LAG(close, 7, 0) OVER (ORDER BY date) AS close_7_days_ago
FROM tutorial.aapl_historical_stock_price;
```

**When to use:**
- Period-over-period comparisons (day-over-day, month-over-month change)
- Detecting changes in sequential data — flagging when a value differs from the previous row
- Session-gap detection (the technique used in the Search Functionality case — flag a new session when the time gap from `LAG(occurred_at)` exceeds 10 minutes)

---

## NTILE

Divides rows within a partition into a specified number of roughly equal-sized buckets, numbered 1 to N.

```sql
-- Split all stocks days into 4 quartiles by closing price
SELECT date,
       close,
       NTILE(4) OVER (ORDER BY close) AS price_quartile
FROM tutorial.aapl_historical_stock_price;
```

Quartile 1 = lowest 25% of close prices, Quartile 4 = highest 25%.

**When to use:**
- Splitting data into percentile/quartile/decile groups for analysis
- Customer segmentation (e.g., NTILE(10) on total spend = decile groups)
- A/B test bucketing into equal-sized groups

---

## Aggregate Functions as Window Functions

Any aggregate function (`SUM`, `AVG`, `COUNT`, `MIN`, `MAX`) can be used with `OVER()` to compute a value across a window without collapsing rows.

```sql
-- Each row shows its own value AND the average for its year — side by side
SELECT year,
       date,
       close,
       AVG(close) OVER (PARTITION BY year) AS yearly_avg_close,
       close - AVG(close) OVER (PARTITION BY year) AS diff_from_yearly_avg
FROM tutorial.aapl_historical_stock_price;
```

This is the cleaner alternative to the scalar-subquery pattern from the Subqueries notes — no need for a separate subquery to get the group average; it's computed inline per row.

### Running totals

```sql
-- Cumulative volume over time
SELECT date,
       volume,
       SUM(volume) OVER (ORDER BY date) AS running_total_volume
FROM tutorial.aapl_historical_stock_price
ORDER BY date;
```

Without `PARTITION BY`, `ORDER BY` inside `OVER()` makes this a **running total** — each row sums all rows up to and including itself.

### Frame specification — controlling the window explicitly

```sql
-- 7-day moving average (current row + 6 preceding rows)
SELECT date,
       close,
       AVG(close) OVER (
           ORDER BY date
           ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
       ) AS moving_avg_7day
FROM tutorial.aapl_historical_stock_price
ORDER BY date;
```

| Frame clause | Meaning |
|---|---|
| `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` | from the start of the partition to current row (default with ORDER BY) |
| `ROWS BETWEEN 6 PRECEDING AND CURRENT ROW` | current row + 6 rows before it (7-row window) |
| `ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING` | current row to the end of the partition |
| `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING` | the entire partition (same as no ORDER BY) |

**When to use window aggregates:**
- Comparing each row to its group's average/total/min/max without losing row-level detail
- Running totals and cumulative sums (finance, inventory)
- Moving averages for smoothing time-series data
- Any "this row vs. the group" calculation — almost always preferable to a correlated subquery

---

## Window Functions vs GROUP BY — Summary

| | GROUP BY | Window Function |
|---|---|---|
| Output rows | One per group | Same as input — every row preserved |
| Can mix raw + aggregated columns | No (must be in GROUP BY or aggregate) | Yes — raw and aggregated values side by side |
| Use case | Summary reports | Row-level analysis with group context |

A query can use **both**: GROUP BY to produce a summary table, and window functions within a CTE to rank or compare rows before the aggregation. They are complementary tools, not competing ones.
