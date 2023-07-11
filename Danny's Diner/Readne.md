# SQL Case Study #1 - Danny's Diner


## Problem Statement

Danny wants to use the data to answer a few simple questions about his customers, especially about their visiting patterns, how much money they’ve spent and also which menu items are their favourite.

## Entity Relationship Diagram

![Screenshot 2023-07-09 015735](https://github.com/aliscfc/SQL-Case-Study/assets/109817585/2356cad1-b489-4cd1-99fc-0172dba63729)


## Questions and Solution


1. **What is the total amount each customer spent at the restaurant?**

```sql
SELECT 
	s.customer_id,
	SUM(m.price) AS total_spent
FROM dannys_diner.sales s
INNER JOIN dannys_diner.menu m
ON s.product_id = m.product_id
GROUP BY s.customer_id
ORDER BY s.customer_id ASC;
```

**Steps:**

- I applied the **SUM** function to calculate the total sales contributed by each customer.
- I use the **JOIN** operation to merge the **`dannys_diner.sales`** and **`dannys_diner.menu`** tables, as the **`sales.customer_id`** and **`menu.price`** are present in both tables.
- Finally, I group the aggregated results by **`sales.customer_id`** to obtain the total sales for each unique customer.
  
<hr>

2. **How many days has each customer visited the restaurant?**

```sql
SELECT 
	customer_id,
	COUNT(DISTINCT order_date) AS days_visited
FROM sales
GROUP BY customer_id;
```

**Steps:**

- I calculated the unique number of visits for each customer using the **COUNT(DISTINCT** `**order_date**`) function.
- Then I applied the **DISTINCT** keyword to avoid double-counting and ensure each day is counted only once.

<hr>

3. **What was the first item from the menu purchased by each customer?**

```sql
WITH ordered_sales AS(
	SELECT 
		s.customer_id,
		s.order_date,
		m.product_name,
		DENSE_RANK() OVER(
			PARTITION BY s.customer_id
			ORDER BY s.order_date) AS rank
	FROM dannys_diner.sales s
	INNER JOIN dannys_diner.menu m
	ON s.product_id = m.product_id)

SELECT
	customer_id,
	product_name
FROM ordered_sales
WHERE rank = 1
GROUP BY customer_id;
```

**Steps:**

- I create a Common Table Expression (CTE) named `**ordered_sales**` to store intermediate results.
- Within the CTE, I added a new column named "rank" using the **DENSE_RANK()** window function. This function calculates the row number within each **`customer_id`** partition based on the **`order_date`**. The **PARTITION BY** clause divides the data by **`customer_id`**, and the ORDER BY clause orders the rows within each partition by **`order_date`**.
- In the outer query, I selected the appropriate columns and filter the results using the **WHERE** clause. I retrieve only the rows where the "rank" column equals 1, which represents the first row within each **`customer_id`** partition.
- Lastly, I used the **GROUP BY** clause to group the results by **`customer_id`**.

<hr>

4. **What is the most purchased item on the menu and how many times was it purchased by all customers?**

```sql
SELECT 
	m.product_name,
	COUNT(s.product_id) AS most_purchased
FROM dannys_diner.sales s
INNER JOIN dannys_diner.menu m
ON s.product_id = m.product_id
GROUP BY m.product_name
ORDER BY most_purchased DESC
LIMIT 1;
```

**Steps:**

- I performed a **COUNT** aggregation on the **`product_id`** column and order the result in descending order using the **`most_purchased`** field.
- Then, I applied the **LIMIT 1** clause to filter and retrieve the most purchased items.

<hr>

5. **Which item was the most popular for each customer?**

```sql
WITH most_popular AS(
	SELECT
		s.customer_id,
		m.product_name,
		COUNT(m.product_id) as order_count
		DENSE_RANK() OVER(
			PARTITION BY s.customer.id
			ORDER BY COUNT(s.customer_id) DESC) AS rank
FROM dannys_diner.menu m
INNER JOIN dannys_diner.sales s
ON m.product_id = s.product_id
GROUP BY s.customer_id, m.product_name)

SELECT
	customer_id
	product_name
	order_count
FROM most_popular
WHERE rank = 1;
```

**Steps:**

- I created a **`most_popular`** CTE and performed an inner join between the **`menu`** and **`sales`** tables using the **`product_id`** column.
- I grouped the results by **`s.customer_id`** and **`m.product_name`**, and calculated the occurrence count of **`m.product_id`** for each group.
- Then, I used the **DENSE_RANK()** window function to determine the ranking of each **`s.customer_id`** partition based on the count of orders (**COUNT**(**`s.customer_id`**)) in descending order.
- In the outer query, I selected the appropriate columns and applied a filter in the **WHERE** clause to retrieve only the rows where the rank column equals 1. These rows represent the highest order count for each customer.

<hr>

6. **Which item was purchased first by the customer after they became a member?**

```sql
WITH join_member AS(
	SELECT 
		mem.customer_id,
		s.product_id
		ROW_NUMBER() OVER(
			PARTITION BY mem.customer_id
			ORDER BY s.order_date ) AS row_num
	FROM dannys_diner.members mem
	INNER JOIN sales s
	ON mem.customer_id = s.customer_id
	AND s.order_date > mem.join_date
	)

SELECT
	customer_id,
	product_name
FROM join_member
INNER JOIN dannys_diner.menu m
ON join_member.product_id = m.product_id
WHERE row_num = 1
ORDER BY customer_id ASC;
```

**Steps:**

- I created a CTE named **`join_member`** to hold required columns and calculated the row number using **ROW_NUMBER()** function.
- I then joined the **`dannys_diner.members`** and **`dannys_diner.sales`** tables on the **`customer_id`** column, considering sales occurring after the member's **`join_date`**.
- In the outer query, I joined the **`join_member`** CTE with the **`dannys_diner.menu`** table on the **`product_id`** column.
- I filtered the results to retrieve only the rows with **`row_num`** = 1, representing the first row within each **`customer_id`** partition.
- Finally, I ordered the results by **`customer_id`** in ascending order.

<hr>

7. **Which item was purchased just before the customer became a member?**

```sql
WITH join_before AS(
	SELECT
		mem.customer_id,
		s.product_id,
		ROW_NUMBER() OVER(
			PARTITION BY mem.customer_id
			ORDER BY s.order_date DESC) AS row_num

	FROM dannys_diner.members mem
	INNER JOIN dannys_diner.sales s
	ON mem.customer_id = s.customer_id
	AND s.order_date < mem.join_date
)

SELECT
	join_before.customer_id,
	m.product_name
FROM join_before
INNER JOIN menu m
ON join_before.product_id = m.product_id
WHERE row_num = 1
ORDER BY join_before.customer_id ASC;
```

**Steps:**

- I created a CTE called **`join_before`** to hold the intermediate results and calculated the rank using the **ROW_NUMBER()** function. The rank was determined based on the **`order_date`** of the sales in descending order within each customer's group.
- I joined the **`dannys_diner.members`** table with the **`dannys_diner.sales`** table based on the **`customer_id`** column, only including sales that occurred before the customer joined as a member.
- Next, I joined the **`join_before`** CTE with the **`dannys_diner.menu`** table based on the **`product_id`** column.
- I filtered the result set to include only the rows where the rank is 1, representing the earliest purchase made by each customer before they became a member.
- Finally, I sorted the result by **`customer_id`** in ascending order.

<hr>

8. **What is the total items and amount spent for each member before they became a member?**

```sql
SELECT
	s.customer_id,
	COUNT(s.product_id) AS total_items,
	SUM(m.price) AS total_spent
FROM dannys_diner.sales s
INNER JOIN dannys_diner.members mem
ON s.customer_id = mem.customer_id
AND s.order_date < mem.join_date
INNER JOIN dannys_diner.menu m
ON s.product_id = m.product_id

GROUP BY s.customer_id
ORDER BY s.customer_id;

```

**Steps:**

- I selected the columns **`sales.customer_id`** and calculated the **COUNT** of **`s.product_id`** and the **SUM** of **`m.price`** for each customer.
- From the **`dannys_diner.sales`** table, I joined the **`dannys_diner.members`** table on the **`customer_id`** column, ensuring that the sales order date is earlier than the member's join date.
- Next, I joined the **`dannys_diner.menu`** table to the **`dannys_diner.sales`** table on the **`product_id`** column.
- I grouped and ordered the results by the **`customer_id`**.

<hr>

9. **If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?**

```sql
WITH points_tally AS(
	SELECT m.product_id,
	CASE
		WHEN product_id = 1 THEN price * 20
		ELSE price * 10 END AS points
	FROM dannys_diner.menu m
)

SELECT
	s.customer_id,
	SUM(points_tally.points) AS total_points
	FROM dannys_diner.sales s
	INNER JOIN points_tally
	ON s.product_id = points_tally.product_id

GROUP BY s.customer_id
ORDER BY s.customer_id;
```

**Steps:**

- I performed a calculation using a conditional **CASE** statement to assign point values based on the **`product_id`**:
    - If the **`product_id`** is 1, I multiplied every $1 by 20 points.
    - Otherwise, I multiplied $1 by 10 points.
- Then, I calculated the total points for each customer by aggregating the results.

<hr>

10. **In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?**

```sql
WITH dates_cte AS (
  SELECT 
    customer_id, 
      join_date, 
      join_date + 6 AS valid_date, 
      DATE_TRUNC(
        'month', '2021-01-31'::DATE)
        + interval '1 month' 
        - interval '1 day' AS last_date
  FROM dannys_diner.members
)

SELECT 
  sales.customer_id, 
  SUM(CASE
    WHEN menu.product_name = 'sushi' THEN 2 * 10 * menu.price
    WHEN sales.order_date BETWEEN dates.join_date AND dates.valid_date THEN 2 * 10 * menu.price
    ELSE 10 * menu.price END) AS points
FROM dannys_diner.sales
INNER JOIN dates_cte AS dates
  ON sales.customer_id = dates.customer_id
  AND dates.join_date <= sales.order_date
  AND sales.order_date <= dates.last_date
INNER JOIN dannys_diner.menu
  ON sales.product_id = menu.product_id
GROUP BY sales.customer_id;
```

**Steps:**

- I created a **`dates_cte`** to calculate relevant dates.
- In **`dates_cte`**, I determined the **`valid_date`** by adding 6 days to the **`join_date`** and found the **`last_date`** of the month by subtracting 1 day from the last day of January 2021.
- From the dannys_diner.sales table, I joined the **`dates_cte`** based on the **`customer_id`** column, ensuring that the order_date of the sale falls within the specified range.
- Then, I joined the **`dannys_diner.menu`** table based on the **`product_id`** column.
- In the outer query, I calculated the points for each sale using a **CASE** statement:
    - If the **`product_name`** is 'sushi', I multiplied the price by 2 and then by 10. Additionally, if the order was placed between the join_date and valid_date, I performed the same multiplication.
    - For all other products, I multiplied the price by 10.
- Finally, I calculated the **SUM** of points for each customer.
