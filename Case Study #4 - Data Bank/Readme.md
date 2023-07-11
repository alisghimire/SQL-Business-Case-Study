# SQL Case Study #4 - Data Bank

## Problem Statement

Data Bank runs just like any other digital bank - but it isn’t only for banking activities, they also have the world’s most secure distributed **data storage platform**!

The management team at Data Bank want to increase their total customer base - but also need some help tracking just how much data storage their customers will need.

This case study is all about calculating metrics, growth and helping the business analyse their data in a smart way to better forecast and plan for their future developments!

<h2>Entity Relationship Diagram</h2>

<img width="631" alt="130343339-8c9ff915-c88c-4942-9175-9999da78542c" src="https://github.com/aliscfc/SQL-Case-Study-4/assets/109817585/091aa587-9412-4719-9dac-e38e9185138f">

## **A. Customer Nodes Exploration**

1. **How many unique nodes are there on the Data Bank system?**

```sql
SELECT 
	COUNT(DISTINCT node_id) AS unique_nodes_count
FROM data_bank.customer_nodes;
```

---

2. **What is the number of nodes per region?**

```sql
SELECT
	r.region_name,
	COUNT(DISTINCT cn.node_id) AS nodes_count
FROM data_bank.regions r
INNER JOIN data_bank.customer_nodes cn
ON r.region_id = cn.region_id

GROUP BY r.region_name;
```

---

3. **How many customers are allocated to each region?**

```sql
SELECT
	region_id,
	COUNT(customer_id) AS total_customers
FROM data_bank.customer_nodes

GROUP BY region_id
ORDER BY region_id;

```

---

4. **How many days on average are customers reallocated to a different node?**

```sql
WITH node_days AS(
	SELECT
		customer_id,
		node_id,
		end_date - start_date AS days_in_node
	FROM data_bank.customer_nodes
	WHERE end_date != '9999-12-31'
	GROUP BY customer_id, node_id, start_date, end_date
),

total_node_days AS(
	SELECT
		customer_id,
		node_id,
		SUM(days_in_node) AS total_days_in_node
	FROM node_days
	GROUP BY customer_id, node_id
)

SELECT
	ROUND(AVG(total_days_in_node)) AS average_reallocated_days
FROM total_node_days;
```
<hr>

5. **What is the median, 80th and 95th percentile for this same reallocation days metric for each region?**

**95th percentile**

```sql
WITH reallocation_days_cte AS
  (SELECT *,
          (datediff(end_date, start_date)) AS reallocation_days
   FROM customer_nodes
   INNER JOIN regions USING (region_id)
   WHERE end_date!='9999-12-31'),
     percentile_cte AS
  (SELECT *,
          percent_rank() over(PARTITION BY region_id
                              ORDER BY reallocation_days)*100 AS p
   FROM reallocation_days_cte)
SELECT region_id,
       region_name,
       reallocation_days
FROM percentile_cte
WHERE p >95
GROUP BY region_id;
```

**80th percentile**

```sql
WITH reallocation_days_cte AS
  (SELECT *,
          (datediff(end_date, start_date)) AS reallocation_days
   FROM customer_nodes
   INNER JOIN regions USING (region_id)
   WHERE end_date!='9999-12-31'),
     percentile_cte AS
  (SELECT *,
          percent_rank() over(PARTITION BY region_id
                              ORDER BY reallocation_days)*100 AS p
   FROM reallocation_days_cte)
SELECT region_id,
       region_name,
       reallocation_days
FROM percentile_cte
WHERE p >80
GROUP BY region_id;
```

**Median**

```sql
WITH reallocation_days_cte AS
  (SELECT *,
          (datediff(end_date, start_date)) AS reallocation_days
   FROM customer_nodes
   INNER JOIN regions USING (region_id)
   WHERE end_date!='9999-12-31'),
     percentile_cte AS
  (SELECT *,
          percent_rank() over(PARTITION BY region_id
                              ORDER BY reallocation_days)*100 AS p
   FROM reallocation_days_cte)
SELECT region_id,
       region_name,
       reallocation_days
FROM percentile_cte
WHERE p >50
GROUP BY region_id;
```

## **B. Customer Transactions**

1. **What is the unique count and total amount for each transaction type?**

```sql
SELECT
	txn_type,
	COUNT(customer_id) AS transaction_count,
	SUM(txn_amount) AS transaction_amount
FROM data_bank.customer_transactions
GROUP BY txn_type;
```
<hr>

2. **What is the average total historical deposit counts and amounts for all customers?**

```sql
WITH deposits AS(
	SELECT
		customer_id,
		COUNT(customer_id) AS txn_count,
		SUM(txn_amount) AS avg_amount,
	FROM data_bank.customer_transactions
	WHERE txt_type = 'deposit'
	GROUP BY customer_id
)

SELECT
	ROUND(AVG(txn_count)),
	ROUND(AVG(avg_amount))
FROM deposits;
```
<hr>

3. **For each month - how many Data Bank customers make more than 1 deposit and either 1 purchase or 1 withdrawal in a single month?**

```sql
WITH monthly_transactions AS(
	SELECT
		customer_id,
		DATE_PART('month', txn_date) AS mnth,
		SUM(CASE WHEN txn_type = 'deposit' THEN 0 ELSE 1 END) AS deposit_count,
		SUM(CASE WHEN txn_type = 'purchase' THEN 0 ELSE 1 END) AS purchase_count,
		SUM(CASE WHEN txn_type = 'withdrawl' THEN 1 ELSE 0 END) AS withdrawl_count
	FROM data_bank.customer_transactions
	GROUP BY customer_id, mnth
)

SELECT
	mnth,
	COUNT(DISTINCT customer_id) AS customer_count
WHERE deposit_count > 1 
AND (purchase_count >=1 OR withdrawl_count >=1)

GROUP BY mnth
ORDER BY mnth;
```
<hr>

4. **What is the closing balance for each customer at the end of the month?**

```sql
WITH txn_monthly_balance_cte AS
  (SELECT customer_id,
          txn_amount,
          month(txn_date) AS txn_month,
          SUM(CASE
                  WHEN txn_type="deposit" THEN txn_amount
                  ELSE -txn_amount
              END) AS net_transaction_amt
   FROM customer_transactions
   GROUP BY customer_id,
            month(txn_date)
   ORDER BY customer_id)
SELECT customer_id,
       txn_month,
       net_transaction_amt,
       sum(net_transaction_amt) over(PARTITION BY customer_id
                                     ORDER BY txn_month ROWS BETWEEN UNBOUNDED preceding AND CURRENT ROW) AS closing_balance
FROM txn_monthly_balance_cte;
```
<hr>

5. **What percentage of customers move from a positive balance in the first month to a negative balance in the second month?**

```sql

WITH following_month_cte AS (
  SELECT
    customer_id, 
    ending_month, 
    ending_balance, 
    LEAD(ending_balance) OVER (
      PARTITION BY customer_id 
      ORDER BY ending_month) AS following_balance
  FROM ranked_monthly_balances
)
, variance_cte AS (
  SELECT *
  FROM following_month_cte
  WHERE ending_month = '2020-01-31'
    AND ending_balance::TEXT NOT LIKE '-%'
    AND following_balance::TEXT LIKE '-%'
)

SELECT 
  ROUND(100.0 * 
    COUNT(customer_id) 
    / (SELECT COUNT(DISTINCT customer_id) 
    FROM ranked_monthly_balances),1) AS positive_to_negative_percentage
FROM variance_cte;
