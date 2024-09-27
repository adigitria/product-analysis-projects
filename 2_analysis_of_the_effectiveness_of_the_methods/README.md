# 2. Analysis of the effectiveness of the methods of promotion of "All.of.Cafe”

**GOAL:**

To analyze the payback of new product users within the first seven days of their lifecycle.

**QUESTIONS:**

• Is user acquisition profitable overall? On which day of the lifecycle does payback occur?

• Which advertising channels, cities, age segments, and platforms lead in terms of total cohort size?

• How is CAC distributed across advertising channels, cities, age segments, and platforms?

• What can be said about the dynamics of LTV across advertising channels, cities, age segments, and platforms?

• How has CAC changed over time across advertising channels, cities, age segments, and platforms?

• Are there differences in payback periods for different advertising channels, cities, age segments, and platforms?

• How does the payback dynamic vary across advertising channels, cities, age segments, and platforms?

• Can we identify leading and lagging cohorts in terms of payback? Describe them, formulate hypotheses about the reasons for abnormally high or low payback, and analyze which cohorts achieve unit economics breakeven and which do not.

• Based on your analysis, provide recommendations to the client. For example, in which city should an advertising campaign be launched, and what user profile should be targeted in the ad storyline?

## SQL

```jsx
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

[https://public.tableau.com/app/profile/svetlana.bogomaz/viz/allFromCaffee/Dashboard1?publish=yes](https://public.tableau.com/app/profile/svetlana.bogomaz/viz/allFromCaffee/Dashboard1?publish=yes)

## Conclusions

- Total marketing investments on the seventh day are close to the breakeven level (ROI = 0.98), but this is not enough to consider the advertising effective. ROI varies across different periods and remains below 1 for most of the time.
- User acquisition is profitable only in Saransk, where all key metrics show the highest and most stable values. The overall ROI for this city is around 1.2.
- The effectiveness in other regions is quite low due to the high acquisition cost and low retention conversion.
- Among acquisition sources, only **Source_B** reaches profitability.
- In terms of age groups, 3 out of 5 cohorts are profitable: **51–65** (the highest ROI), **65+**, and **26–35**.
- Despite being the largest cohort, the **18–25** age group does not appear promising due to its low ROI, which may be related to the audience’s lower purchasing power.

## Recommendation

- Conduct additional research on **Source_C**, as a positive ROI trend has been observed since May 30. Additionally, the cost of acquiring a single customer is significantly lower (0.8) compared to the profitable **Source_B** (1.1). Moreover, the profit per acquired customer by day 7 is higher than for all other sources. For further insights, analyze which combinations of **City, Age, and Platform cohorts** yield the best results. This may help adjust the marketing campaign to better target the audience and improve **Source_C’s** profitability.
- Consider modifying marketing campaigns for the **18–25 age group**, as it is the largest age cohort. There is a positive **LTV trend** by day 7 and an additional upward trend starting in mid-June.
- Among cities, **Sochi and Novosibirsk** stand out due to a positive ROI trend by day 7, as well as since mid-June.