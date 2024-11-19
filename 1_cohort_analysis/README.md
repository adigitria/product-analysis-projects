# 1. Cohort analysis

### Table of Contents
1. [Questions](#questions)
2. [SQL LTV Analysis Pipeline](#sql-ltv-analysis-pipeline)
   1. [User First Values](#user-first-values)
   2. [New Users](#new-users)
   3. [CAC Calculation](#cac-calculation)
   4. [User Profiles](#user-profiles)
   5. [Cohort Creation](#cohort-creation)
   6. [Orders and Revenue](#orders-and-revenue)
   7. [Cohort Revenue](#cohort-revenue)
   8. [Final LTV Calculation](#final-ltv-calculation)
   9. [Final version](#final-version)
3. [DashBoard](#dashboard)

## Questions

| Question                                                                                      | Focus                                                                 |
|-----------------------------------------------------------------------------------------------|-----------------------------------------------------------------------|
| How has DAU changed over time? Any trends or anomalies?                                       | DAU trends, seasonality, spikes, or drops                            |
| How is the new audience distributed across sources?                                           | Share of new users by acquisition channel                            |
| Are there differences in purchase conversion rates by source?                                | Conversion comparison by acquisition source                          |
| What share of active users does each source contribute? Does it align with new user trends?  | Retention and engagement by source vs. initial user distribution     |
| Are there differences in watch time by source?                                                | Watch time variation across channels                                 |
| Top 3 movies by views and rating?                                                             | Most-watched vs. highest-rated movies                                |

## SQL LTV Analysis Pipeline

### User First Values
Get the initial attributes of users: city, device, age, source, etc.

```sql
WITH first_values AS (

    -- Initial user parameters

    SELECT DISTINCT user_id,
                    FIRST_VALUE(c.city_name) OVER (PARTITION BY user_id ORDER BY datetime) AS city_name, FIRST_VALUE(c.city_id) OVER (PARTITION BY user_id ORDER BY datetime) AS city_id, FIRST_VALUE(a.device_type) OVER (PARTITION BY user_id ORDER BY datetime) AS device, FIRST_VALUE(a.age) OVER (PARTITION BY user_id ORDER BY datetime) AS age, FIRST_VALUE(a.source) OVER (PARTITION BY user_id ORDER BY datetime) AS source, FIRST_VALUE(a.first_date) OVER (PARTITION BY user_id ORDER BY datetime) AS first_date
    -- Add extra dimensions for grouping here
    FROM analytics_events AS a
             LEFT JOIN cities c ON a.city_id = c.city_id
    WHERE first_date BETWEEN '2021-05-01' AND '2021-06-25'
      -- Set the user acquisition date range here
      AND event = 'authorization'
      AND user_id IS NOT NULL)
```

---

### New Users
Count new users per day and source for CAC calculation.

```sql
,new AS (
    -- New users per day for CAC calculation
    SELECT first_date,
           source,
           COUNT(DISTINCT user_id) AS new_dau
    FROM first_values
    GROUP BY first_date, source
)
```

---

### CAC Calculation
Join with budget data to calculate CAC.

```sql
,cac AS (
    -- CAC calculation
    SELECT b.source,
           b.date,
           SUM(b.budget) AS budget,
           SUM(n.new_dau) AS new_dau,
           SUM(b.budget) / SUM(n.new_dau) AS cac
    FROM advertisement_budgets AS b
    LEFT JOIN new AS n ON n.source = b.source
                       AND n.first_date = b.date
    WHERE date BETWEEN '2021-05-01' AND '2021-06-25'
          -- Set the user acquisition date range here
    GROUP BY b.source, date
)
```

---

### User Profiles
Enrich user profiles with CAC values.

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
           -- Add extra dimensions for grouping here
           c.cac
    FROM first_values AS f
    LEFT JOIN cac AS c ON c.source = f.source
                       AND c.date = f.first_date
)
```

---

### Cohort Creation
Group users into cohorts based on acquisition parameters.

```sql
,cohorts AS (
    -- Cohort formation
    SELECT first_date,
           city_name,
           city_id,
           device,
           age,
           source,
           -- Add extra dimensions for grouping here
           COUNT(DISTINCT user_id) AS cohort_size,
           SUM(cac) AS ad_cost
    FROM profiles
    GROUP BY first_date,
             city_name,
             city_id,
             device,
             age,
             source
             -- Add extra dimensions for grouping here
)
```

---

### Orders and Revenue
Get revenue per user by day since acquisition.

```sql
,orders AS (
    -- Daily revenue
    SELECT first_date,
           city_id,
           device_type AS device,
           age,
           source,
           -- Add extra dimensions for grouping here
           log_date - first_date AS lifetime,
           MAX(revenue * commission - delivery) AS rev
           -- Add custom revenue formula here
    FROM analytics_events
    WHERE event = 'order'
          AND log_date BETWEEN '2021-06-25' AND TO_DATE('2021-06-25', 'YYYY-MM-DD') + INTERVAL '7 day'
          -- Set the purchase date range here
    GROUP BY first_date,
             city_id,
             device_type,
             age,
             source,
             -- Add extra dimensions for grouping here
             log_date - first_date
)
```

---

### Cohort Revenue
Combine cohort data with revenue data by day since acquisition.

```sql
,rev AS (
    -- Revenue per cohort
    SELECT c.first_date,
           c.city_name,
           c.city_id,
           c.device,
           c.age,
           c.source,
           -- Add extra dimensions for grouping here
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
                       -- Add JOIN dimensions here
)
```

---

### Final LTV Calculation
Compute Day 1 to Day 7 LTV for each cohort.

```sql
SELECT first_date,
       city_name,
       device,
       age,
       source,
       -- Add extra dimensions for grouping here
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
GROUP BY first_date,
         city_name,
         device,
         age,
         source
         -- Add extra dimensions for grouping here
```

### Final version
```sql
WITH first_values as (
    SELECT DISTINCT user_id,
           FIRST_VALUE(c.city_name) OVER (PARTITION BY user_id ORDER BY datetime) AS city_name,
           FIRST_VALUE(c.city_id) OVER (PARTITION BY user_id ORDER BY datetime) AS city_id,
           FIRST_VALUE(a.device_type) OVER (PARTITION BY user_id ORDER BY datetime) AS device,
           FIRST_VALUE(a.age) OVER (PARTITION BY user_id ORDER BY datetime) AS age,
           FIRST_VALUE(a.source) OVER (PARTITION BY user_id ORDER BY datetime) AS source,
    FIRST_VALUE(a.first_date) OVER (PARTITION BY user_id ORDER BY datetime) AS first_date
    FROM analytics_events AS a
    LEFT JOIN cities c ON a.city_id = c.city_id
    WHERE first_date BETWEEN '2021-05-01' AND '2021-06-25'
          AND event = 'authorization'
          AND user_id IS NOT NULL

),
new AS (
    SELECT first_date,
           source,
           COUNT(DISTINCT user_id) AS new_dau
    FROM first_values
    GROUP BY first_date,
             source

),
cac AS (
    SELECT b.source,
           b.date,
           SUM(b.budget) AS budget,
           SUM(n.new_dau) AS new_dau,
           SUM(b.budget) / SUM(n.new_dau) AS cac
    FROM advertisement_budgets AS b
    LEFT JOIN new AS n ON n.source = b.source
                       AND n.first_date = b.date
    WHERE date BETWEEN '2021-05-01' AND '2021-06-25'
    GROUP BY b.source, date

),
profiles AS (
    SELECT f.user_id,
           f.first_date,
           f.city_name,
           f.city_id,
           f.device, 
           f.age,
           f.source,
           c.cac
    FROM first_values AS f
    LEFT JOIN cac AS c ON c.source = f.source
                       AND c.date = f.first_date

),
cohorts AS (
    SELECT first_date,
           city_name,
           city_id,
           device, 
           age,
           source,
           COUNT(DISTINCT user_id) AS cohort_size,
           SUM(cac) AS ad_cost
    FROM profiles
    GROUP BY first_date,
             city_name,
             city_id,
             device, 
             age,
             source
),
orders AS (
    SELECT first_date,
           city_id,
           device_type as device, 
           age,
           source,
           log_date -  first_date AS lifetime,
           MAX(revenue * commission - delivery) AS rev
    FROM analytics_events
    WHERE event = 'order'
          AND log_date BETWEEN '2021-06-25' AND TO_DATE('2021-06-25', 'YYYY-MM-DD') + INTERVAL '7 day'
    GROUP BY first_date,
             city_id,
              device_type, 
              age,
              source,
             log_date -  first_date

),
rev AS (
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

SELECT first_date,
       city_name,
       device, 
       age,
       source,
       MAX(cohort_size) AS cohort_size,
       MAX(ad_cost) AS ad_cost,
       SUM(CASE WHEN lifetime <= 0 THEN rev ELSE 0 END) :: FLOAT /MAX(cohort_size) :: FLOAT AS ltv_d1,
       SUM(CASE WHEN lifetime <= 1 THEN rev ELSE 0 END) :: FLOAT /MAX(cohort_size) :: FLOAT AS ltv_d2,
       SUM(CASE WHEN lifetime <= 2 THEN rev ELSE 0 END) :: FLOAT /MAX(cohort_size) :: FLOAT AS ltv_d3,
       SUM(CASE WHEN lifetime <= 3 THEN rev ELSE 0 END) :: FLOAT /MAX(cohort_size) :: FLOAT AS ltv_d4,
       SUM(CASE WHEN lifetime <= 4 THEN rev ELSE 0 END) :: FLOAT /MAX(cohort_size) :: FLOAT AS ltv_d5,
       SUM(CASE WHEN lifetime <= 5 THEN rev ELSE 0 END) :: FLOAT /MAX(cohort_size) :: FLOAT AS ltv_d6,
       SUM(CASE WHEN lifetime <= 6 THEN rev ELSE 0 END) :: FLOAT /MAX(cohort_size) :: FLOAT AS ltv_d7
FROM rev
GROUP BY first_date,
         city_name,
         device, 
       age,
       source
```

## DashBoard

[https://public.tableau.com/app/profile/svetlana.bogomaz/viz/FirstDashboard_17200388087300/Dashboard1?publish=yes](https://public.tableau.com/app/profile/svetlana.bogomaz/viz/FirstDashboard_17200388087300/Dashboard1?publish=yes)