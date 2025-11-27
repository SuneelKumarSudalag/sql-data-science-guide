# 7 SQL Queries Every Data Scientist Should Know


## Overview
Master these essential SQL patterns to become a production-ready data professional. These queries are used daily in real data science jobs across AWS, Azure, and GCP.


## ⚠️ SQL Dialect Disclaimer

**Note:** This guide primarily uses ANSI SQL standard syntax for maximum portability. However, some examples may include MySQL, PostgreSQL, or SQL Server specific features. When using these queries:

- **MySQL**: Supports column aliases in GROUP BY (e.g., `GROUP BY customer_segment`)
- **SQL Server**: Does NOT support aliases in GROUP BY - use the actual expression instead
- **PostgreSQL**: Does NOT support aliases in GROUP BY
- **Standard SQL**: Always use the full expression in GROUP BY for portability

For production code and teaching materials, always use the full expression in GROUP BY clauses to ensure compatibility across all RDBMS platforms.

---

## 1. Window Functions (ROW_NUMBER, LAG, LEAD)

### What It Is
Window functions perform calculations across a set of rows while maintaining the original row structure.

### Common Window Functions
- `ROW_NUMBER()` - Assigns unique sequential number to rows
- `RANK()` - Assigns rank with ties
- `LAG()` - Access previous row's value
- `LEAD()` - Access next row's value
- `SUM() OVER()` - Running sum without grouping

### Example: Time-Series Analysis
```sql
-- Calculate revenue growth month-over-month
SELECT 
    customer_id,
    month,
    revenue,
    LAG(revenue) OVER (PARTITION BY customer_id ORDER BY month) as prev_month_revenue,
    revenue - LAG(revenue) OVER (PARTITION BY customer_id ORDER BY month) as revenue_change
FROM sales
ORDER BY customer_id, month;
```

### Why It Matters
- 50% more efficient than self-joins
- Easier to read and maintain
- Handles sequential patterns naturally

### Real-World Use Case
**A/B Testing:** Identify user's first visit, last visit, days between visits

---

## 2. CTE (Common Table Expressions)

### What It Is
Temporary named result sets that you reference within a SELECT, INSERT, UPDATE, or DELETE statement.

### Syntax
```sql
WITH cte_name AS (
    SELECT columns FROM table WHERE conditions
)
SELECT * FROM cte_name;
```

### Example: Multi-Step Analysis
```sql
-- Step 1: Get active users
WITH active_users AS (
    SELECT user_id, COUNT(*) as activity_count
    FROM user_events
    WHERE created_at >= CURRENT_DATE - INTERVAL '30 days'
    GROUP BY user_id
    HAVING COUNT(*) > 5
),
-- Step 2: Get their recent purchases
user_purchases AS (
    SELECT 
        au.user_id,
        COUNT(p.purchase_id) as purchase_count,
        SUM(p.amount) as total_spent
    FROM active_users au
    JOIN purchases p ON au.user_id = p.user_id
    WHERE p.created_at >= CURRENT_DATE - INTERVAL '30 days'
    GROUP BY au.user_id
)
-- Step 3: Final analysis
SELECT * FROM user_purchases
ORDER BY total_spent DESC;
```

### Why It Matters
- Makes complex queries readable
- Reuse results multiple times
- Easier to debug (test each CTE independently)
- Better performance than subqueries in many databases

### Real-World Use Case
**Cohort Analysis:** Build user cohorts, apply filters, then analyze behavior

---

## 3. CASE + GROUP BY (Conditional Aggregation)

### What It Is
Use CASE statements within aggregate functions to create conditional segments.

### Example: Cohort Analysis
```sql
-- Segment customers by purchase frequency
SELECT
    CASE
        WHEN purchase_count = 0 THEN 'No Purchases'
        WHEN purchase_count BETWEEN 1 AND 5 THEN 'Light Buyer'
        WHEN purchase_count BETWEEN 6 AND 20 THEN 'Regular Buyer'
        ELSE 'Power Buyer'
    END as customer_segment,
    COUNT(*) as customer_count,
    AVG(total_spent) as avg_spent,
    MIN(last_purchase_date) as earliest_purchase
FROM customer_summary
GROUP BY CASE                    ← ✅ REPEAT THE EXPRESSION
    WHEN purchase_count = 0 THEN 'No Purchases'
    WHEN purchase_count BETWEEN 1 AND 5 THEN 'Light Buyer'
    WHEN purchase_count BETWEEN 6 AND 20 THEN 'Regular Buyer'
    ELSE 'Power Buyer'
END
ORDER BY customer_count DESC;
```

### Example: A/B Testing Analysis
```sql
-- Compare conversion rates by variant
SELECT 
    SUM(CASE WHEN variant = 'A' AND converted = true THEN 1 ELSE 0 END) as variant_a_conversions,
    SUM(CASE WHEN variant = 'A' THEN 1 ELSE 0 END) as variant_a_total,
    SUM(CASE WHEN variant = 'B' AND converted = true THEN 1 ELSE 0 END) as variant_b_conversions,
    SUM(CASE WHEN variant = 'B' THEN 1 ELSE 0 END) as variant_b_total
FROM experiments;
```

### Why It Matters
- Creates business-relevant segments
- Avoids multiple GROUP BY queries
- Enables one-query reporting
- Much faster than doing segmentation in Python

### Real-World Use Case
**RFM Analysis:** Segment users by Recency, Frequency, Monetary value in a single query

---

## 4. UNION vs UNION ALL

### What It Is
Combines results from multiple queries.

### Key Difference
- `UNION` - Removes duplicates (slower)
- `UNION ALL` - Keeps duplicates (faster)

### Example: Combining Multiple Data Sources
```sql
-- UNION: Combine orders from different regions (no duplicates)
SELECT order_id, customer_id, amount, 'US' as region
FROM us_orders
WHERE created_at >= CURRENT_DATE - INTERVAL '30 days'

UNION

SELECT order_id, customer_id, amount, 'EU' as region
FROM eu_orders
WHERE created_at >= CURRENT_DATE - INTERVAL '30 days';
```

### Example: UNION ALL for Performance
```sql
-- UNION ALL: Faster when you know there are no duplicates
SELECT user_id, event_type, 'web' as platform
FROM web_events
WHERE date >= CURRENT_DATE - INTERVAL '1 day'

UNION ALL

SELECT user_id, event_type, 'mobile' as platform
FROM mobile_events
WHERE date >= CURRENT_DATE - INTERVAL '1 day';
```

### Why It Matters
- One tiny word difference (`ALL`) = massive performance difference
- UNION removes duplicates (requires expensive scan)
- UNION ALL is 2-10x faster when you don't need deduplication

### Real-World Use Case
**Multi-Platform Analytics:** Combine web, mobile, and API data into single dataset

---

## 5. JOIN Types (INNER, LEFT, FULL, CROSS)

### What It Is
Methods to combine rows from two or more tables based on a related column.

### INNER JOIN
```sql
-- Only customers with orders
SELECT 
    c.customer_id,
    c.name,
    o.order_id,
    o.amount
FROM customers c
INNER JOIN orders o ON c.customer_id = o.customer_id;
```

### LEFT JOIN
```sql
-- All customers, including those without orders
SELECT 
    c.customer_id,
    c.name,
    COUNT(o.order_id) as order_count,
    COALESCE(SUM(o.amount), 0) as total_spent
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.name;
```

### FULL OUTER JOIN
```sql
-- All rows from both tables
SELECT 
    COALESCE(c.customer_id, o.customer_id) as customer_id,
    c.name,
    o.order_id
FROM customers c
FULL OUTER JOIN orders o ON c.customer_id = o.customer_id
WHERE o.order_id IS NULL OR c.customer_id IS NULL; -- Unmatched records
```

### CROSS JOIN
```sql
-- Cartesian product (use carefully!)
SELECT 
    p.product_id,
    s.store_id
FROM products p
CROSS JOIN stores s;
-- Result: Every product × every store combination
```

### Why It Matters
- Wrong JOIN = wrong answers (silently!)
- Feature engineering at scale depends on correct joins
- Common source of data bugs in production pipelines

### Real-World Use Case
**Feature Engineering:** Join customer data, transaction history, product catalog to create features

---

## 6. Subqueries + Correlated Subqueries

### What It Is
Queries nested within other queries.

### Standard Subquery
```sql
-- Find products with above-average sales
SELECT 
    product_id,
    product_name,
    total_sales
FROM products
WHERE total_sales > (
    SELECT AVG(total_sales)
    FROM products
)
ORDER BY total_sales DESC;
```

### Correlated Subquery
```sql
-- For each customer, show orders above their average order value
SELECT 
    o1.customer_id,
    o1.order_id,
    o1.amount
FROM orders o1
WHERE o1.amount > (
    SELECT AVG(o2.amount)
    FROM orders o2
    WHERE o2.customer_id = o1.customer_id
)
ORDER BY o1.customer_id;
```

### IN Subquery
```sql
-- Find customers who purchased in last 90 days
SELECT 
    customer_id,
    name,
    email
FROM customers
WHERE customer_id IN (
    SELECT DISTINCT customer_id
    FROM orders
    WHERE created_at >= CURRENT_DATE - INTERVAL '90 days'
);
```

### EXISTS Subquery
```sql
-- Find products with at least one review
SELECT 
    p.product_id,
    p.name
FROM products p
WHERE EXISTS (
    SELECT 1
    FROM reviews r
    WHERE r.product_id = p.product_id
);
```

### Why It Matters
- Alternative when window functions aren't available
- Useful for existence checks
- Better readability for some queries
- Performance varies by database engine

### Real-World Use Case
**Data Quality Checks:** Verify data consistency across tables

---

## 7. EXPLAIN PLAN (Query Optimization)

### What It Is
Shows how the database will execute your query (execution plan).

### Syntax
```sql
-- Most databases
EXPLAIN SELECT * FROM orders WHERE created_at >= CURRENT_DATE;

-- SQL Server
SET STATISTICS IO ON;
SELECT * FROM orders WHERE created_at >= CURRENT_DATE;
SET STATISTICS IO OFF;

-- PostgreSQL
EXPLAIN ANALYZE SELECT * FROM orders WHERE created_at >= CURRENT_DATE;
```

### What to Look For
- **Sequential Scan vs Index Scan** - Index scan is faster
- **Join Methods** - Nested Loop, Hash Join, Merge Join
- **Cost** - Lower is better
- **Row Estimates** - Should match actual results

### Example: Before Optimization
```sql
-- SLOW: No index on created_at
SELECT * FROM orders WHERE created_at >= CURRENT_DATE;
-- Result: Full Table Scan (scans 10M rows)
```

### Example: After Optimization
```sql
-- Create index
CREATE INDEX idx_orders_created_at ON orders(created_at);

-- FAST: Uses index
SELECT * FROM orders WHERE created_at >= CURRENT_DATE;
-- Result: Index Scan (scans 100K relevant rows)
```

### Why It Matters
- Difference between 2 seconds and 2 minutes
- Identifies missing indexes
- Shows database's query strategy
- Essential for production queries

### Real-World Use Case
**Production Support:** Identify and fix slow queries before they impact dashboards/reports

---

## Practice Exercises

### Exercise 1: Window Functions
Write a query to find each customer's ranking by total spending.

### Exercise 2: CTE
Create a multi-step analysis that finds the top 10 products, then shows their reviews.

### Exercise 3: CASE + GROUP BY
Segment orders by size (small, medium, large) and calculate average processing time for each.

### Exercise 4: UNION vs UNION ALL
Combine user events from three different tables efficiently.

### Exercise 5: JOIN Types
Create a customer summary that shows all customers, their orders, and products purchased.

### Exercise 6: Subqueries
Find customers whose spending is above the median for their country.

### Exercise 7: EXPLAIN PLAN
Take a slow query and optimize it using indexes.

---

## Cloud Platform Specifics

### Azure SQL / SQL Server
- Window functions: Full support (SQL Server 2012+)
- CTEs: Full support
- Subqueries: Full support
- **Tip:** Use `SET STATISTICS IO` for query analysis

### AWS Athena / Redshift
- Window functions: Full support
- CTEs: Full support
- **Tip:** CTEs are often faster than subqueries
- **Warning:** `UNION` requires DISTINCT columns

### Google BigQuery
- Window functions: Full support
- CTEs: Full support
- **Tip:** Use `UNNEST()` and `STRUCT` for complex data
- **Performance:** Partition tables by date for massive datasets

---

## SQL Dialect Portability Guide

This guide uses ANSI SQL standard syntax wherever possible. Below are key differences when migrating between databases:

### GROUP BY Clause Differences

| Syntax | MySQL | SQL Server | PostgreSQL | Standard SQL |
|--------|-------|------------|------------|--------------|
| `GROUP BY alias_name` | ✅ Works | ❌ Error | ❌ Error | ❌ Not allowed |
| `GROUP BY expression` | ✅ Works | ✅ Works | ✅ Works | ✅ Standard |

**Always use:** `GROUP BY expression` (not alias)

### Example: Correct Portability Pattern



## Key Takeaways

1. **Master window functions** - They replace 80% of self-joins
2. **CTEs make code readable** - Readability = maintainability
3. **CASE + GROUP BY is powerful** - One query instead of multiple
4. **Choose UNION ALL when possible** - Usually 2-10x faster
5. **Understand JOIN types** - Wrong JOIN = wrong data
6. **Know when to use subqueries** - Sometimes simpler than alternatives
7. **Always use EXPLAIN PLAN** - Never assume your query is efficient

---

## Resources

- [SQL Window Functions Guide](https://www.sqlshack.com/en/window-functions-in-sql-server/)
- [CTE Best Practices](https://www.brentozar.com/archive/2016/04/lots-of-common-table-expressions-are-performance-killers/)
- [Query Optimization Techniques](https://use-the-index-luke.com/)
- [Azure SQL Performance Tuning](https://learn.microsoft.com/en-us/azure/azure-sql/database/performance-guidance)
- [AWS Athena Best Practices](https://docs.aws.amazon.com/athena/latest/ug/best-practices.html)

