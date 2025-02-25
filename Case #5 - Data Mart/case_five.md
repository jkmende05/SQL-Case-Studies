# Case Five: Data Mart
### Creating Database and Tables of Data
First, databases and tables where required and built using the code provided from the website where these case studies were found. Because of the length of this code, it is not included in this markdown file, but can be found here: https://8weeksqlchallenge.com/case-study-5/

### Part One: Cleaning Data
```SQL
DROP TABLE IF EXISTS clean_weekly_sales;

CREATE TABLE clean_weekly_sales AS
SELECT
	STR_TO_DATE(week_date, '%d/%m/%y') AS week_date,
    WEEK(STR_TO_DATE(week_date, '%d/%m/%y')) AS week_number,
    MONTH(STR_TO_DATE(week_date, '%d/%m/%y')) AS month_number,
    YEAR(STR_TO_DATE(week_date, '%d/%m/%y')) AS calendar_year,
    region,
    platform,
    segment,
    CASE
		WHEN SUBSTRING(segment,2,1) = '1' THEN 'Young Adults'
		WHEN SUBSTRING(segment,2,1) = '2' THEN  'Middle Aged'
		WHEN SUBSTRING(segment,2,1) = '3' or SUBSTRING(segment,1,1) = '4' THEN 'Retirees'
		ELSE 'unknown' 
	END AS age_band,
	CASE WHEN SUBSTRING(segment,1,1) = 'C' THEN 'Couples'
		WHEN SUBSTRING(segment,1,1) = 'F' THEN 'Families'
		ELSE 'unknown'
	END AS demographic,
    transactions,
    ROUND(sales / transactions, 2) AS avg_transactions,
    sales
FROM weekly_sales;
```

### Part Two: Data Exploration
Question One: Determining days of week for each week_date value
```SQL
SELECT DISTINCT
	DAYNAME(week_date) AS day_of_week
FROM clean_weekly_sales;
```

Question Two: Determining range of week numbers missing from dataset
```SQL
WITH RECURSIVE numbers_week AS (
    SELECT 1 AS week_number
    UNION ALL
    SELECT week_number + 1
    FROM numbers_week
    WHERE week_number < 52
)
SELECT DISTINCT n.week_number
FROM numbers_week AS n
LEFT JOIN clean_weekly_sales AS sales
	ON n.week_number = sales.week_number
WHERE sales.week_number IS NULL;
```

Question Three: Calculating total transations for each year
```SQL
SELECT
	calendar_year,
    SUM(transactions) AS total_transactions
FROM clean_weekly_sales
GROUP BY calendar_year
ORDER BY calendar_year;
```

Question Four: Calculating total sales for each region for each month
```SQL
SELECT
	month_number,
    region,
    SUM(sales) AS total_sales
FROM clean_weekly_sales
GROUP BY month_number, region
ORDER BY month_number, region;
```

Question Five: Calculating total count of transactions for each platform
```SQL
SELECT
	platform,
    SUM(transactions) AS num_transactions
FROM clean_weekly_sales
GROUP BY platform;

```

Question Six: Calculating percentage of sales per platform for each month
```SQL
WITH monthly_transactions AS (
	SELECT
		calendar_year,
        month_number,
        platform,
        SUM(sales) AS num_sales
    FROM clean_weekly_sales
    GROUP BY calendar_year, month_number, platform
)
SELECT
	calendar_year,
    month_number,
    ROUND(100 * MAX(
		CASE
			WHEN platform = 'Retail' THEN num_sales
			ELSE NULL
		END) / SUM(num_sales), 2) AS retail_purchases,
	ROUND(100 * MAX(
		CASE
			WHEN platform = 'Shopify' THEN num_sales
			ELSE NULL
		END) / SUM(num_sales), 2) AS shopify_purchases
FROM monthly_transactions
GROUP BY calendar_year, month_number
ORDER BY calendar_year, month_number;
```

Question Seven: Determining the percentage of sales by demogrpahic for each year
```SQL
WITH yearly_demographic AS (
	SELECT
		calendar_year,
        demographic,
        SUM(sales) AS yearly_sales
	FROM clean_weekly_sales
    GROUP BY calendar_year, demographic
)
SELECT
	calendar_year,
	ROUND(100 * MAX(
		CASE
			WHEN demographic = 'Couples' THEN yearly_sales
			ELSE NULL
		END) / SUM(yearly_sales), 2) AS couples_percentage,
	ROUND(100 * MAX(
		CASE
			WHEN demographic = 'Families' THEN yearly_sales
			ELSE NULL
		END) / SUM(yearly_sales), 2) AS family_percentage,
	ROUND(100 * MAX(
		CASE
			WHEN demographic = 'unknown' THEN yearly_sales
			ELSE NULL
		END) / SUM(yearly_sales), 2) AS unknown_percentage
FROM yearly_demographic
GROUP BY calendar_year
ORDER BY calendar_year;
```

Question Eight: Finding which age_band and demographic contribute the most to Retail sales
```SQL
SELECT
	age_band,
    demographic,
    SUM(sales) AS retail_sales,
    ROUND(100 * SUM(sales) / SUM(SUM(sales)) OVER (), 1) AS contribution_percentage
FROM clean_weekly_sales
WHERE platform = 'Retail'
GROUP BY age_band, demographic
ORDER BY retail_sales DESC;
```

Question Nine: Using avg_transactions columns to find average transaction size for each year for each platform
```SQL
SELECT
	calendar_year,
    platform,
    ROUND(AVG(avg_transactions), 2) AS avg_transaction_size
FROM clean_weekly_sales
GROUP BY calendar_year, platform
ORDER BY calendar_year, platform;
```

### Part Three: Before and After Analysis
Question One: Total sales for four weeks before and after 2020-06-15
```SQL
SELECT DISTINCT week_number
FROM clean_weekly_sales
WHERE week_date = '2020-06-15' AND calendar_year = '2020';

WITH summarized_sales AS (
	SELECT
		week_date,
        week_number,
        SUM(sales) AS total_sales
	FROM clean_weekly_sales
    WHERE (week_number BETWEEN 20 AND 27) AND (calendar_year = 2020)
    GROUP BY week_date, week_number
), changes AS (
	SELECT
		SUM(
			CASE
				WHEN week_number BETWEEN 20 AND 23 THEN total_sales END) AS before_sales,
		SUM(
			CASE
				WHEN week_number BETWEEN 25 AND 27 THEN total_sales END) AS after_sales
    FROM summarized_sales
)
SELECT
    before_sales - after_sales AS sales_variance,
    ROUND(100 * (after_sales - before_sales) / before_sales, 2) AS variance_percentage
FROM changes;

```

Question Two: Total sales for twelve weeks before and after 2020-06-15
```SQL
WITH summarized_sales AS (
	SELECT
		week_date,
        week_number,
        SUM(sales) AS total_sales
	FROM clean_weekly_sales
    WHERE (week_number BETWEEN 20 AND 27) AND (calendar_year = 2020)
    GROUP BY week_date, week_number
), changes AS (
	SELECT
		SUM(
			CASE
				WHEN week_number BETWEEN 12 AND 23 THEN total_sales END) AS before_sales,
		SUM(
			CASE
				WHEN week_number BETWEEN 25 AND 36 THEN total_sales END) AS after_sales
    FROM summarized_sales
)
SELECT
    before_sales - after_sales AS sales_variance,
    ROUND(100 * (after_sales - before_sales) / before_sales, 2) AS variance_percentage
FROM changes;
```

Question Three: Sale metrics for the twelve week period compared to past years
```SQL
WITH summarized_sales AS (
	SELECT
		calendar_year,
        week_number,
        SUM(sales) AS total_sales
	FROM clean_weekly_sales
    WHERE (week_number BETWEEN 20 AND 27)
    GROUP BY calendar_year, week_number
), changes AS (
	SELECT
		calendar_year,
		SUM(
			CASE
				WHEN week_number BETWEEN 20 AND 23 THEN total_sales END) AS before_sales,
		SUM(
			CASE
				WHEN week_number BETWEEN 25 AND 27 THEN total_sales END) AS after_sales
    FROM summarized_sales
    GROUP BY calendar_year
)
SELECT
	calendar_year,
    before_sales - after_sales AS sales_variance,
    ROUND(100 * (after_sales - before_sales) / before_sales, 2) AS variance_percentage
FROM changes;
```
