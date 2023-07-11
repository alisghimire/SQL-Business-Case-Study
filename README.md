# SQL-Case-Study-Data-Mart

# **Case Study Questions**

The following case study questions require some data cleaning steps before we start to unpack Danny’s key business questions in more depth.

# **Available Data**

![case-study-5-erd](https://github.com/aliscfc/SQL-Case-Study---Data-Mart/assets/109817585/e0285019-abda-4b13-9156-a26e1c76e77f)


### **1. Data Cleansing Steps**

In a single query, perform the following operations and generate a new table in the `data_mart` schema named `clean_weekly_sales`:

- Convert the `week_date` to a `DATE` format
- Add a `week_number` as the second column for each `week_date` value, for example any value from the 1st of January to 7th of January will be 1, 8th to 14th will be 2 etc
- Add a `month_number` with the calendar month for each `week_date` value as the 3rd column
- Add a `calendar_year` column as the 4th column containing either 2018, 2019 or 2020 values
- Add a new column called `age_band` after the original `segment` column using the following mapping on the number inside the `segment` value

| segment | age_band |
| --- | --- |
| 1 | Young Adults |
| 2 | Middle Aged |
| 3 or 4 | Retirees |
- Add a new `demographic` column using the following mapping for the first letter in the `segment` values:

| segment | demographic |
| --- | --- |
| C | Couples |
| F | Families |
- Ensure all `null` string values with an `"unknown"` string value in the original `segment` column as well as the new `age_band` and `demographic` columns
- Generate a new `avg_transaction` column as the `sales` value divided by `transactions` rounded to 2 decimal places for each record

```sql
CREATE TABLE clean_weekly_sales AS(
SELECT
	TO_DATE(week_date, 'DD/MM/YY') AS week_date,
	DATE_PART(week, TO_DATE(week_date, 'DD/MM/YY') AS week_number,
	DATE_PART(month, TO_DATE(week_date, 'DD/MM/YY') AS month_num,
	DATE_PART(year, TO_DATE(week_date, 'DD/MM/YY') AS year_num,
	region,
	platform,
	segment,
	CASE
		WHEN RIGHT(segment, 1) = '1' THEN 'Young Adults'
		WHEN RIGHT(segment, 1) = '2' THEN 'Middle Aged'
		WHEN RIGHT(segment, 1) in ('3','4') THEN 'Retirees'
		ELSE 'unknown' END AS age_band,
	CASE
		WHEN LEFT(segment, 1) = 'C' THEN 'Couples'
		WHEN LEFT(segment, 1) = 'F' THEN 'Families'
		ELSE 'unknown' END AS demographic,
	transactions,
	ROUND((sales::NUMERIC/transactions), 2) As avg_transaction,
	sales

FROM data_mart.weekly_sales
);
```
---
### **2. Data Exploration**

1. **What day of the week is used for each `week_date` value?**

```sql
SELECT 
	DISTINCT(TO_CHAR(week_date, 'day')) AS week_day
FROM clean_weekly_sales;
```

**Answer:**

Monday

---
2. **How many total transactions were there for each year in the dataset?**

```sql
SELECT
	year_num,
	SUM(transactions) AS total_transactions
FROM clean_weekly_sales
GROUP BY year_num
ORDER BY calendar_year;
```

**Answer:**

| year_num | total_transactions |
| --- | --- |
| 2018 | 346406460 |
| 2019 | 365639285 |
| 2020 | 375813651 |

---
3. **What is the total sales for each region for each month?**

```sql
SELECT
	region,
	month,
	SUM(sales) AS total_sales
FROM clean_weekly_sales
GROUP BY month_num, region
ORDER BY month_num, region;
```

**Answer:**

*Note: Only March sales is displayed here as the actual output is too long.*

| month_num | region | total_sales |
| --- | --- | --- |
| 3 | AFRICA | 567767480 |
| 3 | ASIA | 529770793 |
| 3 | CANADA | 144634329 |
| 3 | EUROPE | 35337093 |
| 3 | OCEANIA | 783282888 |
| 3 | SOUTH AMERICA | 71023109 |
| 3 | USA | 225353043 |

---
4. **What is the total count of transactions for each platform?**

```sql
SELECT
	platform,
	SUM(transactions) AS total_transactions
FROM clean_weekly_sales
GROUP BY platform;
```

**Answer:**

| platform | total_transactions |
| --- | --- |
| Retail | 1081934227 |
| Shopify | 5925169 |

---
5. **What is the percentage of sales for Retail vs Shopify for each month?**

```sql
WITH monthly_transactions AS(
	SELECT
		year_num,
		month_num,
		platform,
		SUM(sales) AS monthly_sales
FROM clean_weekly_sales
GROUP BY year_num, month_num, platform
)

SELECT
	year_num,
	month_num,
	platform,
	ROUND(100 * MAX(CASE WHEN platform = 'Retail' THEN monthly_sales ELSE NULL END)
	/ SUM(monthly_sales), 2) AS retail_percentage,
	ROUND(100 * MAX(CASE WHEN platform = 'Shopify' THEN monthly_sales ELSE NULL END) 
	/ SUM(monthly_sales), 2) AS shopify_percentage
FROM monthly_transactions
GROUP BY year_num, month_num
ORDER BY year_num, month_num;

```

**Answer:**

*Note: Actual result consists of 20 rows, here’s a sample of 5.*

| year_num | month_num | retail_percentage | shopify_percentage |
| --- | --- | --- | --- |
| 2018 | 3 | 97.92 | 2.08 |
| 2018 | 4 | 97.93 | 2.07 |
| 2018 | 5 | 97.73 | 2.27 |
| 2018 | 6 | 97.76 | 2.24 |
| 2018 | 7 | 97.75 | 2.25 |

---
6. **What is the percentage of sales by demographic for each year in the dataset?**

```sql
WITH yearly_transactions AS(
	SELECT
		year_num,
		demographic,
		SUM(sales) as yearly_sales
	FROM clean_weekly_sales
	GROUP BY year_num, demographic
)

SELECT
	year_num,
	ROUND(100 * MAX
		(CASE WHEN demographic = 'Couples' THEN yearly_sales ELSE NULL END)
	/ SUM(yearly_sales), 2) AS couples_percentage,
	ROUND(100 * MAX
		(CASE WHEN demographic = 'Families' THEN yearly_sales ELSE NULL END)
	/ SUM(yearly_sales), 2) AS families_percentage,
	ROUND(100 * MAX
		(CASE WHEN demographic = 'unknown' THEN yearly_sales ELSE NULL END)
	/ SUM(yearly_sales), 2) AS unknown_percentage
FROM yearly_transactions
GROUP BY year_num;	
		
```

**Answer:**

| year_num | couples_percentage | families_percentage | unknown_percentage |
| --- | --- | --- | --- |
| 2019 | 27.28 | 32.47 | 40.25 |
| 2018 | 26.38 | 31.99 | 41.63 |
| 2020 | 28.72 | 32.73 | 38.55 |

---
7. **Which `age_band` and `demographic` values contribute the most to Retail sales?**

```sql
SELECT
	age_band,
	demographic,
	SUM(sales) AS total_sales,
	ROUND(100 * SUM(sales)::NUMERIC / SUM(SUM(sales)) OVER(), 1) AS each_contribution
FROM clean_weekly_sales
WHERE platform = 'Retail'
GROUP BY age_band, demographic
ORDER BY total_sales DESC;
```

**Answer:**

| age_band | demographic | total_sales | each_contribution |
| --- | --- | --- | --- |
| unknown | unknown | 16067285533 | 40.5 |
| Retirees | Families | 6634686916 | 16.7 |
| Retirees | Couples | 6370580014 | 16.1 |
| Middle Aged | Families | 4354091554 | 11.0 |
| Young Adults | Couples | 2602922797 | 6.6 |
| Middle Aged | Couples | 1854160330 | 4.7 |
| Young Adults | Families | 1770889293 | 4.5 |

---
8. **Can we use the `avg_transaction` column to find the average transaction size for each year for Retail vs Shopify? If not - how would you calculate it instead?**

```sql
SELECT
	year_num,
	platform,
	ROUND(AVG(avg_transactions), 0) AS avg_trans_row,
	SUM(sales) / sum(transactions) AS avg_trans_group
FROM clean_weekly_sales
GROUP BY year_num, platform
ORDER BY year_num, platform;
```

**Answer:**

| year_num | platform | avg_trans_row | avg_trans_group |
| --- | --- | --- | --- |
| 2018 | Retail | 43 | 36 |
| 2018 | Shopify | 188 | 192 |
| 2019 | Retail | 42 | 36 |
| 2019 | Shopify | 178 | 183 |
| 2020 | Retail | 41 | 36 |
| 2020 | Shopify | 175 | 179 |

---
### **3. Before & After Analysis**

This technique is usually used when we inspect an important event and want to inspect the impact before and after a certain point in time.

Taking the `week_date` value of `2020-06-15` as the baseline week where the Data Mart sustainable packaging changes came into effect.

We would include all `week_date` values for `2020-06-15` as the start of the period **after** the change and the previous `week_date` values would be **before**

Using this analysis approach - answer the following questions:

1. **What is the total sales for the 4 weeks before and after `2020-06-15`? What is the growth or reduction rate in actual values and percentage of sales?**

```sql
SELECT DISTINCT week_number
FROM clean_weekly_sales
WHERE week_date = '2020-06-15' 
  AND calendar_year = '2020';
```

*Note: ^This gives us the week number of date 2020-06-15 which is 25. Now we calculate four weeks before and after (21 to 28).*

```sql
WITH packaging_sales AS(
	SELECT
		week_date,
		week_number,
		SUM(sales) AS total_sales
	FROM clean_weekly_sales
	WHERE (week_number BETWEEN 21 AND 28)
	AND year_num = 2020
	GROUP BY week_date, week_number
),

changes_before_after AS(
	SELECT
		SUM(CASE WHEN week_number BETWEEN 21 AND 24 THEN total_sales END) AS before_sales,
		SUM(CASE WHEN week_number BETWEEN 25 AND 28 THEN total_sales END) AS after_sales
	FROM packaging_sales
)

SELECT
	after_sales - before_sales AS sales_variance,
	ROUND(100 * (after_sales - before_sales) / before_sales, 2) AS variance_percentage
FROM changes_before_after;
```

**Answer:**

| sales_variance | variance_percentage |
| --- | --- |
| -26884188 | -1.15 |

---
2. **What about the entire 12 weeks before and after?**

```sql
WITH packaging_sales AS(
	SELECT
		week_number,
		week_date,
		SUM(sales) AS total_sales
	FROM clean_weekly_sales
	WHERE week_number BETWEEN 13 AND 37
	GROUP BY week_date, week_num
),

before_after_change AS(
	SELECT
		SUM(CASE WHEN week_number BETWEEN 13 AND 24 THEN total_sales END) AS before_sales,
		SUM(CASE WHEN week_number BETWEEN 25 AND 37 THEN total_sales END) AS after_sales
	FROM packaging_sales
)

SELECT
	(after_sales - before_sales) AS sales_variance,
	ROUND(100 * (after_sales - before_sales) / before_sales, 2) AS variance_percentage
FROM before_after_change;
```

**Answer:**

| sales_variance | variance_percentage |
| --- | --- |
| -152325394 | -2.14 |
