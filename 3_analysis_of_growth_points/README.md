# 3. Analysis of Growth points

### **User Journey Analysis**

1. **How many sessions do users typically generate?**
    - Identifying the most common number of sessions per user to understand engagement levels.
2. **During which session does the first purchase usually occur?**
    - Analyzing whether the first purchase happens early in the user journey or requires multiple interactions.
    - Examining differences in the session number of the first purchase across platforms, cities, and acquisition sources.
3. **What is the most typical user path during the first session?**
    - Mapping common actions users take in their initial interaction with the product.
4. **Do typical event sequences vary across platforms, cities, and acquisition sources?**
    - Checking if different user segments engage with the product in distinct ways.
5. **What are the key steps in the user journey from first entry to first purchase?**
    - Outlining the major touchpoints users go through before completing their first transaction.
6. **What bottlenecks exist in this user journey?**
    - Identifying drop-off points and friction areas that may cause users to abandon the journey.
    - Providing recommendations to the product team for optimizing the funnel and improving conversions.
7. **Do user journeys differ across platforms, cities, and acquisition sources?**
    - Analyzing variations in user behavior based on segmentation.
    - Interpreting what these differences indicate about product experience or audience preferences.

---

### **RFM Analysis & ABC-XYZ Analysis**

1. **What are the three largest RFM segments across different platforms, cities, and acquisition sources?**
    - Identifying dominant user groups based on **Recency, Frequency, and Monetary Value**.
2. **Highlight and describe the healthiest and most problematic customer segments.**
    - Evaluating user retention, purchase frequency, and revenue contribution to pinpoint strong and weak segments.
3. **Describe the distribution of partner networks across ABC-XYZ segments.**
    - Classifying partners based on sales volume (**ABC**) and demand variability (**XYZ**).
    - Making recommendations to the product team on which partners should be prioritized for collaboration.
4. **Why shouldn't segmentation parameters be applied to ABC-XYZ analysis results?**
    - Discussing how ABC-XYZ focuses on inventory and demand classification rather than behavioral segmentation.
    - Explaining the risks of misinterpreting ABC-XYZ results when over-segmenting data.

## SQL

```jsx
WITH daily_revenue AS (

    /* Рассчитываем недельную выручку сетей */

    SELECT p.chain,
           date_trunc('week', log_date) as log_week,
           SUM(revenue) AS revenue
    FROM module3_analytics_events e
    LEFT JOIN module3_partners p on p.rest_id = e.rest_id
    WHERE event = 'order'
    GROUP BY p.chain,
             date_trunc('week', log_date)

),
partners AS (

    /* Рассчитываем коэффициенты вариативности */

    SELECT chain,
           SUM(revenue) AS revenue,
           STDDEV(revenue) AS std,
           STDDEV(revenue) / AVG(revenue) AS var_coeff
    FROM daily_revenue
    GROUP BY chain

),
abc_xyz AS (

    /* Рассчитываем доли от общей выручки */

    SELECT chain,
           revenue,
           SUM(revenue) OVER (ORDER BY revenue DESC) AS cumulative_revenue,
           SUM(revenue) OVER () tot_rev,
           SUM(revenue) OVER (ORDER BY revenue DESC) / SUM(revenue) OVER () AS perc,
           std,
           var_coeff
    FROM partners
    ORDER BY revenue DESC

)

/* Распределяем партнёрские рестораны по группам */

SELECT chain,
       CASE 
           WHEN perc <= 0.8 THEN 'A'
           WHEN perc <= 0.95 THEN 'B'
           ELSE 'C'
       END AS abc,
       CASE 
           WHEN var_coeff <= 0.1 THEN 'X'
           WHEN var_coeff <= 0.3 THEN 'Y'
           ELSE 'Z'
       END  AS xyz
FROM abc_xyz
```

## DashBoard

https://public.tableau.com/app/profile/svetlana.bogomaz/viz/Userjourney_17244414208700/RFM-ABC-XYZ-?publish=yes

## Conclusions

### **Improving the User Journey to Purchase**

- Since "Vse.iz.Kafe" customers often place an order during their first session, but the biggest bottleneck in conversion across all segments is **phone number verification**, the **registration confirmation interface** needs improvement, especially for mobile users.
- Additionally, **adding items to the cart** also sees a significant drop in user transitions. Conducting a **UX test** to identify potential difficulties at this step could help optimize the process.

### **Increasing Customer Loyalty**

- The main challenge for the service is **repeat purchases** (especially in segments 111, 112, 211, and 311). Implementing **loyalty programs and cumulative discounts** could help increase order frequency and **raise the average check**, which is currently low across many segments.
- Consider targeted **marketing campaigns** in specific cities, such as **Novosibirsk**, where segment **312** is the largest but has only **5 unique customers**. Expanding the customer base in this segment could drive growth.

### **Revising the Partner Restaurant List**

- The key growth opportunity lies in **stabilizing demand** for restaurants such as **Gastronomic Storm, Gourmet Delight, Breakfasts for Every Taste, and Chocolate Paradise**. Introducing **permanent promotions or loyalty systems** in these restaurants could help ensure steady revenue.
- Consider **discontinuing partnerships** with underperforming restaurants in **Group C**, including:
    - **Sandwich Universe**
    - **Sandwich Traveler**
    - **Venetian Pub**
    - **Breakfasts Every Day**
    - **Confectionery Story**
    - **Mamma Mia**
    - **Incredible Sandwiches**
    - **Pasta & Friends**
    - **Sweet Journey**
    - **Sandwich Parade**
    - **Florentine Restaurant**

Focusing on **high-performing partners** and optimizing the **user experience** will enhance both conversion rates and customer retention.