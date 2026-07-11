# Sales Fact Table Documentation

## Sales Fact Overview

The `sales_fact` table summarizes e-commerce sales at the **order-item level**.

Each row represents **one item within one order**.  
This means the table follows the same grain as `order_items`.

The final row count was validated against `order_items`:

| Table | Row count |
|---|---:|
| `order_items` | 112,650 |
| `sales_fact` | 112,650 |

Because the row counts match, the joins did not remove or multiply item rows.

---

## Sales Fact Column Summary

| # | Column name | Type | Where it came from / meaning |
|---:|---|---|---|
| 1 | `order_id` | `VARCHAR` | From `order_items.order_id`; identifies the order. |
| 2 | `order_item_id` | `INTEGER` | From `order_items.order_item_id`; identifies the item line within the order. |
| 3 | `order_date` | `TIMESTAMP` | From `orders.order_purchase_timestamp`; date and time when the order was placed. |
| 4 | `customer_city` | `VARCHAR` | From `customers.customer_city`; customer city. |
| 5 | `customer_state` | `VARCHAR` | From `customers.customer_state`; customer state. |
| 6 | `product_category` | `VARCHAR/TEXT` | From `category_translation.product_category_name_english`; if unavailable, uses `products.product_category_name`. Missing values were later labeled as `Unknown`. |
| 7 | `price` | `NUMERIC(10,2)` | From `order_items.price`; price of the item row. |
| 8 | `quantity` | `INTEGER` | Created as `1`; each order-item row represents one purchased item. |
| 9 | `total_revenue` | `NUMERIC` | Calculated as `price * quantity`; since quantity is 1, this is the item-level revenue. |
| 10 | `payment_types_used` | `TEXT` | From summarized `order_payments.payment_type`; lists the payment type or payment types used for the order. |
| 11 | `payment_type_count` | `BIGINT/INTEGER` | Calculated from `COUNT(DISTINCT payment_type)`; number of distinct payment types recorded for the order. |
| 12 | `total_payment_value` | `NUMERIC` | From summarized `order_payments.payment_value`; total amount paid for the order. |
| 13 | `delivery_time_days` | `NUMERIC(10,2)` | From `orders.delivery_time_days`; calculated as actual delivery time in days using delivery timestamp minus purchase timestamp. |

---

## Important Grain Notes

`total_revenue` is **item-level** and is safe to sum for sales forecasting.

`total_payment_value` is **order-level**, not item-level.  
It may repeat across multiple item rows from the same order, so it should **not be summed directly from the item-level table** unless the data is first grouped correctly.

---

## Null Value Summary and Handling

Initial null check on `sales_fact` showed:

| Column | Initial null count | Handling decision | Justification |
|---|---:|---|---|
| `delivery_time_days` | 2,454 | Kept as `NULL` | Actual delivery time cannot be calculated when the actual delivery timestamp is missing. This column is for delivery analysis, not the main ARIMA/Prophet sales forecast. |
| `product_category` | 1,603 | Replaced with `Unknown` | Rows still contain valid sales and date information, so dropping them would lose useful revenue data. |
| `payment_types_used` | 3 | Imputed | The 3 rows belonged to one order with missing payment records. Payment type was filled using the most frequent payment type. |
| `payment_type_count` | 3 | Imputed as `1` | After assigning the most frequent payment type, the distinct payment type count becomes 1. |
| `total_payment_value` | 3 | Imputed using order item revenue total | Since payment value was missing for one order, the proxy used was the sum of item-level `total_revenue` for that order. |
| All other columns | 0 | No action needed | No missing values were found. |

---

## Final Null Handling Position

After handling product and payment nulls, the only remaining null column is:

| Column | Remaining null handling |
|---|---|
| `delivery_time_days` | Left as `NULL` |

This is acceptable because the main forecasting models, **ARIMA** and **Prophet**, will use:

- `order_date`
- `total_revenue`

They do **not** require `delivery_time_days`.

---

## Forecasting Relevance

For sales forecasting, the main modeling dataset will later be created by aggregating:

`SUM(total_revenue)` by `order_date`

The core forecasting table will look conceptually like:

| order_date | daily_sales |
|---|---:|
| 2017-01-01 | 5000.00 |
| 2017-01-02 | 6200.00 |
| 2017-01-03 | 5800.00 |

`delivery_time_days` can still be used later for business analysis, such as delivery performance, but it should not block the sales forecasting workflow.
