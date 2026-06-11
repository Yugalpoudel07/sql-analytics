# Case 2 — Analyzing In-Product Search Functionality

**Platform:** Mode Analytics (now part of ThoughtSpot) | mode.com/sql-tutorial  
**Level:** SQL Analytics Training  
**Dataset:** Yammer  
**Tables used:** `tutorial.yammer_events`

---

## The Business Problem

The product team is considering a revamp of Yammer's search feature. Before investing engineering time, they ask:

1. Is search worth improving at all — do people actually use it?
2. If yes, what specifically should be changed?

---

## The Search Event Flow

A user searching in Yammer produces this exact sequence in `yammer_events`:

```
User types in search box
        ↓
event_name = 'search_autocomplete'    ← fires as they type (before submitting)
        ↓
User hits enter / clicks search
        ↓
event_name = 'search_run'             ← a full search was executed
        ↓
User clicks a result
        ↓
event_name = 'search_click_result_1'  ← clicked the 1st result
event_name = 'search_click_result_2'  ← clicked the 2nd result
...
event_name = 'search_click_result_10' ← clicked the 10th result
```

All are `event_type = 'engagement'`.

**Autocomplete and full search are different features.** A user can satisfy their need from autocomplete and never run a full search. That is a success for autocomplete, not a failure for full search.

---

## The Analytical Framework — What "Good Search" Means

Define the purpose first: **search exists to help users find what they are looking for, quickly, with minimal effort.**

Every metric below is a measurement of whether that is happening.

| Metric | What it measures | Good signal | Bad signal |
|---|---|---|---|
| Session usage rate | Does anyone use search? | >20% of sessions | <5% of sessions |
| Searches per session | Does it work on first try? | 1–2 per session | 4+ per session (repeated refinement) |
| Clickthrough on results | Are results useful? | High click rate | Near-zero click rate |
| Click position distribution | Is ranking correct? | Front-weighted (most clicks on result 1–2) | Flat distribution across positions 1–10 |
| Month-over-month repeat usage | Does it retain users? | Users return to feature | Try-and-abandon pattern |

**Critical rule:** High frequency on a utility feature like search is a failure signal, not an engagement signal. Multiple searches per session = the user is not finding what they want.

---

## The Session Concept

A **session** = a continuous string of events from one user with no gap longer than 10 minutes between consecutive events. If a user goes 10 minutes without logging anything, the session ends.

Search frequency only makes sense within a session. A user searching 5 times in 8 minutes = repeated failed refinement. A user searching once on Monday and once on Wednesday = two separate, unrelated searches.

### Defining sessions in SQL using LAG()

```sql
SELECT user_id,
       occurred_at,
       event_name,
       -- Flag a new session when gap to previous event exceeds 10 minutes
       CASE WHEN occurred_at - LAG(occurred_at) OVER (PARTITION BY user_id ORDER BY occurred_at)
                 > INTERVAL '10 minutes'
            THEN 1
            ELSE 0
       END AS is_new_session
FROM tutorial.yammer_events
WHERE event_type = 'engagement';
```

Then assign session IDs using a cumulative sum of the new-session flags:

```sql
SELECT user_id,
       occurred_at,
       event_name,
       SUM(is_new_session) OVER (PARTITION BY user_id ORDER BY occurred_at) AS session_id
FROM (
    SELECT user_id,
           occurred_at,
           event_name,
           CASE WHEN occurred_at - LAG(occurred_at) OVER (PARTITION BY user_id ORDER BY occurred_at)
                     > INTERVAL '10 minutes'
                THEN 1
                ELSE 0
           END AS is_new_session
    FROM tutorial.yammer_events
    WHERE event_type = 'engagement'
) sub;
```

---

## Key SQL Patterns Used

### 1. Search session usage rate

```sql
-- What % of sessions include any search activity?
SELECT COUNT(DISTINCT CASE WHEN event_name IN ('search_autocomplete','search_run',
                                               'search_click_result_1','search_click_result_2',
                                               'search_click_result_3','search_click_result_4',
                                               'search_click_result_5','search_click_result_6',
                                               'search_click_result_7','search_click_result_8',
                                               'search_click_result_9','search_click_result_10')
                           THEN session_id END)::float
  / COUNT(DISTINCT session_id) AS search_session_pct
FROM sessions_table;  -- sessions_table = result of session assignment query above
```

---

### 2. Average searches per session (frequency check)

```sql
SELECT AVG(search_count) AS avg_searches_per_session
FROM (
    SELECT session_id,
           COUNT(CASE WHEN event_name = 'search_run' THEN 1 END) AS search_count
    FROM sessions_table
    GROUP BY session_id
    HAVING COUNT(CASE WHEN event_name = 'search_run' THEN 1 END) > 0
) search_sessions;
```

---

### 3. Result click rate (are results useful?)

```sql
SELECT COUNT(CASE WHEN event_name LIKE 'search_click_result_%' THEN 1 END)::float
     / NULLIF(COUNT(CASE WHEN event_name = 'search_run' THEN 1 END), 0) AS click_per_search
FROM sessions_table;
```

---

### 4. Click position distribution (ranking quality check)

```sql
SELECT event_name                AS result_position,
       COUNT(*)                  AS click_count,
       COUNT(*) * 100.0
         / SUM(COUNT(*)) OVER () AS pct_of_clicks
FROM tutorial.yammer_events
WHERE event_name LIKE 'search_click_result_%'
GROUP BY 1
ORDER BY 1;
```

**What to look for:** if `search_click_result_1` has 40–50% of all clicks and it drops sharply by result 3–4, the ranking is working. If all positions get roughly 10% each, the ranking algorithm is broken.

---

## What the Investigation Found

| Finding | Signal | Interpretation |
|---|---|---|
| Autocomplete used in ~25% of sessions | Moderate adoption | Users have genuine search need |
| Full search used in ~8% of sessions | Low adoption | Full search is a fallback, not primary |
| Autocomplete frequency: 1–2 per session | Low | Works on first attempt — healthy |
| Full search frequency: multiple per session | High | Repeated failed refinement — broken |
| Almost no clicks on full search results | Near zero | Results are not useful |
| Click distribution flat across positions 1–10 | Flat | Ranking algorithm is broken |
| Near-zero repeat usage of full search month-over-month | Try-and-abandon | Bad first experience drives abandonment |

---

## The Recommendation

**Autocomplete:** Leave it alone or make minor improvements. Healthy usage, low frequency per session, reasonable retention. It is doing its job.

**Full search:** The priority fix is the **ranking algorithm**. Results are returned but in the wrong order. Users are not clicking because the best result is not at position 1. Fix ranking before changing anything else about the UI or feature set.

**Additional insight:** Users run full search precisely when autocomplete fails them. Full search needs to return results that are meaningfully better than autocomplete — not the same results in list format.

---

## Analytical Skills This Case Teaches

**1. Define the feature's purpose before measuring it.**  
Search exists to help users find things quickly. Every metric flows from that definition. Without it, you are just counting events.

**2. High frequency on a utility feature is a failure signal.**  
More searches looks like more engagement. It is actually more frustration. This is counterintuitive and important.

**3. Distribution tells you more than totals.**  
Total clicks on search results tells you little. Where those clicks land tells you everything about ranking quality.

**4. Session-based analysis is the correct unit for search behaviour.**  
A search event only has meaning relative to what happened before and after it in the same session. Individual event counts are misleading without session context.
