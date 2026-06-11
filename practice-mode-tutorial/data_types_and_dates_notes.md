# Data Types, CAST/CONVERT & Date-Time Functions — Advanced SQL Notes

**Platform:** Mode Analytics (now part of ThoughtSpot) | mode.com/sql-tutorial  
**Level:** Advanced  
**Dataset used:** `tutorial.aapl_historical_stock_price`, `tutorial.us_housing_units`

---

## SQL Data Types — Core Categories

| Category | Common types | Notes |
|---|---|---|
| Integer | `INT`, `BIGINT`, `SMALLINT` | Whole numbers, no decimals |
| Decimal | `NUMERIC(p,s)`, `DECIMAL(p,s)`, `FLOAT` | Use NUMERIC for money — exact precision. FLOAT can introduce rounding errors |
| Text | `VARCHAR(n)`, `TEXT`, `CHAR(n)` | VARCHAR has a length limit, TEXT is unlimited, CHAR is fixed-length (padded) |
| Date/Time | `DATE`, `TIMESTAMP`, `TIMESTAMPTZ` | TIMESTAMPTZ stores timezone info — preferred for multi-region data |
| Boolean | `BOOLEAN` | TRUE / FALSE / NULL |

**Why types matter for analysis:** a column stored as text that "looks like" a number cannot be used in arithmetic or aggregate functions until cast. A common real-world problem: a CSV import stores `"1,200"` as text — SQL sees a string, not a number.

---

## CAST and CONVERT

`CAST` changes a value's data type. `CONVERT` is the SQL Server equivalent (rarely used in PostgreSQL/Mode — CAST is the standard).

```sql
-- CAST syntax
CAST(value AS target_type)

-- Shorthand syntax (PostgreSQL specific)
value::target_type
```

### Common CAST patterns

```sql
-- Text to integer
SELECT CAST('150' AS INTEGER);
SELECT '150'::INTEGER;          -- shorthand, same result

-- Text to numeric (preserves decimals)
SELECT CAST('19.99' AS NUMERIC);

-- Integer to text (for concatenation)
SELECT 'Order #' || CAST(order_id AS VARCHAR)
FROM tutorial.orders;

-- Text to date
SELECT CAST('2014-01-15' AS DATE);
SELECT '2014-01-15'::DATE;

-- Numeric division that avoids integer truncation
SELECT CAST(close AS NUMERIC) / CAST(volume AS NUMERIC) AS price_per_volume
FROM tutorial.aapl_historical_stock_price;
```

### The integer division trap

```sql
-- WRONG: integer / integer = integer (truncates decimals)
SELECT 5 / 2;          -- returns 2, not 2.5

-- CORRECT: cast at least one operand to NUMERIC or FLOAT
SELECT 5::NUMERIC / 2;       -- returns 2.5
SELECT CAST(5 AS NUMERIC) / 2; -- returns 2.5
```

This is one of the most common silent bugs in analytics SQL. Any ratio, percentage, or average calculated from integer columns needs an explicit cast.

**When to use CAST:**
- Converting text columns from raw imports into usable numeric/date types
- Fixing integer division before calculating ratios or percentages
- Preparing values for string concatenation (numbers must be cast to text first)
- Ensuring consistent types when UNIONing tables with mismatched column types

---

## Date and Time Functions

### EXTRACT — pull a specific component from a date

```sql
SELECT EXTRACT(YEAR  FROM occurred_at) AS yr,
       EXTRACT(MONTH FROM occurred_at) AS mo,
       EXTRACT(DAY   FROM occurred_at) AS dy,
       EXTRACT(DOW   FROM occurred_at) AS day_of_week  -- 0 = Sunday, 6 = Saturday
FROM tutorial.aapl_historical_stock_price;
```

### DATE_TRUNC — round a timestamp down to a specified precision

```sql
-- Truncate to the start of the week, month, year
SELECT DATE_TRUNC('week',  occurred_at) AS week_start,
       DATE_TRUNC('month', occurred_at) AS month_start,
       DATE_TRUNC('year',  occurred_at) AS year_start
FROM tutorial.aapl_historical_stock_price;
```

`DATE_TRUNC` is the standard way to bucket timestamps for GROUP BY in time-series reports. `EXTRACT` pulls a number out; `DATE_TRUNC` keeps it as a date/timestamp aligned to the period start — better for ORDER BY and charting.

### Date arithmetic

```sql
-- Add or subtract intervals
SELECT occurred_at + INTERVAL '7 days'  AS one_week_later,
       occurred_at - INTERVAL '1 month' AS one_month_earlier
FROM tutorial.aapl_historical_stock_price;

-- Difference between two dates
SELECT date_a - date_b AS days_between
FROM some_table;

-- Difference in specific units
SELECT EXTRACT(EPOCH FROM (timestamp_a - timestamp_b)) / 3600 AS hours_between
FROM some_table;
```

### NOW() and CURRENT_DATE

```sql
SELECT NOW();           -- current timestamp with timezone
SELECT CURRENT_DATE;    -- current date only

-- Rows from the last 30 days
SELECT *
FROM tutorial.aapl_historical_stock_price
WHERE occurred_at >= NOW() - INTERVAL '30 days';
```

### Formatting dates for display — TO_CHAR

```sql
SELECT TO_CHAR(occurred_at, 'YYYY-MM-DD')   AS iso_date,
       TO_CHAR(occurred_at, 'Mon DD, YYYY') AS readable_date,
       TO_CHAR(occurred_at, 'Day')          AS day_name
FROM tutorial.aapl_historical_stock_price;
```

| Format code | Meaning | Example |
|---|---|---|
| `YYYY` | 4-digit year | 2014 |
| `MM` | 2-digit month | 07 |
| `Mon` | abbreviated month name | Jul |
| `DD` | 2-digit day | 28 |
| `Day` | full day name | Monday |
| `HH24:MI:SS` | 24-hour time | 14:30:00 |

**When to use date functions:**
- `DATE_TRUNC` for any time-series GROUP BY (weekly, monthly, yearly trends)
- `EXTRACT` when you need a specific component as a standalone number (e.g., filtering "all January records across years" using `EXTRACT(MONTH FROM date) = 1`)
- `TO_CHAR` only for final display formatting — never for filtering or grouping (it converts to text, which sorts incorrectly)
- Date arithmetic for cohort windows, exposure periods, and "events in the last N days" filters
