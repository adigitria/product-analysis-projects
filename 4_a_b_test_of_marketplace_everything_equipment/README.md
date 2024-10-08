# 4. A/B test of marketplace "Everything.Equipment”

### **RESEARCH OBJECTIVE AND CONTEXT**

### **Context:**

The marketplace **"Vse.tekhnika"** has decided to create a separate category for **gaming laptops**, which were previously grouped with PCs and regular laptops. Before rolling out this change across the entire platform, the team wants to validate the idea through an **A/B test**.

### **Objective:**

Conduct an **A/B test** on the **"Vse.tekhnika"** marketplace audience.

### **Hypothesis:**

Demand for **gaming laptops** could be higher, but users struggle to find them among other laptops.

### **Test Metrics:**

- **Conversion rate** will increase by **100%**
- **Average order value** in the category will **remain stable**

## SQL

```jsx

            WITH

            -- Сформируем профиль пользователя

            profiles AS (

                SELECT DISTINCT user_id,
                       MIN(install_date) OVER (PARTITION BY user_id) AS install_date,
                       MAX(registration_flag) OVER (PARTITION BY user_id) AS registration_flag,
                       FIRST_VALUE(region) OVER (PARTITION BY user_id ORDER BY session_start_ts) AS first_region,
                       FIRST_VALUE(device) OVER (PARTITION BY user_id ORDER BY session_start_ts) AS first_device
                FROM sessions_project_history
                WHERE install_date BETWEEN '2020-08-11' AND '2020-09-10'

            ),

            -- Получим данные о покупках

            orders AS (

                  SELECT user_id,
                         COUNT(purchases_number) AS transactions,
                /* Добавьте выражение для расчёта числа транзакций здесь */
                         SUM(price) AS revenue
                /* Добавьте выражение для определения выручки здесь */
                  FROM purchases_project_history
                  WHERE category = 'computer_equipments' /* Задайте условие для выбора категории товаров здесь */
                  GROUP BY user_id

            )

            -- Объединим всё в единый набор данных

            SELECT p.user_id,
                   p.install_date,
                   o.transactions,
                   o.revenue
            FROM profiles p
            LEFT JOIN orders o ON o.user_id = p.user_id
            /* задайте условие для JOIN здесь */
```

```jsx
            WITH

            -- Сформируем профиль пользователя

            profiles AS (

                SELECT DISTINCT user_id,
                       MIN(install_date) OVER (PARTITION BY user_id) AS install_date,
                       MAX(registration_flag) OVER (PARTITION BY user_id) AS registration_flag,
                       FIRST_VALUE(region) OVER (PARTITION BY user_id ORDER BY session_start_ts) AS first_region,
                       FIRST_VALUE(device) OVER (PARTITION BY user_id ORDER BY session_start_ts) AS first_device,
                       FIRST_VALUE(test_name) OVER (PARTITION BY user_id ORDER BY session_start_ts) AS test_name,
                       test_group
                FROM sessions_project_test_part
                WHERE install_date BETWEEN '2020-10-14' AND '2020-10-14'

            )
            SELECT user_id,
                   COUNT(DISTINCT test_group) AS in_groups
                    /* Задайте выражение для расчёта числа уникальных групп здесь */
            FROM profiles
            WHERE test_name = 'gaming_laptops_test'
            GROUP BY user_id
            HAVING COUNT(DISTINCT test_group) > 1
```

```jsx

            WITH

            -- Сформируем профиль пользователя

            profiles AS (

                SELECT DISTINCT user_id,
                       MIN(install_date) OVER (PARTITION BY user_id) AS install_date,
                       MAX(registration_flag) OVER (PARTITION BY user_id) AS registration_flag,
                       FIRST_VALUE(region) OVER (PARTITION BY user_id ORDER BY session_start_ts) AS first_region,
                       FIRST_VALUE(device) OVER (PARTITION BY user_id ORDER BY session_start_ts) AS first_device,
                       FIRST_VALUE(test_name) OVER (PARTITION BY user_id ORDER BY session_start_ts) AS test_name,
                       FIRST_VALUE(test_group) OVER (PARTITION BY user_id ORDER BY session_start_ts) AS test_group
                FROM sessions_project_test
                WHERE install_date BETWEEN '2020-10-14' AND '2020-10-20'

            ),

            -- Получим данные о покупках

            orders AS (

                SELECT user_id,
                         COUNT(purchases_number) AS transactions,
                         SUM(price) AS revenue
                  FROM purchases_project_test
                  WHERE category = 'computer_equipments' 
                  GROUP BY user_id
                  /* Задайте подзапрос для определения числа покупок (trnsactions) и выручки (revenue) здесь */

            )

            -- Объединим всё в единый набор данных

            SELECT p.user_id,
                   p.install_date,
                   p.test_group,
                   o.transactions,
                   o.revenue
            FROM profiles p
            LEFT JOIN orders o ON o.user_id = p.user_id
            WHERE test_name = 'gaming_laptops_test'
            /* Добавьте условие на название теста здесь */
```

## DashBoard

https://public.tableau.com/app/profile/svetlana.bogomaz/viz/ABtest_17260813740980/Rev

## Conclusions and recommendations

 

### **A/B Test Dashboard Summary**

- The **control (15,256 users)** and **test (15,078 users)** groups were evenly distributed, with a total sample size of **30,334 users**, making the test valid.
- **Conversion rate** in the test group (**1.11%**) was **0.59% higher** than in the control group (**0.52%**).
- **Average order value (AOV)** was **580 RUB higher** in the control group due to outliers in the test group, which had both very low and very high purchase amounts.

### **Evaluation of Test Results**

- **Z-test for conversion rate:** p-value = **0.00000** → Statistically significant difference; conversion improved in the test group.
- **T-test for average order value:** p-value = **0.26938** → No significant difference in revenue between groups; we fail to reject the null hypothesis.
- **Mann-Whitney test for average order value:** p-value = **0.01016** → Significant difference in distributions between the two groups.

### **Business Conclusions & Recommendations**

- The test **was not entirely successful**, as the hypothesis about increasing AOV **was not confirmed**, even though conversion **significantly improved**.
- The increase in conversion suggests that users found it easier to locate **gaming laptops**, leading to **more purchases** in the test group.
- The lower AOV in the test group may be due to:
    - The presence of **lower-priced products** in the category.
    - A younger audience (gamers) with **lower purchasing power**.
    - Fewer **upsell opportunities** for accessories and high-end models.

### **Recommendations:**

1. **Reevaluate the product assortment** in the gaming laptop category to ensure a balanced price range.
2. **Introduce a loyalty program** to encourage higher spending on future purchases.
3. **Explore targeted promotions** or bundling strategies to increase AOV (e.g., discounts on gaming accessories with laptop purchases).