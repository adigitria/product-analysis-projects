# 1. Cohort analysis

**QUESTIONS:**

**How has the product's DAU level changed over the observation period? Were there any trends, outliers, or anomalies?**

- Analyzing daily active users (DAU) trends over time, identifying any seasonal patterns, spikes, or unusual drops.

**How is the new audience distributed across acquisition sources? Can we identify leading and lagging sources?**

- Evaluating the share of new users from each acquisition channel and determining the most and least effective sources.

**Are there differences in conversion rates to purchases among users from different acquisition sources?**

- Comparing the purchase conversion rates across acquisition channels to see which source brings the most engaged buyers.

**What share of active users does each acquisition source contribute? Does this align with the distribution of new users from the second chart?**

- Assessing the retention and engagement levels of users from different sources and checking whether new user trends translate into long-term activity.

**Are there differences in the time spent watching content among users from different acquisition sources?**

- Analyzing watch time variations by acquisition channel to understand engagement patterns.

**Which movies are in the top 3 by number of views? Which movies are in the top 3 by rating?**

- Identifying the most-watched films and the highest-rated ones, highlighting potential differences in popularity versus perceived quality.

## SQL

```sql
WITH first_values as (

    -- Начальные параметры пользователей

    SELECT DISTINCT user_id,
           FIRST_VALUE(c.city_name) OVER (PARTITION BY user_id ORDER BY datetime) AS city_name,
           FIRST_VALUE(c.city_id) OVER (PARTITION BY user_id ORDER BY datetime) AS city_id,
           FIRST_VALUE(a.device_type) OVER (PARTITION BY user_id ORDER BY datetime) AS device,
           FIRST_VALUE(a.age) OVER (PARTITION BY user_id ORDER BY datetime) AS age,
           FIRST_VALUE(a.source) OVER (PARTITION BY user_id ORDER BY datetime) AS source,
    FIRST_VALUE(a.first_date) OVER (PARTITION BY user_id ORDER BY datetime) AS first_date
    /* Добавьте дополнительные измерения для группировки здесь */
    FROM analytics_events AS a
    LEFT JOIN cities c ON a.city_id = c.city_id
    WHERE first_date BETWEEN '2021-05-01' AND '2021-06-25'
    /* Задайте границы интервала привлечения пользователей здесь */
          AND event = 'authorization'
          AND user_id IS NOT NULL

),
new AS (

    -- Новые пользователи по дням для расчёта CAC

    SELECT first_date,
           source,
           COUNT(DISTINCT user_id) AS new_dau
    FROM first_values
    GROUP BY first_date,
             source

),
cac AS (

    -- Расчёт CAC

    SELECT b.source,
           b.date,
           SUM(b.budget) AS budget,
           SUM(n.new_dau) AS new_dau,
           SUM(b.budget) / SUM(n.new_dau) AS cac
    FROM advertisement_budgets AS b
    LEFT JOIN new AS n ON n.source = b.source
                       AND n.first_date = b.date
    WHERE date BETWEEN '2021-05-01' AND '2021-06-25'
    /* Задайте границы интервала привлечения пользователей здесь */
    GROUP BY b.source, date

),
profiles AS (

    -- Формирование профилей

    SELECT f.user_id,
           f.first_date,
           f.city_name,
           f.city_id,
           f.device, 
           f.age,
           f.source,
    /* Добавьте дополнительные измерения для группировки здесь */
           c.cac
    FROM first_values AS f
    LEFT JOIN cac AS c ON c.source = f.source
                       AND c.date = f.first_date

),
cohorts AS (

    -- Формирование когорт

    SELECT first_date,
           city_name,
           city_id,
           device, 
           age,
           source,
           /* Добавьте дополнительные измерения для группировки здесь */
           COUNT(DISTINCT user_id) AS cohort_size,
           SUM(cac) AS ad_cost
    FROM profiles
    GROUP BY first_date,
             city_name,
             city_id,
             device, 
             age,
             source
             /* Добавьте дополнительные измерения для группировки здесь */

),
/* Расчёт выручки */
orders AS (

    -- Подневная выручка

    SELECT first_date,
           city_id,
           device_type as device, 
           age,
           source,
    /* Добавьте дополнительные измерения для группировки здесь */
           log_date -  first_date AS lifetime,
           MAX(revenue * commission - delivery) /* Добавьте формулу для расчёта выручки здесь */ AS rev
    FROM analytics_events
    WHERE event = 'order'
          AND log_date BETWEEN '2021-06-25' AND TO_DATE('2021-06-25', 'YYYY-MM-DD') + INTERVAL '7 day'
    /* Задайте границы интервала совершения покупок здесь */
    GROUP BY first_date,
             city_id,
              device_type, 
              age,
              source,
             /* Добавьте дополнительные измерения для группировки здесь */
             log_date -  first_date

),
rev AS (

    -- Выручка по когортам

    SELECT c.first_date,
           c.city_name,
           c.city_id,
           c.device, 
           c.age,
           c.source,
           /* Добавьте дополнительные измерения для группировки здесь */
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
                       /* Добавьте измерения для JOIN здесь */

)
/* Финальный результат */
SELECT first_date,
       city_name,
       device, 
       age,
       source,
       /* Добавьте дополнительные измерения для группировки здесь */
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
         /* Добавьте дополнительные измерения для группировки здесь */
```

## DashBoard

[https://public.tableau.com/app/profile/svetlana.bogomaz/viz/FirstDashboard_17200388087300/Dashboard1?publish=yes](https://public.tableau.com/app/profile/svetlana.bogomaz/viz/FirstDashboard_17200388087300/Dashboard1?publish=yes)