# String Functions & Data Wrangling — Advanced SQL Notes

**Platform:** Mode Analytics (now part of ThoughtSpot) | mode.com/sql-tutorial  
**Level:** Advanced  
**Dataset used:** `tutorial.billboard_top_100_year_end`, `tutorial.crunchbase_companies`

---

## Why This Matters

Real-world data is messy. Names have inconsistent casing, extra whitespace, embedded special characters, and concatenated fields that need splitting. String functions and data wrangling techniques are what turn raw, dirty columns into analyzable fields — this is most of what a junior analyst actually does day to day.

---

## Core String Functions

### LENGTH — character count

```sql
SELECT artist, LENGTH(artist) AS name_length
FROM tutorial.billboard_top_100_year_end;
```

Useful for finding suspiciously short/long values, or validating fixed-length codes (e.g., country codes should be LENGTH = 2 or 3).

---

### TRIM, LTRIM, RTRIM — remove whitespace

```sql
-- Remove leading and trailing spaces
SELECT TRIM(artist) FROM tutorial.billboard_top_100_year_end;

-- Remove only leading spaces
SELECT LTRIM(artist) FROM tutorial.billboard_top_100_year_end;

-- Remove only trailing spaces
SELECT RTRIM(artist) FROM tutorial.billboard_top_100_year_end;

-- Remove specific characters (not just whitespace)
SELECT TRIM(BOTH '$' FROM '$1500') AS cleaned;  -- returns '1500'
```

**This connects directly to the IS NULL pattern from Basic SQL:** `WHERE artist IS NULL OR TRIM(artist) = ''` catches both true NULLs and whitespace-only "empty" values.

---

### UPPER, LOWER — case normalisation

```sql
SELECT UPPER(artist) AS artist_upper,
       LOWER(artist) AS artist_lower
FROM tutorial.billboard_top_100_year_end;
```

**Critical use case:** standardising text before comparison or grouping. `'Drake'`, `'drake'`, and `'DRAKE'` are three different strings to SQL unless normalised.

```sql
-- Without normalisation, these are 3 separate groups
SELECT artist, COUNT(*)
FROM tutorial.billboard_top_100_year_end
GROUP BY artist;

-- With normalisation, they collapse into 1
SELECT LOWER(TRIM(artist)) AS artist_clean, COUNT(*)
FROM tutorial.billboard_top_100_year_end
GROUP BY 1;
```

---

### SUBSTRING — extract part of a string

```sql
-- SUBSTRING(string FROM start FOR length)
SELECT SUBSTRING(date_string FROM 1 FOR 4) AS year_part
FROM some_table;

-- Alternative syntax
SELECT SUBSTRING(date_string, 1, 4) AS year_part
FROM some_table;
```

Common use: extracting a year, area code, or fixed-position code from a longer string field.

---

### CONCAT and || — combine strings

```sql
-- CONCAT function (handles NULLs gracefully — treats as empty string)
SELECT CONCAT(artist, ' - ', song_name) AS display_name
FROM tutorial.billboard_top_100_year_end;

-- || operator (PostgreSQL) — NULL anywhere makes the whole result NULL
SELECT artist || ' - ' || song_name AS display_name
FROM tutorial.billboard_top_100_year_end;
```

**Rule:** if any concatenated column might be NULL, use `CONCAT()` or wrap with `COALESCE(column, '')` — otherwise `||` silently returns NULL for the entire row.

---

### REPLACE — substitute substrings

```sql
-- Remove all dollar signs and commas from a price string before casting
SELECT REPLACE(REPLACE(price_text, '$', ''), ',', '')::NUMERIC AS price_clean
FROM tutorial.crunchbase_companies;
```

Nesting REPLACE calls is the standard pattern for stripping multiple unwanted characters before a CAST.

---

### POSITION / STRPOS — find a substring's location

```sql
-- Find where '@' appears in an email address
SELECT email,
       POSITION('@' IN email) AS at_position
FROM some_table;

-- Extract the domain after the @
SELECT email,
       SUBSTRING(email FROM POSITION('@' IN email) + 1) AS domain
FROM some_table;
```

---

### SPLIT_PART — split a string by delimiter

```sql
-- Split 'San Francisco, CA' into city and state
SELECT SPLIT_PART(location, ',', 1) AS city,
       TRIM(SPLIT_PART(location, ',', 2)) AS state
FROM tutorial.crunchbase_companies;
```

`SPLIT_PART(string, delimiter, position)` — position is 1-indexed. This is the most common technique for breaking apart concatenated location/name fields.

---

## Data Wrangling — Putting It Together

### Pattern 1: Cleaning a messy text-based number column

```sql
SELECT company_name,
       REPLACE(REPLACE(TRIM(raised_amount_text), '$', ''), ',', '')::NUMERIC AS raised_amount
FROM tutorial.crunchbase_companies;
```

### Pattern 2: Standardising inconsistent category labels

```sql
SELECT CASE WHEN LOWER(TRIM(status)) IN ('acquired', 'acquired by')   THEN 'acquired'
            WHEN LOWER(TRIM(status)) IN ('operating', 'active', 'live') THEN 'operating'
            WHEN LOWER(TRIM(status)) IN ('closed', 'shut down', 'dead') THEN 'closed'
            ELSE 'unknown'
       END AS status_clean,
       COUNT(*)
FROM tutorial.crunchbase_companies
GROUP BY 1;
```

This combines CASE (from Intermediate) with TRIM/LOWER to collapse messy real-world category values into a clean set.

### Pattern 3: Parsing a combined field into components

```sql
SELECT permalink,
       SPLIT_PART(location, ',', 1)        AS city,
       TRIM(SPLIT_PART(location, ',', 2))  AS state_or_country,
       LENGTH(TRIM(name))                   AS name_length
FROM tutorial.crunchbase_companies
WHERE name IS NOT NULL AND TRIM(name) != '';
```

---

## When to Use These Techniques

- **TRIM + LOWER** before any GROUP BY, JOIN, or DISTINCT on a text column from raw/imported data — inconsistent casing and whitespace silently create duplicate groups
- **REPLACE chains** to strip currency symbols, commas, or unit suffixes before CAST to numeric
- **SPLIT_PART** to break apart "City, State" or "FirstName LastName" style fields
- **CASE + TRIM/LOWER** to standardise free-text category fields into a fixed, analyzable set
- **CONCAT over ||** whenever NULLs are possible in any of the concatenated columns
