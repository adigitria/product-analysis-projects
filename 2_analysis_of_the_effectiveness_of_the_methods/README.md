# 2. Analysis of the Effectiveness of Promotion Methods for "All.of.Cafe"

## Table of Contents
1. [Goal](#goal)
2. [Questions](#questions)
3. [SQL Pipeline](#sql-pipeline)
   - [User Attributes](#user-attributes)
   - [New Users per Day](#new-users-per-day)
   - [CAC Calculation](#cac-calculation)
   - [User Profiles](#user-profiles)
   - [Cohort Formation](#cohort-formation)
   - [Revenue Calculation](#revenue-calculation)
   - [Cohort Revenue](#cohort-revenue)
   - [Final Metrics (LTV)](#final-metrics-ltv)
4. [Dashboard](#dashboard)
5. [Conclusions](#conclusions)
6. [Recommendation](#recommendation)

---

## Goal

To analyze the payback of new product users within the first seven days of their lifecycle.

## Questions

| Question                                                                                                          | Focus                                                                 |
|-------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------|
| Is user acquisition profitable overall? On which day of the lifecycle does payback occur?                        | ROI and breakeven analysis                                             |
| Which advertising channels, cities, age segments, and platforms lead in terms of total cohort size?              | Cohort size by segmentation                                           |
| How is CAC distributed across advertising channels, cities, age segments, and platforms?                         | Acquisition cost comparison                                           |
| What can be said about LTV dynamics across different segments?                                                   | Revenue trend analysis                                                |
| How has CAC changed over time across segments?                                                                   | Time-based CAC trends                                                 |
| Are there differences in payback periods for different segments?                                                 | Payback time per segment                                              |
| Can we identify leading and lagging cohorts in terms of payback?                                                 | Identify best/worst performing cohorts                                |
| Based on your analysis, what recommendations can be made to the client?                                          | Strategic suggestions                                                 |

## SQL Pipeline

### User Attributes
Extract initial user information.

```sql
WITH first_values AS (
    -- Initial user parameters
    SELECT DISTINCT user_id,
           FIRST_VALUE(c.city_name) OVER (PARTITION BY user_id ORDER BY datetime) AS city_name,
           FIRST_VALUE(c.city_id) OVER (PARTITION BY user_id ORDER BY datetime) AS city_id,
           FIRST_VALUE(a.device_type) OVER (PARTITION BY user_id ORDER BY datetime) AS device,
           FIRST_VALUE(a.age) OVER (PARTITION BY user_id ORDER BY datetime) AS age,
           FIRST_VALUE(a.source) OVER (PARTITION BY user_id ORDER BY datetime) AS source,
           FIRST_VALUE(a.first_date) OVER (PARTITION BY user_id ORDER BY datetime) AS first_date
    -- Add more dimensions here if needed
    FROM analytics_events AS a
    LEFT JOIN cities c ON a.city_id = c.city_id
    WHERE first_date BETWEEN '2021-05-01' AND '2021-06-25'
          AND event = 'authorization'
          AND user_id IS NOT NULL
)
```

### New Users per Day
Used for CAC calculation.

```sql
,new AS (
    -- New users per day for CAC
    SELECT first_date,
           source,
           COUNT(DISTINCT user_id) AS new_dau
    FROM first_values
    GROUP BY first_date, source
)
```

### CAC Calculation
Combining budget and user data.

```sql
,cac AS (
    -- CAC calculation
    SELECT b.source,
           b.date,
           SUM(b.budget) AS budget,
           SUM(n.new_dau) AS new_dau,
           SUM(b.budget) / SUM(n.new_dau) AS cac
    FROM advertisement_budgets AS b
    LEFT JOIN new AS n ON n.source = b.source AND n.first_date = b.date
    WHERE date BETWEEN '2021-05-01' AND '2021-06-25'
    GROUP BY b.source, date
)
```

### User Profiles
User profiles enriched with CAC.

```sql
,profiles AS (
    -- Profile enrichment
    SELECT f.user_id,
           f.first_date,
           f.city_name,
           f.city_id,
           f.device, 
           f.age,
           f.source,
           c.cac
    FROM first_values AS f
    LEFT JOIN cac AS c ON c.source = f.source AND c.date = f.first_date
)
```

### Cohort Formation
Group users into cohorts.

```sql
,cohorts AS (
    -- Cohort grouping
    SELECT first_date,
           city_name,
           city_id,
           device, 
           age,
           source,
           COUNT(DISTINCT user_id) AS cohort_size,
           SUM(cac) AS ad_cost
    FROM profiles
    GROUP BY first_date, city_name, city_id, device, age, source
)
```

### Revenue Calculation
Revenue by day since acquisition.

```sql
,orders AS (
    -- Daily revenue per user
    SELECT first_date,
           city_id,
           device_type AS device, 
           age,
           source,
           log_date - first_date AS lifetime,
           MAX(revenue * commission - delivery) AS rev
    FROM analytics_events
    WHERE event = 'order'
          AND log_date BETWEEN '2021-06-25' AND TO_DATE('2021-06-25', 'YYYY-MM-DD') + INTERVAL '7 day'
    GROUP BY first_date, city_id, device_type, age, source, log_date - first_date
)
```

### Cohort Revenue
Join cohorts with revenue.

```sql
,rev AS (
    -- Cohort revenue aggregation
    SELECT c.first_date,
           c.city_name,
           c.city_id,
           c.device, 
           c.age,
           c.source,
           o.lifetime,
           o.rev,
           c.cohort_size,
           c.ad_cost
    FROM cohorts c
    LEFT JOIN orders o ON c.first_date = o.first_date
                       AND c.city_id = o.city_id
                       AND c.device = o.device 
                       AND c.age = o.age
                       AND c.source = o.source
)
```

### Final Metrics (LTV)
Calculate LTV for each day.

```sql
SELECT first_date,
       city_name,
       device, 
       age,
       source,
       MAX(cohort_size) AS cohort_size,
       MAX(ad_cost) AS ad_cost,
       SUM(CASE WHEN lifetime <= 0 THEN rev ELSE 0 END)::FLOAT / MAX(cohort_size)::FLOAT AS ltv_d1,
       SUM(CASE WHEN lifetime <= 1 THEN rev ELSE 0 END)::FLOAT / MAX(cohort_size)::FLOAT AS ltv_d2,
       SUM(CASE WHEN lifetime <= 2 THEN rev ELSE 0 END)::FLOAT / MAX(cohort_size)::FLOAT AS ltv_d3,
       SUM(CASE WHEN lifetime <= 3 THEN rev ELSE 0 END)::FLOAT / MAX(cohort_size)::FLOAT AS ltv_d4,
       SUM(CASE WHEN lifetime <= 4 THEN rev ELSE 0 END)::FLOAT / MAX(cohort_size)::FLOAT AS ltv_d5,
       SUM(CASE WHEN lifetime <= 5 THEN rev ELSE 0 END)::FLOAT / MAX(cohort_size)::FLOAT AS ltv_d6,
       SUM(CASE WHEN lifetime <= 6 THEN rev ELSE 0 END)::FLOAT / MAX(cohort_size)::FLOAT AS ltv_d7
FROM rev
GROUP BY first_date, city_name, device, age, source
```

## Dashboard

[View Tableau Dashboard](https://public.tableau.com/app/profile/svetlana.bogomaz/viz/allFromCaffee/Dashboard1?publish=yes)

## Conclusions

- Total marketing investments on the seventh day are close to breakeven (ROI = 0.98).
- Acquisition is profitable only in Saransk with ROI ≈ 1.2.
- Other regions have low ROI due to high CAC and weak retention.
- Only **Source_B** is profitable among acquisition sources.
- Profitable age groups: **51–65**, **65+**, and **26–35**.
- **18–25** is the largest cohort but has low ROI, likely due to limited purchasing power.

## Recommendation

- Investigate **Source_C** due to emerging positive trends and lower CAC.
- Rework campaigns targeting **18–25** group due to upward LTV trend.
- Focus on cities like **Sochi** and **Novosibirsk**, which show improving ROI trends.
