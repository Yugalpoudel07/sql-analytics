# SQL Joins — Intermediate SQL Notes

**Platform:** Mode Analytics (now part of ThoughtSpot) | mode.com/sql-tutorial  
**Level:** Intermediate  
**Datasets used:** `tutorial.crunchbase_companies`, `tutorial.crunchbase_investments`

---

## What Joins Are

A JOIN combines rows from two tables based on a related column. Without a JOIN, SQL can only query one table at a time. Joins are the mechanism for answering questions that require data from multiple tables simultaneously.

Every join has three parts:
1. The **left table** (the one in FROM)
2. The **right table** (the one in JOIN)
3. The **join condition** (the ON clause — which columns to match on)

---

## INNER JOIN

Returns only rows where the join condition matches in **both** tables. Rows with no match in either table are excluded.

```sql
-- Returns only companies that have at least one investment record
SELECT companies.name          AS company_name,
       companies.status,
       investments.funded_year,
       investments.raised_amount_usd
FROM tutorial.crunchbase_companies  AS companies
INNER JOIN tutorial.crunchbase_investments AS investments
    ON companies.permalink = investments.company_permalink
ORDER BY investments.funded_year;
```

If a company has no investment records, it does not appear. If an investment record has no matching company, it does not appear.

**When to use:**
- When you only want records that exist in both tables
- The most common join type for clean, matched datasets
- When unmatched rows would make the result meaningless

---

## LEFT JOIN (LEFT OUTER JOIN)

Returns **all rows from the left table**, plus matching rows from the right table. Where no match exists in the right table, the right-table columns return NULL.

```sql
-- Returns ALL companies, with investment data where it exists
SELECT companies.name             AS company_name,
       companies.status,
       investments.funded_year,
       investments.raised_amount_usd
FROM tutorial.crunchbase_companies  AS companies
LEFT JOIN tutorial.crunchbase_investments AS investments
    ON companies.permalink = investments.company_permalink
ORDER BY companies.name;
```

Companies with no investments still appear — their `funded_year` and `raised_amount_usd` columns will be NULL.

**Practical use — find unmatched rows:**

```sql
-- Find companies that have NO investment records at all
SELECT companies.name
FROM tutorial.crunchbase_companies AS companies
LEFT JOIN tutorial.crunchbase_investments AS investments
    ON companies.permalink = investments.company_permalink
WHERE investments.company_permalink IS NULL;
```

This pattern (LEFT JOIN + WHERE right.column IS NULL) is the standard way to find records in one table that have no corresponding entry in another.

**When to use:**
- When the left table is your primary subject and you want to keep all of it
- Finding gaps: customers with no orders, products with no sales
- Most real analytics work uses LEFT JOIN more than INNER JOIN

---

## RIGHT JOIN (RIGHT OUTER JOIN)

Returns **all rows from the right table**, plus matching rows from the left table. Mirror of LEFT JOIN.

```sql
-- Returns ALL investments, with company data where it exists
SELECT companies.name,
       investments.funded_year,
       investments.raised_amount_usd
FROM tutorial.crunchbase_companies  AS companies
RIGHT JOIN tutorial.crunchbase_investments AS investments
    ON companies.permalink = investments.company_permalink;
```

In practice: RIGHT JOIN is rarely used. Any RIGHT JOIN can be rewritten as a LEFT JOIN by swapping the table order. Most teams standardise on LEFT JOIN for readability.

**When to use:**
- When the right table is your primary subject
- In practice: rewrite as LEFT JOIN instead — easier to read

---

## FULL OUTER JOIN

Returns **all rows from both tables**. Where a match exists, columns are populated. Where no match exists, the non-matching side returns NULL.

```sql
-- All companies and all investments — matched where possible
SELECT companies.name,
       investments.funded_year,
       investments.raised_amount_usd
FROM tutorial.crunchbase_companies  AS companies
FULL OUTER JOIN tutorial.crunchbase_investments AS investments
    ON companies.permalink = investments.company_permalink;
```

Rows with NULL on the left side = investments with no matching company.  
Rows with NULL on the right side = companies with no investments.

**When to use:**
- Auditing two datasets for completeness
- Reconciliation work: finding what exists in one table but not the other
- Less common in day-to-day analytics; most useful for data quality checks

---

## Join Type Summary

| Join Type | Left table rows | Right table rows |
|---|---|---|
| INNER JOIN | Only matched | Only matched |
| LEFT JOIN | All | Only matched (NULL if no match) |
| RIGHT JOIN | Only matched (NULL if no match) | All |
| FULL OUTER JOIN | All | All |

---

## WHERE vs ON

Both filter rows, but they run at different points in query execution.

```sql
-- ON: filters DURING the join (before rows are assembled)
SELECT companies.name,
       investments.funded_year
FROM tutorial.crunchbase_companies AS companies
LEFT JOIN tutorial.crunchbase_investments AS investments
    ON companies.permalink = investments.company_permalink
   AND investments.funded_year > 2010;
-- companies with no investments after 2010 still appear (NULLs)

-- WHERE: filters AFTER the join (removes rows from the assembled result)
SELECT companies.name,
       investments.funded_year
FROM tutorial.crunchbase_companies AS companies
LEFT JOIN tutorial.crunchbase_investments AS investments
    ON companies.permalink = investments.company_permalink
WHERE investments.funded_year > 2010;
-- companies with no investments after 2010 are removed entirely
```

**The rule:**
- Conditions that define the relationship → ON
- Conditions that filter the final result → WHERE
- For INNER JOIN: ON and WHERE produce the same result (no functional difference)
- For LEFT/RIGHT/FULL JOIN: ON and WHERE behave differently — use the wrong one and you silently convert a LEFT JOIN into an INNER JOIN

---

## Joining on Multiple Keys

Use AND in the ON clause when one key is not enough to uniquely identify a match.

```sql
-- Join on two columns: company permalink AND the funding round
SELECT companies.name,
       investments.funded_year,
       investments.funding_round_type
FROM tutorial.crunchbase_companies AS companies
INNER JOIN tutorial.crunchbase_investments AS investments
    ON companies.permalink     = investments.company_permalink
   AND investments.funding_round_type = 'series-a';
```

**When to use:**
- Composite keys: when a single column doesn't uniquely identify a row
- Filtering the join itself to a specific relationship type
- Date-range joins: `ON a.id = b.id AND b.date BETWEEN a.start AND a.end`

---

## Self Join

A table joined to itself. Requires aliases to distinguish the two instances.

```sql
-- Find all pairs of companies founded in the same year
SELECT a.name     AS company_a,
       b.name     AS company_b,
       a.founded_year
FROM tutorial.crunchbase_companies AS a
INNER JOIN tutorial.crunchbase_companies AS b
    ON  a.founded_year = b.founded_year
   AND  a.permalink    < b.permalink  -- prevents duplicate pairs (A,B) and (B,A)
ORDER BY a.founded_year;
```

The `a.permalink < b.permalink` condition is the standard trick to avoid returning both (A, B) and (B, A) as separate rows.

**When to use:**
- Comparing rows within the same table (employees and their managers, products and related products)
- Hierarchy traversal (parent-child relationships in one table)
- Finding pairs that share a common attribute

---

## UNION and UNION ALL

Stacks result sets from two queries vertically. Both queries must have the same number of columns in the same order with compatible data types.

```sql
-- UNION: combines results and removes duplicate rows
SELECT name, founded_year
FROM tutorial.crunchbase_companies
WHERE country_code = 'AUS'
UNION
SELECT name, founded_year
FROM tutorial.crunchbase_companies
WHERE country_code = 'NZL'
ORDER BY founded_year;

-- UNION ALL: combines results and KEEPS all rows including duplicates
SELECT name, founded_year
FROM tutorial.crunchbase_companies
WHERE country_code = 'AUS'
UNION ALL
SELECT name, founded_year
FROM tutorial.crunchbase_companies
WHERE country_code = 'NZL'
ORDER BY founded_year;
```

**UNION vs UNION ALL:**

| | UNION | UNION ALL |
|---|---|---|
| Duplicates | Removed | Kept |
| Performance | Slower (runs deduplication) | Faster |
| Use when | You need unique rows only | You need all rows + performance matters |

ORDER BY goes at the end of the final query only — it applies to the entire stacked result.

**When to use:**
- Combining data from the same schema across different time periods or regions
- Appending one table to another with the same structure
- Building datasets where records from multiple sources need to be treated as one

---

## Joins with Comparison Operators

The ON clause is not limited to equality. You can join on ranges and inequalities.

```sql
-- Join investments to a date range table
-- Returns investments that fall within each defined period
SELECT investments.company_permalink,
       investments.funded_year,
       periods.period_name
FROM tutorial.crunchbase_investments AS investments
INNER JOIN tutorial.funding_periods  AS periods
    ON investments.funded_year BETWEEN periods.start_year AND periods.end_year;
```

**When to use:**
- Time-bucketing: assigning events to periods
- Tolerance matching: joining records that are close but not identical
- Slowly changing dimensions: matching a record to the version of a lookup table that was valid at that time
