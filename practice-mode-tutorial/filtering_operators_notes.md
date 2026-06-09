# Filtering Operators — Basic SQL Notes

**Platform:** Mode Analytics (now part of ThoughtSpot) | mode.com/sql-tutorial  
**Level:** Basic  
**Datasets used:** `tutorial.billboard_top_100_year_end`, `tutorial.us_housing_units`

---

## Comparison Operators

Used in WHERE to compare column values. Work with both numbers and text.

```sql
-- Numerical: returns months where West region produced more than 30k housing units
SELECT *
FROM tutorial.us_housing_units
WHERE west > 30;

-- Text not-equal: removes January from results
SELECT *
FROM tutorial.us_housing_units
WHERE month_name != 'January';

-- Text comparison is alphabetical
SELECT *
FROM tutorial.us_housing_units
WHERE month_name > 'January';
-- returns months that come after January alphabetically
```

**Arithmetic on columns:**

```sql
-- Derived column: adds west + south values per row
SELECT year,
       month,
       west,
       south,
       west + south AS south_plus_west
FROM tutorial.us_housing_units;

-- Average of two regions per row
SELECT year,
       month,
       (west + south) / 2 AS south_west_avg
FROM tutorial.us_housing_units;
```

Always use single quotes for text values. Double quotes are for column/table names.

---

## BETWEEN Operator

Filters rows within an inclusive range. Equivalent to writing `>= AND <=` but more readable.

```sql
-- Returns songs ranked 5 through 10 (inclusive)
SELECT *
FROM tutorial.billboard_top_100_year_end
WHERE year_rank BETWEEN 5 AND 10;
```

```sql
-- Equivalent explicit form
SELECT *
FROM tutorial.billboard_top_100_year_end
WHERE year_rank >= 5
  AND year_rank <= 10;
```

**When to use:**
- Numeric ranges: ranks, scores, prices
- Date ranges: `BETWEEN '2020-01-01' AND '2020-12-31'`
- Use the explicit `>= AND <=` form when the inclusive boundary behaviour needs to be obvious to a reader

---

## IN Operator

Matches any value from a defined list. Cleaner replacement for multiple OR conditions.

```sql
-- Numerical list
SELECT *
FROM tutorial.billboard_top_100_year_end
WHERE year_rank IN (1, 2, 3);

-- Text list
SELECT *
FROM tutorial.billboard_top_100_year_end
WHERE artist IN ('Taylor Swift', 'Usher', 'Ludacris');
```

**IN vs OR — same result, different readability:**

```sql
-- OR version (verbose)
WHERE artist = 'Taylor Swift'
   OR artist = 'Usher'
   OR artist = 'Ludacris'

-- IN version (clean)
WHERE artist IN ('Taylor Swift', 'Usher', 'Ludacris')
```

**When to use:**
- When filtering 3 or more known values — IN is always cleaner than chained OR
- When filtering by category, ID, year, or name lists

---

## LIKE and ILIKE Operators

Pattern matching for text columns. ILIKE is the case-insensitive version (PostgreSQL-specific).

**Wildcards:**
- `%` — matches any number of characters (including zero)
- `_` — matches exactly one character

```sql
-- Starts with "Snoop" (case-sensitive)
SELECT *
FROM tutorial.billboard_top_100_year_end
WHERE "group_name" LIKE 'Snoop%';

-- Starts with "snoop" — case-insensitive
SELECT *
FROM tutorial.billboard_top_100_year_end
WHERE "group_name" ILIKE 'snoop%';

-- Single character wildcard: matches "drake", "droke", etc.
SELECT *
FROM tutorial.billboard_top_100_year_end
WHERE artist ILIKE 'dr_ke';
```

**Common patterns:**

```sql
LIKE 'A%'       -- starts with A
LIKE '%son'     -- ends with son
LIKE '%love%'   -- contains love anywhere
LIKE 'A__B'     -- A, then exactly two characters, then B
```

**When to use:**
- Partial name or keyword searches
- Filtering inconsistent text data where exact values are unknown
- Use ILIKE over LIKE unless case sensitivity is specifically required

---

## IS NULL

Checks for missing or unknown values. NULL cannot be compared with `=` or `!=` — you must use IS NULL or IS NOT NULL.

```sql
-- Finds rows where artist was not recorded
SELECT *
FROM tutorial.billboard_top_100_year_end
WHERE artist IS NULL OR TRIM(artist) = '';

-- Removes rows with missing artist data
SELECT *
FROM tutorial.billboard_top_100_year_end
WHERE year = 2013
  AND artist IS NOT NULL;
```

`TRIM(column) = ''` catches columns that contain only spaces — NULL check alone misses those.

**When to use:**
- Data quality checks before analysis
- Removing incomplete records
- Counting how many rows are missing a value: `COUNT(*) WHERE column IS NULL`
