# Case 3 — Validating A/B Test Results

**Platform:** Mode Analytics (now part of ThoughtSpot) | mode.com/sql-tutorial  
**Level:** SQL Analytics Training  
**Dataset:** Yammer  
**Tables used:** `tutorial.yammer_experiments`, `tutorial.yammer_events`

---

## The Business Problem

Yammer ran an A/B test on a new messaging notification feature. Results show the treatment group sent significantly more messages. The product team wants to ship immediately.

Your job: validate whether the test was run correctly before anyone acts on the result.

**A positive test result is not proof. It is a claim that requires verification.**

---

## Schema Reference

**tutorial.yammer_experiments**

| Column | Type | Description |
|---|---|---|
| user_id | integer | User in the experiment |
| occurred_at | timestamp | When the user was assigned |
| experiment | varchar | Name of the experiment |
| experiment_group | varchar | 'test_group' or 'control_group' |
| location | varchar | Country |
| device | varchar | Device type |

Outcome measured from `tutorial.yammer_events`: `event_name = 'send_message'`

---

## The Correct Methodology: Hypothesis Before Data

The most important instruction in this entire case:

**Form your hypotheses about what could be wrong before looking at the data.**

If you look at the data first and then explain what you see, your brain will always construct a plausible story. This is called post-hoc rationalisation. It produces confident-sounding analysis that is often wrong.

Correct sequence:
1. Understand what the test was measuring
2. List all the ways the test could be invalid — before seeing results
3. Look at the data and check each hypothesis

---

## A/B Test Validity Checklist — What to Check Before Interpreting Any Result

| Check | Question | What a problem looks like |
|---|---|---|
| Group size balance | Are treatment and control roughly equal in size? | 90/10 split when 50/50 was intended |
| User type mix | Are new and existing users distributed evenly across both groups? | All new users in one group |
| Device mix | Does each group have the same device distribution? | All mobile users in treatment |
| Exposure window | Did all users have the same amount of time to exhibit the measured behaviour? | Users added on day 20 of a 21-day test vs users added on day 1 |
| Assignment randomness | Was assignment truly random? | Systematic pattern in who was assigned where |
| Metric selection | Is the primary metric actually measuring the thing the feature is trying to improve? | Using message count for a feature that changes notification styling |
| Supporting metrics | Do other core metrics move in the same direction? | Primary metric up but logins down |

---

## Key SQL Patterns Used

### 1. Check group size balance

```sql
SELECT experiment_group,
       COUNT(DISTINCT user_id) AS user_count
FROM tutorial.yammer_experiments
WHERE experiment = 'publisher_update_global'
GROUP BY 1;
```

Expect approximately 50/50 split. A major imbalance suggests an assignment bug.

---

### 2. Check primary metric by group

```sql
SELECT e.experiment_group,
       COUNT(DISTINCT e.user_id)                                          AS users,
       COUNT(CASE WHEN ev.event_name = 'send_message' THEN 1 END)        AS messages_sent,
       COUNT(CASE WHEN ev.event_name = 'send_message' THEN 1 END)::float
         / COUNT(DISTINCT e.user_id)                                      AS messages_per_user
FROM tutorial.yammer_experiments  AS e
LEFT JOIN tutorial.yammer_events  AS ev
    ON e.user_id     = ev.user_id
   AND ev.event_type = 'engagement'
   AND ev.occurred_at >= e.occurred_at   -- only count events after assignment
WHERE e.experiment = 'publisher_update_global'
GROUP BY 1;
```

---

### 3. Check supporting metrics

```sql
SELECT e.experiment_group,
       COUNT(DISTINCT e.user_id)                                                 AS users,
       COUNT(DISTINCT ev.user_id || DATE_TRUNC('day', ev.occurred_at)::text)
         / COUNT(DISTINCT e.user_id)::float                                      AS avg_days_engaged,
       COUNT(CASE WHEN ev.event_name = 'login' THEN 1 END)::float
         / COUNT(DISTINCT e.user_id)                                             AS avg_logins
FROM tutorial.yammer_experiments AS e
LEFT JOIN tutorial.yammer_events  AS ev
    ON e.user_id     = ev.user_id
   AND ev.event_type = 'engagement'
   AND ev.occurred_at >= e.occurred_at
WHERE e.experiment = 'publisher_update_global'
GROUP BY 1;
```

If messages_per_user is up in treatment but avg_logins or avg_days_engaged is flat or down, the primary metric result is suspicious.

---

### 4. Separate new users from existing users

```sql
SELECT e.experiment_group,
       CASE WHEN u.activated_at >= '2014-06-01' THEN 'new_user'
            ELSE 'existing_user'
       END                                                                AS user_cohort,
       COUNT(DISTINCT e.user_id)                                          AS users,
       COUNT(CASE WHEN ev.event_name = 'send_message' THEN 1 END)::float
         / COUNT(DISTINCT e.user_id)                                      AS messages_per_user
FROM tutorial.yammer_experiments AS e
LEFT JOIN tutorial.yammer_users   AS u  ON e.user_id = u.user_id
LEFT JOIN tutorial.yammer_events  AS ev
    ON e.user_id     = ev.user_id
   AND ev.event_type = 'engagement'
   AND ev.occurred_at >= e.occurred_at
WHERE e.experiment = 'publisher_update_global'
GROUP BY 1, 2
ORDER BY 1, 2;
```

**Finding:** All new users were placed in the control group — a bug in the assignment code. New users post fewer messages than existing users regardless of feature exposure. This loaded the control group with structurally lower-activity users, making treatment look better than it was.

---

### 5. The exposure window problem

```sql
-- Days each user was in the experiment
SELECT e.experiment_group,
       AVG(EXTRACT(DAY FROM MAX(ev.occurred_at) - MIN(e.occurred_at))) AS avg_exposure_days
FROM tutorial.yammer_experiments AS e
LEFT JOIN tutorial.yammer_events AS ev
    ON e.user_id = ev.user_id
   AND ev.occurred_at >= e.occurred_at
WHERE e.experiment = 'publisher_update_global'
GROUP BY 1;
```

A user added on day 1 of a 21-day test has 21 days to post messages. A user added on day 20 has 1 day. Both counted identically in messages_per_user. The correct fix is either: restrict analysis to users enrolled from day 1, or normalise by dividing messages by days exposed.

---

## What the Investigation Found

| Check | Finding |
|---|---|
| Primary metric (messages sent) | Up in treatment — looks positive |
| Supporting metrics (logins, days engaged) | Also up — consistent with genuine engagement increase |
| Group size balance | Roughly equal — no issue here |
| New vs. existing user distribution | **Bug: all new users placed in control group** |
| Effect size after correcting for bug | Narrower than original result — feature still positive but smaller magnitude |
| Exposure window | Not normalised — users enrolled late had less time to show behaviour |

---

## The Recommendation

**Do not kill the feature.** Results remain positive after correcting for the bug.

**Do not ship immediately either.** Three things must happen first:

1. Fix the experiment assignment bug before running any future tests
2. Re-analyse with new and existing users separated to get the true effect size
3. Validate results across device types and user cohorts before full rollout

---

## Core Concepts This Case Introduces

### Novelty Effect
When a new feature is introduced, existing users may temporarily increase activity simply because something changed — not because the feature is genuinely better. New users have no prior version to compare against, so they do not exhibit this effect. Separating new from existing users lets you detect whether an effect is real or novelty-driven.

### Post-hoc Rationalisation
Looking at data first and then building an explanation for what you see. The result always sounds plausible but is often wrong. The discipline of forming hypotheses before looking at results exists specifically to prevent this.

### Interaction Effects
Places where the tested feature changes behaviour in unexpected ways elsewhere in the product. Even after finding the assignment bug, continue checking: does the feature behave differently on mobile vs desktop? Does it affect creators differently from readers? One problem found does not mean no other problems exist.

---

## Analytical Skills This Case Teaches

**1. Validate the test before interpreting the result.**  
A good-looking number is not evidence. Check assignment randomness, metric selection, and cohort composition before drawing conclusions.

**2. Separate cohorts with structurally different behaviour.**  
New and existing users are not the same population. Mixing them in the same metric without acknowledgment produces technically accurate and analytically meaningless results.

**3. Check supporting metrics, not just the primary metric.**  
If messages are up but logins are down, something is wrong. Primary metric improvement must be consistent with movement in supporting metrics.

**4. One problem found does not mean investigation is complete.**  
Finding the assignment bug is not the end. Always ask: what else could be wrong?
