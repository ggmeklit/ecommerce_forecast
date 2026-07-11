# E-Commerce SQL Analysis Queries and Findings

## 1. Daily Sales

```sql
SELECT
    order_date::date AS sales_date,
    SUM(total_revenue) AS daily_sales
FROM sales_fact
GROUP BY order_date::date
ORDER BY daily_sales DESC;
```

**Finding:** The highest daily sales were recorded on **2017-11-24**, with sales of **152,653**.

---

## 2. Monthly Revenue

```sql
SELECT
    DATE_TRUNC('month', order_date)::date AS month_date,
    SUM(total_revenue) AS monthly_revenue
FROM sales_fact
GROUP BY DATE_TRUNC('month', order_date)::date
ORDER BY monthly_revenue DESC;
```

**Finding:** The highest monthly revenue was recorded in **November 2017**, with revenue of **1,010,271.37**.

---

## 3. Revenue by Category

```sql
SELECT
    product_category,
    SUM(total_revenue) AS category_revenue
FROM sales_fact
GROUP BY product_category
ORDER BY category_revenue DESC;
```

**Finding:** This query ranks product categories by total revenue.

---

## 4. Revenue by Region

```sql
SELECT
    customer_state,
    SUM(total_revenue) AS region_revenue
FROM sales_fact
GROUP BY customer_state
ORDER BY region_revenue DESC;
```

**Finding:** The top revenue regions were **SP**, **RJ**, and **MG**.

- **SP** = São Paulo  
- **RJ** = Rio de Janeiro  
- **MG** = Minas Gerais  

---

## 5. Revenue by City

```sql
SELECT
    customer_city,
    customer_state,
    SUM(total_revenue) AS city_revenue
FROM sales_fact
GROUP BY customer_city, customer_state
ORDER BY city_revenue DESC;
```

**Finding:** The top cities with the highest sales were **São Paulo**, **Rio de Janeiro**, and **Belo Horizonte**.

---

## 6. Average Order Value

```sql
SELECT
    ROUND(AVG(order_total), 2) AS average_order_value
FROM (
    SELECT
        order_id,
        SUM(total_revenue) AS order_total
    FROM sales_fact
    GROUP BY order_id
) order_totals;
```

**Finding:** The average order value was **137.75**.

---

## 7. Average Order Value by Month

```sql
SELECT
    month_date,
    ROUND(AVG(order_total), 2) AS average_order_value
FROM (
    SELECT
        order_id,
        DATE_TRUNC('month', order_date)::date AS month_date,
        SUM(total_revenue) AS order_total
    FROM sales_fact
    GROUP BY order_id, DATE_TRUNC('month', order_date)::date
) order_totals
GROUP BY month_date
ORDER BY month_date;
```

**Finding:** This query shows how average order value changed month by month.

---

## 8. Top-Selling Products by Revenue

```sql
SELECT
    oi.product_id,
    ct.product_category_name_english AS product_category,
    SUM(oi.price) AS product_revenue,
    RANK() OVER (ORDER BY SUM(oi.price) DESC) AS revenue_rank
FROM order_items oi
LEFT JOIN products p
    ON oi.product_id = p.product_id
LEFT JOIN category_translation ct
    ON p.product_category_name = ct.product_category_name
GROUP BY
    oi.product_id,
    ct.product_category_name_english
ORDER BY revenue_rank;
```

**Finding:** By revenue, the top-selling products belonged mainly to **health/beauty** and **computers** categories.

---

## 9. Top-Selling Products by Quantity

```sql
SELECT
    oi.product_id,
    ct.product_category_name_english AS product_category,
    COUNT(*) AS quantity_sold,
    RANK() OVER (ORDER BY COUNT(*) DESC) AS quantity_rank
FROM order_items oi
LEFT JOIN products p
    ON oi.product_id = p.product_id
LEFT JOIN category_translation ct
    ON p.product_category_name = ct.product_category_name
GROUP BY
    oi.product_id,
    ct.product_category_name_english
ORDER BY quantity_rank;
```

**Finding:** By quantity, the top-selling products belonged mainly to **furniture_decor**, **bed_bath_table**, and **garden_tools**.

---

## 10. Customer Repeat Rate

```sql
SELECT
    ROUND(
        100.0 * COUNT(*) FILTER (WHERE order_count > 1) / COUNT(*),
        2
    ) AS repeat_rate_percent
FROM (
    SELECT
        c.customer_unique_id,
        COUNT(DISTINCT o.order_id) AS order_count
    FROM customers c
    JOIN orders o
        ON c.customer_id = o.customer_id
    GROUP BY c.customer_unique_id
) customer_orders;
```

**Finding:** The customer repeat rate was **3.12%**.

---

## 11. Repeat vs One-Time Customers

```sql
SELECT
    CASE
        WHEN order_count > 1 THEN 'Repeat Customer'
        ELSE 'One-Time Customer'
    END AS customer_type,
    COUNT(*) AS customer_count
FROM (
    SELECT
        c.customer_unique_id,
        COUNT(DISTINCT o.order_id) AS order_count
    FROM customers c
    JOIN orders o
        ON c.customer_id = o.customer_id
    GROUP BY c.customer_unique_id
) customer_orders
GROUP BY customer_type;
```

**Finding:** There were far more one-time customers than repeat customers.

| Customer type | Count |
|---|---:|
| One-Time Customer | 93,099 |
| Repeat Customer | 2,997 |

---

## 12. Average Delivery Time by Region

```sql
SELECT
    customer_state,
    ROUND(AVG(delivery_time_days), 2) AS avg_delivery_time_days
FROM sales_fact
WHERE delivery_time_days IS NOT NULL
GROUP BY customer_state
ORDER BY avg_delivery_time_days DESC;
```

**Finding:** The longest average delivery times were recorded for **RR**, **AP**, and **AM**.

- **RR** = Roraima  
- **AP** = Amapá  
- **AM** = Amazonas  


##Code to create sales_fact table

CREATE TABLE sales_fact AS
WITH payment_summary AS (
    SELECT
        order_id,
        STRING_AGG(DISTINCT payment_type, ', ' ORDER BY payment_type) AS payment_types_used,
        COUNT(DISTINCT payment_type) AS payment_type_count,
        SUM(payment_value) AS total_payment_value
    FROM order_payments
    GROUP BY order_id
)

SELECT
    oi.order_id,
    oi.order_item_id,
    o.order_purchase_timestamp AS order_date,
    c.customer_city,
    c.customer_state,
    COALESCE(ct.product_category_name_english, p.product_category_name) AS product_category,
    oi.price AS price,
    1 AS quantity,
    oi.price * 1 AS total_revenue,
    ps.payment_types_used,
    ps.payment_type_count,
    ps.total_payment_value,
    o.delivery_time_days

FROM order_items oi

JOIN orders o
    ON oi.order_id = o.order_id

JOIN customers c
    ON o.customer_id = c.customer_id

LEFT JOIN products p
    ON oi.product_id = p.product_id

LEFT JOIN category_translation ct
    ON p.product_category_name = ct.product_category_name

LEFT JOIN payment_summary ps
    ON oi.order_id = ps.order_id;