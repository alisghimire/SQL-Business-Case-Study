# SQL Case Study #2 - Pizza Runner

## Data Cleaning & Transformation

### 🔨 Table: customer_orders

Looking at the `customer_orders` table below, we can see that there are

- In the `exclusions` column, there are missing/ blank spaces ' ' and null values.
- In the `extras` column, there are missing/ blank spaces ' ' and null values.

!https://user-images.githubusercontent.com/81607668/129472388-86e60221-7107-4751-983f-4ab9d9ce75f0.png

Our course of action to clean the table:

- Create a temporary table with all the columns
- Remove null values in `exlusions` and `extras` columns and replace with blank space ' '.

```sql
CREATE TEMP TABLE temp_customer_orders AS(
	SELECT
		order_id,
		customer_id,
		pizza_id,
		CASE
			WHEN exclusions IS null OR exclusions LIKE 'null' THEN ' '
			ELSE exclusions
			END AS exclusions,
	
		CASE(
			WHEN extras IS null OR extras LIKE 'null' THEN ' '
			ELSE extra
			END AS extras)
	FROM pizza_runner.customers_orders
);
```

This is how the clean `customers_orders_temp` table looks like and we will use this table to run all our queries.

!https://user-images.githubusercontent.com/81607668/129472551-fe3d90a0-1e8b-4f32-a2a7-2ecd3ac469ef.png

---

### 🔨 Table: runner_orders

Looking at the `runner_orders` table below, we can see that there are

- In the `exclusions` column, there are missing/ blank spaces ' ' and null values.
- In the `extras` column, there are missing/ blank spaces ' ' and null values

!https://user-images.githubusercontent.com/81607668/129472585-badae450-52d2-442e-9d50-e4d0d8fce83a.png

Our course of action to clean the table:

- In `pickup_time` column, remove nulls and replace with blank space ' '.
- In `distance` column, remove "km" and nulls and replace with blank space ' '.
- In `duration` column, remove "minutes", "minute" and nulls and replace with blank space ' '.
- In `cancellation` column, remove NULL and null and and replace with blank space ' '.

```sql
CREATE TEMP TABLE temp_runner_orders AS(
	SELECT
		order_id,
		runner_id
		CASE
			WHEN pickup_time LIKE 'null' THEN ' '
			ELSE pickup_time
			END AS pickup_time,
		CASE
			WHEN distance LIKE 'null' THEN ' '
			WHEN distance LIKE '%km' THEN TRIM('km' from duration)
			ELSE distance
			END AS distance,
		CASE
			WHEN duration LIKE 'null' THEN ' '
			WHEN duration LIKE '%minutes' THEN TRIM ('minutes' from duration)
			WHEN duration LIKE '%minute' THEN TRIM('minute' from duration)
			WHEN duration LIKE '%mins' THEN TRIM('mins' from duration)
			ELSE duration
			END AS duration,
		CASE
			WHEN cancellation IS NULL OR cancellation LIKE 'null' THEN ' '
			ELSE cancellation
			END AS cancellation
	FROM pizza_runner.runner_orders
);
```

Now, let’s convert `**pickup_time**`, `**distance**` and `**duration**` to the correct data types. 

```sql
ALTER TABLE temp_runner_orders
ALTER COLUMN pickup_time DATETIME,
ALTER COLUMN distance FLOAT,
ALTER COLUMN duration INT;
```

This is how the table `**temp_runner_orders**` looks like and we run all our queries.

!https://user-images.githubusercontent.com/81607668/129472778-6403381d-6e30-4884-a011-737b1eff7379.png

### **A. Pizza Metrics**

1. **How many pizzas were ordered?**

```sql
SELECT
	COUNT(*) AS total_order_count
FROM temp_customer_orders;
```

**Answer**: 14 pizzas were ordered.

---

2. **How many unique customer orders were made?**

```sql
SELECT 
	COUNT(DISTINCT order_id) AS unique_customer_orders
FROM temp_customer_orders;
```

**Answer**: Altogether 10 unique customer orders were made.

---

3. **How many successful orders were delivered by each runner?**

```sql
SELECT
	runner_id,
	COUNT(order_id)
FROM #runner_orders
WHERE distance != 0
GROUP BY runner_id;
```

**Answer:** 

- Runner 1 has 4 successful delivered orders.
- Runner 2 has 3 successful delivered orders.
- Runner 3 has 1 successful delivered order.

---

4. **How many of each type of pizza was delivered?**

```sql
SELECT
	p.pizza_name,
	COUNT(DISTINCT pizza_id) AS type_delivered
FROM #customer_orders c
JOIN runner_orders r
	ON c.order_id = r.order_id
JOIN pizza_names p
	ON c.pizza_id = p.pizza_id
WHERE r.distance != 0
GROUP BY p.pizza_name;
```

**Answer:** There were 9 Meatlovers and 3 Vegetarian pizzas delivered.

---

5. **How many Vegetarian and Meatlovers were ordered by each customer?**

```sql
SELECT
	c.customer_id,
	p.pizza_name,
	COUNT(p.pizza_name) AS order_count,
FROM #customer_orders c
JOIN pizza_names p
ON c.pizza_id = p.pizza_id
GROUP BY c.customer_id, p.pizza_name
ORDER BY c.customer_id;
```

**Answer:** 

- Customer 101 ordered 2 Meatlovers pizzas and 1 Vegetarian pizza.
- Customer 102 ordered 2 Meatlovers pizzas and 2 Vegetarian pizzas.
- Customer 103 ordered 3 Meatlovers pizzas and 1 Vegetarian pizza.
- Customer 104 ordered 1 Meatlovers pizza.
- Customer 105 ordered 1 Vegetarian pizza.

---

6. **What was the maximum number of pizzas delivered in a single order?**

```sql
WITH pizza_count_cte AS(
	SELECT
		c.order_id,
		COUNT(c.pizza_id) AS pizza_order
	FROM #customer_orders c
	JOIN runner_orders r
	ON c.order_id = r.order_id
	WHERE distance != 0
	GROUP BY c.order_id
)

SELECT
	MAX(pizza_order) as max_pizza_number
FROM pizza_count_cte;
```

**Answer:** Max. number of pizza delivered in a single order is 3.

---

7. **For each customer, how many delivered pizzas had at least 1 change and how many had no changes?**

```sql
SELECT
	c.customer_id
	SUM(
		CASE WHEN c.exclusions <> ' ' OR c.extras <> ' ' THEN 1
		ELSE 0 END ) AS atleast_1_change,
	SUM(
		CASE WHEN c.exclusions = ' ' OR c.extras = ' ' THEN 1
		ELSE 0 END) AS no_change
FROM #customers_orders c
JOIN #runner_orders r
ON c.order_id = r.order_id
GROUP BY c.customer_id
ORDER BY c.customer_id;
```

**Answer:** Customers 103, 104, and 105 requested at least 1 changes to their pizza. The rest ordered as it is.

---

8. **How many pizzas were delivered that had both exclusions and extras?**

```sql
SELECT
	SUM(
		CASE WHEN exclusions IS NOT NULL AND extras IS NOT NULL THEN 1
		ELSE 0 END) AS altered_pizza_count
	FROM #customer_orders c
	JOIN #runner_orders r
	ON c.order_id = r.order_id
	WHERE r.distance != 0
		AND exclusions <> ' '
		AND extras <> ' ';
```

**Answer:** 1 pizza had both exclusions and extras.

---

9. **What was the total volume of pizzas ordered for each hour of the day?**

```sql
SELECT
	DATEPART(HOUR, [order_time]) AS hour_of_day,
	COUNT(order_id) AS volume_of_orders
FROM #customer_orders
GROUP BY hour_of_day;
```

**Answer:** 

- Highest volume of pizza ordered is at 13 (1:00 pm), 18 (6:00 pm) and 21 (9:00 pm).
- Lowest volume of pizza ordered is at 11 (11:00 am), 19 (7:00 pm) and 23 (11:00 pm).

---

10. **What was the volume of orders for each day of the week?**

```sql
SELECT
	FORMAT(DATEADD(DAY, 2, order_time), 'dddd') AS day_of_week,
	COUNT(order_id) AS order_volume
FROM #customer_orders
GROUP BY day_of_week;
```

**Answer:** 

- There are 5 pizzas ordered on Friday and Monday.
- There are 3 pizzas ordered on Saturday.
- There is 1 pizza ordered on Sunday.

---

### **B. Runner and Customer Experience**

1. **How many runners signed up for each 1 week period? (i.e. week starts `2021-01-01`)**

```sql
SELECT
	DATEPART(WEEK, registration_date) AS registration_week,
	COUNT(runner_id) AS runners_signup
FROM runners
GROUP BY registration_week;
```

**Answer:**

- On week 1 of Jan, 2 new runners signed up.
- On week 3 and 4, 1 new runner signed up.

---

2. **What was the average time in minutes it took for each runner to arrive at the Pizza Runner HQ to pickup the order?**

```sql
WITH average_time_cte AS(
	SELECT
		c.order_id,
		c.order_time
		r.pickup_time
		DATEDIFF(MINUTE, c.order_time, r.pickup_time) AS pickup_minutes
	FROM #customer_orders c
	JOIN #runner_orders r
	ON c.order_id = r.order_id
	WHERE r.distance != 0
	GROUP BY c.order_id, c.order_time, r.pickup_time
)

SELECT
	runner_id,
	AVG(pickup_minutes)
FROM average_time_cte
WHERE pickup_minutes > 1;
```

**Answer:** The average pickup time of runners is 15 minutes.

---

3. **Is there any relationship between the number of pizzas and how long the order takes to prepare?**

```sql
WITH prep_time_cte AS(
	SELECT
		c.order_id,
		COUNT(order_id) AS total_pizzas
		c.order_time,
		r.pickup_time,
		DATEDIFF(MINUTE, c.order_time, r.pickup_time) AS prep_minutes
	FROM #customer_orders c
	JOIN #runner_orders r
	ON c.order_id = r.order_id
	WHERE r.distance != 0
	GROUP BY c.order_id, c.order_time, r.pickup_time
);

SELECT
	total_pizzas,
	AVG(prep_minutes)
FROM prep_time_cte
WHERE prep_minutes > 1
GROUP BY total_pizzas;
```

**Answer:**

- On average, it takes 12 minutes to prepare 1 pizza
- 16 minutes for 2 pizzas (**MOST EFFICIENT**)
- And 30 minutes for 3 pizzas.

---

4. **What was the average distance traveled for each customer?**

```sql
SELECT
	c.customer_id,
	AVG(distance) AS avg_distance
FROM #customer_orders c
JOIN #runner_orders r
ON c.order_id = r.order_id
WHERE distance != 0
GROUP BY c.customer_id;
```

Answer: 

- 104 is the nearest customer with an average distance of 10 km.
- 105 is the farthest with an average distance of 25 km.

---

5. **What was the difference between the longest and shortest delivery times for all orders?**

```sql
SELECT
	MAX(duration::NUMERIC) - MIN(duration::NUMERIC) AS total_time_difference
FROM #runner_orders
WHERE duration NOT LIKE ' ';
```

**Answer:** The difference between longest (40 minutes) and shortest (10 minutes) delivery time for all orders is 30 minutes.****

---

6. **What was the average speed for each runner for each delivery and do you notice any trend for these values?**

```sql
SELECT
	r.runner_id,
	c.customer_id,
	c.order_id,
	COUNT(c.order_id) AS pizza_count,
	r.distance,
	ROUND(r.distance/r.duration * 60), 2) AS avg_speed
FROM #runner_orders r
JOIN #customer_orders c
ON r.order_id = c.order_id
WHERE r.distance != 0
GROUP BY r.runner_id, c.customer_id. c.order_id, r.distance
ORDER BY c.order_id;
```

**Answer:** 

- Runner 1’s average speed ranges from 37.5km/h to 60km/h.
- Runner 2’s average speed ranges from 35.1km/h to 93.6km/h. (**Fluctuation of 300%**)
- Runner 3’s average range is 40km/h

---

7. **What is the successful delivery percentage for each runner?**

```sql
SELECT 
  runner_id, 
  ROUND(100 * SUM(
    CASE WHEN distance = 0 THEN 0
    ELSE 1 END) / COUNT(*), 0) AS success_percentage
FROM #runner_orders
GROUP BY runner_id;
```

**Answer:** 

- Runner 1 = 100% successful delivery.
- Runner 2 = 75% successful delivery.
- Runner 3 = 50% successful delivery
