# DISTINCT and CASE — Intermediate SQL Notes

**Platform:** Mode Analytics (now part of ThoughtSpot) | mode.com/sql-tutorial  
**Level:** Intermediate  
**Datasets used:** `tutorial.billboard_top_100_year_end`, `tutorial.aapl_historical_stock_price`

---

## DISTINCT

Returns unique values only — removes duplicate rows from the result. Applies to the entire row, not just one column.

```sql
-- All unique artist names in the dataset
SELECT DISTINCT artist
FROM tutorial.billboard_top_100_year_end
ORDER BY artist;

-- Unique combinations of year and artist
-- (same artist in multiple years = separate rows)
SELECT DISTINCT year, artist
FROM tutorial.billboard_top_100_year_end
ORDER BY year, artist;
```

**DISTINCT with COUNT:**

```sql
-- How many unique artists ever appeared in the top 100
SELECT COUNT(DISTINCT artist) AS unique_artists
FROM tutorial.billboard_top_100_year_end;

-- Unique artists per year
SELECT year,
       COUNT(DISTINCT artist) AS unique_artists
FROM tutorial.billboard_top_100_year_end
GROUP BY year
ORDER BY year;
```

**DISTINCT applies to all columns in SELECT:**

```sql
-- This returns unique (year, month) combinations — not unique years alone
SELECT DISTINCT year, month
FROM tutorial.aapl_historical_stock_price;
```

If you need unique values from one specific column while selecting others, use GROUP BY instead of DISTINCT.

**Performance note:** DISTINCT forces a deduplication step across the full result. On large tables, it is expensive. If you find yourself using DISTINCT to clean up unexpected duplicates, the real problem is usually in the JOIN — fix the join logic rather than masking duplicates with DISTINCT.

**When to use:**
- Viewing the unique values in a column to understand what's in a dataset
- Counting unique entities (unique customers, unique products, unique events)
- Quick data exploration on an unfamiliar table

---

## CASE Statement

Adds conditional logic inside a query. Works like if/then/else — evaluates conditions in order and returns the value from the first condition that is true.

**Basic syntax:**

```sql
CASE WHEN condition1 THEN result1
     WHEN condition2 THEN result2
     ELSE default_result
END AS column_alias
```

ELSE is optional. If no condition matches and there is no ELSE, the result is NULL.

---

### CASE for Bucketing / Categorising

```sql
-- Label each stock trading day by price range
SELECT date,
       close,
       CASE WHEN close >= 200 THEN 'high'
            WHEN close >= 100 THEN 'mid'
            ELSE 'low'
       END AS price_tier
FROM tutorial.aapl_historical_stock_price
ORDER BY date;
```

Conditions are evaluated top to bottom. Once a condition matches, the rest are skipped. Order matters — put the most restrictive condition first.

---

### CASE for Renaming or Standardising Values

```sql
-- Standardise inconsistent song rank labels
SELECT year,
       song_name,
       year_rank,
       CASE WHEN year_rank = 1  THEN 'Number One'
            WHEN year_rank <= 5 THEN 'Top Five'
            WHEN year_rank <= 10 THEN 'Top Ten'
            ELSE 'Outside Top Ten'
       END AS rank_label
FROM tutorial.billboard_top_100_year_end
ORDER BY year DESC, year_rank;
```

---

### CASE inside COUNT and SUM (Conditional Aggregation)

This is the most analytically powerful use of CASE. It lets you pivot data or count specific subsets without multiple queries.

```sql
-- Count how many songs per year fell in each rank tier
SELECT year,
       COUNT(CASE WHEN year_rank <= 10  THEN 1 END) AS top_10_count,
       COUNT(CASE WHEN year_rank <= 50  AND year_rank > 10 THEN 1 END) AS rank_11_50_count,
       COUNT(CASE WHEN year_rank > 50   THEN 1 END) AS bottom_50_count
FROM tutorial.billboard_top_100_year_end
GROUP BY year
ORDER BY year;
```

```sql
-- Sum volume only for days where the stock closed higher than it opened
SELECT year,
       SUM(CASE WHEN close > open THEN volume ELSE 0 END) AS up_day_volume,
       SUM(CASE WHEN close < open THEN volume ELSE 0 END) AS down_day_volume
FROM tutorial.aapl_historical_stock_price
GROUP BY year
ORDER BY year;
```

`COUNT(CASE WHEN condition THEN 1 END)` — the CASE returns 1 when true, NULL when false. COUNT ignores NULLs, so it only counts the rows where the condition was true.

---

### CASE in WHERE

```sql
-- Filter rows based on a conditional rule
SELECT *
FROM tutorial.aapl_historical_stock_price
WHERE CASE WHEN EXTRACT(YEAR FROM date) > 2010 THEN close > 300
           ELSE close > 100
      END;
```

This applies a different threshold depending on the year. Useful when filter logic itself needs to be conditional.

---

### CASE in ORDER BY

```sql
-- Sort: Number One songs first, then everything else by rank
SELECT year, song_name, year_rank
FROM tutorial.billboard_top_100_year_end
ORDER BY CASE WHEN year_rank = 1 THEN 0 ELSE 1 END,
         year DESC,
         year_rank;
```

**When to use CASE:**
- Bucketing continuous values into categories (age groups, price tiers, score bands)
- Standardising inconsistent labels in a column
- Conditional aggregation: count or sum only rows that meet a condition
- Pivoting rows to columns (turn category values into separate columns)
- Any time the output depends on a condition that varies per row
