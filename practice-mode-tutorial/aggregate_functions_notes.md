# Aggregate Functions — Intermediate SQL Notes

**Platform:** Mode Analytics (now part of ThoughtSpot) | mode.com/sql-tutorial  
**Level:** Intermediate  
**Datasets used:** `tutorial.aapl_historical_stock_price`, `tutorial.billboard_top_100_year_end`

---

## What Aggregate Functions Are

Aggregate functions collapse multiple rows into a single value. They run on a group of rows and return one result per group. Used with GROUP BY to produce summaries instead of raw data.

All aggregate functions ignore NULL values — except `COUNT(*)`.

---

## COUNT

Counts rows. Two forms with different behaviour:

```sql
-- COUNT(*): counts every row including NULLs
SELECT COUNT(*)
FROM tutorial.aapl_historical_stock_price;

-- COUNT(column): counts only rows where the column is NOT NULL
SELECT COUNT(high)
FROM tutorial.aapl_historical_stock_price;

-- Count rows per year
SELECT year,
       COUNT(*) AS trading_days
FROM tutorial.aapl_historical_stock_price
GROUP BY year
ORDER BY year;
```

**COUNT(*) vs COUNT(column):** use COUNT(*) to count all rows. Use COUNT(column) when you specifically want to know how many rows have a non-null value in that column.

**When to use:**
- Row counts per category (transactions per month, entries per region)
- Checking for missing data: compare COUNT(*) vs COUNT(column) — difference = NULL rows
- Volume metrics in dashboards

---

## SUM

Adds up all values in a numeric column. Ignores NULLs.

```sql
-- Total trading volume for the entire dataset
SELECT SUM(volume) AS total_volume
FROM tutorial.aapl_historical_stock_price;

-- Total volume per year
SELECT year,
       SUM(volume) AS annual_volume
FROM tutorial.aapl_historical_stock_price
GROUP BY year
ORDER BY year;

-- SUM with a derived calculation
SELECT year,
       SUM(high - low) AS total_daily_range
FROM tutorial.aapl_historical_stock_price
GROUP BY year;
```

**When to use:**
- Revenue totals, sales totals, volume totals
- Any metric where you need the cumulative value across a group

---

## MIN and MAX

Return the lowest and highest value in a column. Work on numbers, dates, and text (alphabetical order for text).

```sql
-- Highest and lowest closing price ever recorded
SELECT MIN(close) AS lowest_close,
       MAX(close) AS highest_close
FROM tutorial.aapl_historical_stock_price;

-- Per-year high and low
SELECT year,
       MIN(low)  AS yearly_low,
       MAX(high) AS yearly_high
FROM tutorial.aapl_historical_stock_price
GROUP BY year
ORDER BY year;

-- Earliest and latest date in the dataset
SELECT MIN(date) AS first_date,
       MAX(date) AS last_date
FROM tutorial.aapl_historical_stock_price;
```

**When to use:**
- Finding extremes (best performer, worst month, first/last date)
- Range calculations: `MAX(value) - MIN(value)`
- Date boundary checks on a dataset

---

## AVG

Returns the arithmetic mean of a numeric column. Ignores NULLs — the denominator is the count of non-null rows only.

```sql
-- Average closing price across all records
SELECT AVG(close) AS avg_close
FROM tutorial.aapl_historical_stock_price;

-- Average closing price per year
SELECT year,
       AVG(close) AS avg_annual_close
FROM tutorial.aapl_historical_stock_price
GROUP BY year
ORDER BY year;

-- Round to 2 decimal places
SELECT year,
       ROUND(AVG(close), 2) AS avg_close
FROM tutorial.aapl_historical_stock_price
GROUP BY year;
```

**NULL behaviour matters:** if 10 rows exist but 2 have NULL in the column, AVG divides by 8, not 10. If you want to include NULLs as zero: `AVG(COALESCE(column, 0))`.

**When to use:**
- Mean price, mean duration, mean score per category
- Benchmarking: compare a row's value against its group average (requires window functions — covered in Advanced)

---

## Using Multiple Aggregates Together

```sql
-- Full price summary per year
SELECT year,
       COUNT(*)            AS trading_days,
       MIN(low)            AS yearly_low,
       MAX(high)           AS yearly_high,
       ROUND(AVG(close),2) AS avg_close,
       SUM(volume)         AS total_volume
FROM tutorial.aapl_historical_stock_price
GROUP BY year
ORDER BY year;
```

This is the standard pattern for a summary report: one row per category, multiple metrics in columns.

---

## DISTINCT with COUNT

COUNT(DISTINCT column) counts unique values only.

```sql
-- How many unique artists appear in the dataset
SELECT COUNT(DISTINCT artist) AS unique_artists
FROM tutorial.billboard_top_100_year_end;

-- Unique artists per year
SELECT year,
       COUNT(DISTINCT artist) AS unique_artists
FROM tutorial.billboard_top_100_year_end
GROUP BY year
ORDER BY year;
```

**When to use:**
- Unique customer counts, unique product counts, unique event types
- Whenever "how many different X" is the question

---

## HAVING

Filters the results of a GROUP BY. WHERE filters rows before grouping; HAVING filters groups after aggregation.

```sql
-- Only years with more than 250 trading days recorded
SELECT year,
       COUNT(*) AS trading_days
FROM tutorial.aapl_historical_stock_price
GROUP BY year
HAVING COUNT(*) > 250
ORDER BY year;

-- Only years where average close price exceeded 100
SELECT year,
       ROUND(AVG(close), 2) AS avg_close
FROM tutorial.aapl_historical_stock_price
GROUP BY year
HAVING AVG(close) > 100
ORDER BY year;
```

**WHERE vs HAVING — the rule:**

| | WHERE | HAVING |
|---|---|---|
| Runs | Before GROUP BY | After GROUP BY |
| Filters | Individual rows | Aggregated groups |
| Can reference | Column values | Aggregate results |

You cannot use WHERE to filter on `COUNT(*) > 250` — the count doesn't exist yet when WHERE runs. That is what HAVING is for.

**When to use:**
- Filtering out low-volume groups (e.g., categories with fewer than 10 records)
- Returning only groups that meet a threshold (avg > x, sum > y)
- Any filter that depends on the result of an aggregate function
