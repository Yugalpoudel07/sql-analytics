# SELECT, LIMIT, WHERE — Basic SQL Notes

**Platform:** Mode Analytics (now part of ThoughtSpot) | mode.com/sql-tutorial  
**Level:** Basic  
**Dataset used:** `tutorial.us_housing_units`

---

## SELECT Statement

The starting point of every query. Tells the database which columns to return and which table to pull from. Running a SELECT query creates a temporary result — it does **not** modify the original data.

```sql
-- Select specific columns only
SELECT year,
       month,
       west
FROM tutorial.us_housing_units;

-- Select all columns
SELECT *
FROM tutorial.us_housing_units;
```

**Column aliasing with AS:**

```sql
SELECT west AS "West Region"
FROM tutorial.us_housing_units;
```

Use double quotes when the alias contains spaces. Without quotes, SQL treats each word as a separate identifier.

**When to use:**
- Always — every query starts with SELECT
- Use specific column names instead of `*` when working with wide tables
- Use AS to rename columns for cleaner output in reports or dashboards

---

## LIMIT

Restricts how many rows the query returns. Does not change the data. Without ORDER BY, the rows returned are not guaranteed to be meaningful — they are whatever the database returns first.

```sql
SELECT *
FROM tutorial.us_housing_units
LIMIT 100;
```

**When to use:**
- When previewing a new dataset for the first time
- When testing a query before running it on a full table
- When working with large production tables to avoid returning millions of rows

---

## WHERE Clause

Filters rows based on a condition. Every row is evaluated individually — only rows that meet the condition are included in the result.

```sql
SELECT *
FROM tutorial.us_housing_units
WHERE month = 1;
```

```sql
-- Combining WHERE with multiple conditions
SELECT *
FROM tutorial.us_housing_units
WHERE month = 1 AND west > 1000;
```

**Operators you can use inside WHERE:**

| Operator | Meaning |
|---|---|
| `=` | equal to |
| `!=` or `<>` | not equal to |
| `>` `<` `>=` `<=` | numerical comparison |
| `BETWEEN` | inclusive range |
| `IN` | matches a list of values |
| `LIKE` / `ILIKE` | pattern matching |
| `IS NULL` | checks for missing values |
| `AND` `OR` `NOT` | logical operators |

**When to use:**
- When analyzing a subset of data (e.g., one month, one year, one category)
- When cleaning or removing unwanted rows
- When preparing data for a report or dashboard
