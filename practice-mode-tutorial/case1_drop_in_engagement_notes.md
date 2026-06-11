# Case 1 — Investigating a Drop in User Engagement

**Platform:** Mode Analytics (now part of ThoughtSpot) | mode.com/sql-tutorial  
**Level:** SQL Analytics Training  
**Dataset:** Yammer (fictional workplace communication tool)  
**Tables used:** `tutorial.yammer_users`, `tutorial.yammer_events`, `tutorial.yammer_emails`

---

## The Business Problem

A weekly engaged users chart showed steady growth from January through late July 2014, then a clear downward dip starting the week of July 28. The head of product asks: what caused the drop?

**Engagement definition used:** a user who logged at least one event with `event_type = 'engagement'` during that week.

---

## Schema Reference

**tutorial.yammer_users**

| Column | Type | Description |
|---|---|---|
| user_id | integer | Unique user identifier |
| created_at | timestamp | Account creation date |
| state | varchar | 'active' or 'pending' |
| activated_at | timestamp | Account activation date |
| company_id | integer | Company the user belongs to |
| language | varchar | User language setting |

**tutorial.yammer_events**

| Column | Type | Description |
|---|---|---|
| user_id | integer | User who performed the action |
| occurred_at | timestamp | When the event happened |
| event_type | varchar | 'engagement' or 'signup_flow' |
| event_name | varchar | Specific action: 'login', 'home_page', 'like_message', 'send_message', etc. |
| location | varchar | Country |
| device | varchar | Device name: 'iphone', 'macbook pro', 'samsung galaxy', etc. |

**tutorial.yammer_emails**

| Column | Type | Description |
|---|---|---|
| user_id | integer | The user |
| occurred_at | timestamp | When the email event happened |
| action | varchar | 'sent_weekly_digest', 'sent_reengagement_email', 'email_open', 'email_clickthrough' |

---

## Hypothesis List — Correct Priority Order

| Priority | Hypothesis | Why this order |
|---|---|---|
| 1 | Data / tracking issue | If data is broken, all other analysis is invalid. Rule out first. |
| 2 | Holiday / seasonal drop | Simplest explanation. Late July = northern hemisphere summer. No code fix required. |
| 3 | Device-specific problem | Narrow the surface area — is it all users or only phone users? |
| 4 | Cohort / retention problem | Are new users fine but long-time users dropping off? |
| 5 | Email digest failure | Is the re-engagement channel down? Check yammer_emails. |
| 6 | Regional outage | Is it one country or global? Slice by location. |
| 7 | Large account churn | Did one big company leave? Slice by company_id. |
| 8 | Product bug / UI change | Broad catch-all after narrowing above. |

**Rule:** Always rule out data/tracking issues before drawing any product conclusions. If the logging pipeline broke, the drop in the chart is not real.

---

## Key SQL Patterns Used

### 1. Weekly event volume by event_type (tracking check)

```sql
SELECT DATE_TRUNC('week', occurred_at) AS week,
       event_type,
       COUNT(*)                         AS event_count
FROM tutorial.yammer_events
GROUP BY 1, 2
ORDER BY 1, 2;
```

If `event_type = 'engagement'` drops but `event_type = 'signup_flow'` is normal → the engagement tracking pipeline broke, not actual behaviour.  
If both drop together → something real happened.

---

### 2. Weekly engaged users (reproducing the chart)

```sql
SELECT DATE_TRUNC('week', occurred_at)  AS week,
       COUNT(DISTINCT user_id)           AS engaged_users
FROM tutorial.yammer_events
WHERE event_type = 'engagement'
GROUP BY 1
ORDER BY 1;
```

---

### 3. New user growth check

```sql
SELECT DATE_TRUNC('week', created_at) AS week,
       COUNT(*)                        AS new_users
FROM tutorial.yammer_users
WHERE state = 'active'
GROUP BY 1
ORDER BY 1;
```

**Finding:** Growth was normal. The problem is with existing users not returning — not a new user acquisition issue.

---

### 4. Cohort analysis — segment by signup age

```sql
SELECT DATE_TRUNC('week', e.occurred_at)                          AS week,
       CASE WHEN u.activated_at < '2014-06-01' THEN 'old_users'
            ELSE 'new_users'
       END                                                         AS user_cohort,
       COUNT(DISTINCT e.user_id)                                   AS engaged_users
FROM tutorial.yammer_events  AS e
LEFT JOIN tutorial.yammer_users AS u ON e.user_id = u.user_id
WHERE e.event_type = 'engagement'
GROUP BY 1, 2
ORDER BY 1, 2;
```

**Finding:** Users who signed up more than 10 weeks prior showed a clear engagement drop. Newer users were fine. The problem is retention of long-time users, not new user experience.

---

### 5. Device-type breakdown

```sql
SELECT DATE_TRUNC('week', occurred_at) AS week,
       device,
       COUNT(DISTINCT user_id)          AS engaged_users
FROM tutorial.yammer_events
WHERE event_type = 'engagement'
GROUP BY 1, 2
ORDER BY 1, 2;
```

Useful to first bucket by device category:

```sql
SELECT DATE_TRUNC('week', occurred_at)              AS week,
       CASE WHEN device IN ('iphone', 'samsung galaxy', 'nexus 5',
                            'htc one', 'nokia lumia 635')
                 THEN 'phone'
            WHEN device IN ('ipad', 'kindle fire', 'nexus 7',
                            'nexus 10', 'samsung galaxy tablet')
                 THEN 'tablet'
            ELSE 'computer'
       END                                           AS device_type,
       COUNT(DISTINCT user_id)                       AS engaged_users
FROM tutorial.yammer_events
WHERE event_type = 'engagement'
GROUP BY 1, 2
ORDER BY 1, 2;
```

**Finding:** Phone engagement dropped steeply. Desktop and tablet were normal. The problem is localised to the mobile app.

---

### 6. Email re-engagement check

```sql
SELECT DATE_TRUNC('week', occurred_at) AS week,
       action,
       COUNT(*)                         AS email_count
FROM tutorial.yammer_emails
GROUP BY 1, 2
ORDER BY 1, 2;
```

**Finding:** Emails were being sent and opened at normal rates. But clickthrough rates dropped significantly. Users saw the digest email but were not returning to the product via it.

---

## What the Investigation Found

| Problem | Evidence | Team to investigate |
|---|---|---|
| Mobile app issue | Phone engagement dropped among long-time users | Mobile engineering |
| Email clickthrough failure | Clickthrough rates down despite normal send/open rates | Email / growth engineering |

---

## The Complete Answer

Two separate problems, not one:

1. **Mobile app regression** — something changed in the mobile app that prevented long-time phone users from engaging. The mobile engineering team needs to check what was deployed around July 28.

2. **Email re-engagement channel broken** — weekly digest emails are being sent and opened, but users are not clicking through. This is a second, independent failure suppressing engagement from the re-engagement channel.

The analyst's job ends here. You do not fix either problem. You take these two findings to the product lead and direct them to the two relevant engineering teams.

---

## Analytical Skills This Case Teaches

**1. Rule out data problems before drawing conclusions.**  
A drop in a chart can mean the tracking broke, not that behaviour changed. Always verify instrumentation first.

**2. Never interpret aggregate metrics alone.**  
Total engagement hid completely opposite trends in new vs. existing users. Always segment before concluding.

**3. Follow the re-engagement chain.**  
For products with email notifications, a drop in engagement may be a communication channel problem, not a product problem. Always check what brings users back, not just whether they come back.

**4. Narrow the surface area before escalating.**  
By the end of the investigation, a company-wide concern was narrowed to two specific engineering areas. That is the value of the analysis.
