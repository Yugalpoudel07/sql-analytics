# SQL Analytics Training — Core Principles & Conclusion

**Platform:** Mode Analytics (now part of ThoughtSpot) | mode.com/sql-tutorial  
**Level:** SQL Analytics Training  
**Covers:** Yammer overview, analytical framework, conclusion

---

## What This Training Is

SQL Analytics Training is not a syntax section. It assumes you already know SELECT, JOINs, aggregations, and window functions. What it teaches is **analytical reasoning with SQL** — how a working analyst thinks through a business problem using data.

Three real-world case studies. One fictional company. All three cases use the same database and the same core discipline.

---

## The Company Context — Yammer

Yammer is a B2B workplace social network (similar to Slack) used by companies for internal communication. Acquired by Microsoft.

The analytics team at Yammer sits inside Engineering. Their job is not to build decorative dashboards. Their job is to drive product and business decisions using data. They do two things:

1. Provide tools and education so other teams can self-serve on data
2. Run ad-hoc analysis to support specific decisions when called on

**Key characteristic of how Yammer analysts think:** every product decision is evaluated against core metrics — engagement, retention, and growth — in addition to whatever product-specific metric is being measured. A feature metric is never looked at in isolation.

---

## The Three Cases — Summary

| Case | Business question | Core technique | Key finding |
|---|---|---|---|
| 1 — Drop in Engagement | What caused the engagement drop? | Cohort segmentation, device slicing, email funnel analysis | Two separate problems: mobile app regression + email clickthrough failure |
| 2 — Search Functionality | Is search worth improving? What specifically? | Session analysis, frequency analysis, click position distribution | Full search ranking algorithm is broken; autocomplete is healthy |
| 3 — A/B Test Validation | Are these test results valid? | Cohort separation, exposure window normalisation, supporting metrics check | Assignment bug placed all new users in control; effect is real but smaller than reported |

---

## The Analytical Discipline — Applied to Every Case

### Step 1: Define what you are measuring and why

Before writing a query, answer: what does this metric actually mean? What behaviour does it reflect?

- "Engaged users" = users who logged at least one engagement event that week
- "Search frequency" = number of search_run events per session per user
- "Messages per user" = total send_message events divided by distinct users in each group

If you cannot define the metric precisely, the number means nothing.

### Step 2: Form hypotheses before looking at data

List every possible explanation. Order by likelihood. Write one sentence on how you would test each. Then look at the data.

If you look at the data first and build your explanation after, you will always find a story. That story is usually wrong. This is post-hoc rationalisation.

### Step 3: Segment before concluding

Aggregate metrics lie. A total always hides variation underneath. Before drawing any conclusion from a total:

- Cut by cohort (new vs. existing users)
- Cut by device (phone vs. desktop vs. tablet)
- Cut by geography (is it one country or global?)
- Cut by user type (creators vs. readers, active vs. recently activated)

**The rule:** never report a total without knowing whether that total is uniform across segments or driven by one specific segment.

### Step 4: Rule out instrumentation failures first

A drop in a chart can mean two completely different things:
- The behaviour actually changed
- The logging pipeline broke

Always check whether the event volume drop is uniform across all event types or isolated to one. If only one event type dropped, the tracker broke. If all events dropped together, the behaviour is real.

### Step 5: Follow the re-engagement chain

For any product with notification or email features, reduced engagement may not be a product problem. It may be a channel problem. Always check:

- Are re-engagement emails being sent?
- Are they being opened?
- Are users clicking through?

Each of those is a separate failure point with a different fix.

---

## The Underlying Principle — Connecting All Three Cases

| Case | What the aggregate hid | What segmentation revealed |
|---|---|---|
| Case 1 | Total engagement drop looked like one problem | Two separate problems in two separate segments (mobile users + email channel) |
| Case 2 | High search frequency looked like engagement | Repeated failed refinement — a failure signal disguised as usage |
| Case 3 | Positive test result looked like proof | Assignment bug had loaded control group with structurally lower-activity users |

**The single principle underneath all three:**

Numbers do not interpret themselves. Every metric, every result, every chart requires you to ask **what it is not showing you** before you conclude anything from what it is showing you.

---

## SQL Techniques Introduced in This Training

| Technique | Case | What it does |
|---|---|---|
| `DATE_TRUNC('week', column)` | Case 1 | Bucket timestamps into calendar weeks |
| Cohort CASE WHEN on activated_at | Case 1 | Segment users by signup age |
| CASE WHEN on device names | Case 1 | Bucket specific device strings into categories |
| `LAG()` window function | Case 2 | Compare each event's timestamp to the previous one |
| Session ID via cumulative SUM() OVER | Case 2 | Assign session numbers using running sum of new-session flags |
| `LIKE 'search_click_result_%'` | Case 2 | Match all 10 click result event names in one filter |
| `COUNT(*) * 100.0 / SUM(COUNT(*)) OVER ()` | Case 2 | Distribution percentage across groups |
| Joining experiments to events on user_id + occurred_at | Case 3 | Only count events that happened after a user was assigned to the test |
| Cohort separation in A/B analysis | Case 3 | Isolate new vs. existing user effects independently |
| Supporting metric check alongside primary metric | Case 3 | Verify primary metric movement is real and not an artefact |

---

## What This Training Prepares You For

These three cases directly mirror real scenarios in AU/NZ analyst roles:

- **Drop in a metric** — every data team gets asked this. The answer is always: segment first, rule out tracking issues, follow the re-engagement chain.
- **Feature evaluation** — product teams ask analysts whether something is working. The answer requires defining what "working" means, measuring frequency and distribution, not just totals.
- **A/B test review** — any company running experiments needs analysts who can validate tests before results are acted on. The most common failure mode is exactly what Case 3 describes: structurally different user mixes in each group.
