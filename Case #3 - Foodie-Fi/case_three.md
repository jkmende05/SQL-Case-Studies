# Case Study Three: Foodie-Fi
### Creating Database and Tables of Data
First, databases and tables where required and built using the code provided from the website where these case studies were found. Because of the length of this code, it is not included in this markdown file, but can be found here: https://8weeksqlchallenge.com/case-study-3/

### Part A: Customer Journey
```SQL
SELECT *
FROM subscriptions
WHERE customer_id <= 8;
```
Using the code above, the data for all customers with a customer_id less than or equal to 19 was determined and shown. 

Customer One:
Customer One started with a free trial before transitioning to a basic monthly plan.

Customer Two:
Customer Two, like Customer One, started with a free trial and then a pro monthly plan.

Customer Three:
Customer Three began with a free trial and then bought a basic monthly plan.

Customer Four:
Customer Four had a free trial, then a basic monthly plan before terminating the subscription a couple of months later. 

Customer Five:
Customer Five followed the exact same process as Customers One and Three. They had a free trial, then a basic monthly plan.

Customer Six:
Customer Six had a free trial, then a basic monthly plan, then churned their subscription.

Customer Seven:
Customer Seven followed up their free trial with a basic monthly plan before moving to a pro monthly plan.

Customer Eight:
Customer Eight did the exact same as customer seven. They had a free trial, basic monthly plan, and a pro monthly plan.

### Part B: Data Analysis Questions
Question One: Determining the total number of customers
```SQL
SELECT COUNT(DISTINCT customer_id) AS unique_customers
FROM subscriptions;
```

Question Two: Determining monthly distribution of trial start dates. 
```SQL
SELECT
	MONTH(start_date) AS month_num,
    MONTHNAME(start_date) AS month_name,
    COUNT(s.customer_id) AS total_trial_plans
FROM subscriptions AS s
WHERE s.plan_id = 0
GROUP BY MONTH(start_date), MONTHNAME(start_date)
ORDER BY month_num;
```

Question Three: Counting the number of each plan in 2021.
```SQL
SELECT
	p.plan_id,
    p.plan_name,
    COUNT(s.customer_id) AS num_subscriptions
FROM subscriptions AS s
JOIN plans AS p
	ON s.plan_id = p.plan_id
WHERE YEAR(start_date) > 2020
GROUP BY p.plan_id, p.plan_name
ORDER BY p.plan_id;
```

Question Four: Number and percentage of customers who have churned their subscriptions.
```SQL
SELECT
	COUNT(s.customer_id) AS churn_num,
    ROUND(100.0 * COUNT(s.customer_id) / (SELECT COUNT(DISTINCT customer_id) FROM subscriptions), 1) AS churn_percentage
FROM subscriptions AS s
JOIN plans AS p
	ON s.plan_id = p.plan_id
WHERE p.plan_id = 4;
```

Question Five: Number of customers that have churned right after the end of their free trial.
```SQL
WITH sub_order AS (
	SELECT
		s.customer_id,
		p.plan_id,
		ROW_NUMBER() OVER (PARTITION BY s.customer_id ORDER BY s.start_date) AS row_num
	FROM subscriptions AS s
	JOIN plans AS p
		ON s.plan_id = p.plan_id
)
SELECT
	COUNT(customer_id) AS num_churns,
    ROUND(100.0 * COUNT(customer_id) / (SELECT COUNT(DISTINCT customer_id) FROM subscriptions), 0) AS churn_percentage
FROM sub_order
WHERE plan_id = 4 AND row_num = 2;
```

Question Six: Number and percentage of customer plans after the end of their free trial.
```SQL
WITH next_plans AS (
	SELECT
		customer_id,
        plan_id,
        LEAD(plan_id) OVER (PARTITION BY customer_id ORDER BY plan_id) AS next_plan_id
    FROM subscriptions AS s
)
SELECT
	next_plan_id,
    COUNT(customer_id) AS converted_customers,
    ROUND(100.0 * COUNT(customer_id) / (SELECT COUNT(DISTINCT customer_id) FROM subscriptions), 2) AS converted_percentage
FROM next_plans
WHERE next_plan_id IS NOT NULL AND plan_id = 0
GROUP BY next_plan_id
ORDER BY next_plan_id;
```

Question Seven: Determining customer count and percentage of all five different plans at the end of 2020.
```SQL
WITH next_dates AS (
	SELECT
		customer_id,
        plan_id,
        start_date,
        LEAD(start_date) OVER (PARTITION BY customer_id ORDER BY start_date) AS next_date
	FROM subscriptions
    WHERE start_date <= '2020-12-31'
)
SELECT
	plan_id,
    COUNT(plan_id) AS customers,
    ROUND(100.0 * COUNT(customer_id) / (SELECT COUNT(DISTINCT customer_id) FROM subscriptions), 2) AS percentage
FROM next_dates
WHERE next_date IS NULL
GROUP BY plan_id
ORDER BY plan_id;
```

Question Eight: Number of customers that upgraded to an annual plan during 2020.
```SQL
SELECT
	COUNT(DISTINCT customer_id) AS num_upgrades
FROM subscriptions AS s
WHERE plan_id = 3 AND YEAR(start_date) = 2020;
```

Question Nine: Determining the average number of days to upgrade to an annual plan.
```SQL
WITH trial_plan AS (
	SELECT
		customer_id,
        start_date AS trial_date
	FROM subscriptions
    WHERE plan_id = 0
), annual_plan AS (
	SELECT
		customer_id,
        start_date AS annual_date
	FROM subscriptions
    WHERE plan_id = 3
)
SELECT
	ROUND(AVG(annual_plan.annual_date - trial_plan.trial_date), 2) AS avg_days
FROM trial_plan
JOIN annual_plan
	ON trial_plan.customer_id = annual_plan.customer_id;
```

Question Ten: Breaking down the number of days to upgrade to an annual plan into 30 day periods.
```SQL
WITH trial_plan AS (
	SELECT
		customer_id,
        start_date AS trial_date
        FROM subscriptions
        WHERE plan_id = 0
), annual_plan AS (
	SELECT
		customer_id,
        start_date AS annual_date
        FROM subscriptions
        WHERE plan_id = 3
), bins AS (
	SELECT
		FLOOR((DATEDIFF(a.annual_date, t.trial_date) - 0) / (365 / 12)) + 1 AS avg_days_to_upgrade
    FROM trial_plan AS t
    JOIN annual_plan AS a
		ON t.customer_id = a.customer_id
)
SELECT 
  CONCAT((avg_days_to_upgrade - 1) * 30, ' - ', (avg_days_to_upgrade * 30), ' days') AS bucket,
  COUNT(*) AS num_of_customers
FROM bins
GROUP BY avg_days_to_upgrade
ORDER BY avg_days_to_upgrade;
```

Question Eleven: Determining how many customers downgraded from a pro monthly to a basic monthly plan in 2020.
```SQL
WITH downgrades AS (
	SELECT
		s.customer_id,
        p.plan_id,
        p.plan_name,
        LEAD(p.plan_id) OVER (
			PARTITION BY s.customer_id
            ORDER BY s.start_date) AS next_plan_id
    FROM subscriptions AS s
    JOIN plans AS p
		ON s.plan_id = p.plan_id
	WHERE YEAR(start_date) = 2020
)
SELECT
	COUNT(customer_id) AS num_customers
FROM downgrades
WHERE plan_id = 2 AND next_plan_id = 1;
```

### Part C: Payment Questions
The following SQL code below includes the creation of table showing all required payments for each customer in 2020. It is based on each customer's subscription.
```SQL
WITH RECURSIVE RecursivePayments (
    customer_id, plan_id, plan_name, payment_date, amount, start_date
) AS (
    SELECT
        s.customer_id,
        s.plan_id,
        p.plan_name,
        s.start_date AS payment_date,
        p.price AS amount,
        s.start_date
    FROM subscriptions AS s
    JOIN plans AS p ON s.plan_id = p.plan_id
    WHERE YEAR(s.start_date) = 2020
    AND p.plan_name NOT IN ('trial', 'churn')
    
    UNION ALL

    SELECT
        rp.customer_id,
        rp.plan_id,
        rp.plan_name,
        CASE  
            WHEN rp.plan_name IN ('Basic Monthly', 'Pro Monthly')  
                THEN DATE_ADD(rp.payment_date, INTERVAL 1 MONTH)
            WHEN rp.plan_name IN ('Basic Annual', 'Pro Annual')  
                THEN DATE_ADD(rp.payment_date, INTERVAL 1 YEAR)
        END AS payment_date,
        rp.amount,
        rp.start_date
    FROM RecursivePayments AS rp
    WHERE  
        rp.payment_date < '2020-12-31' 
        AND rp.plan_name NOT IN ('Trial', 'Churned')
        AND (
            (rp.plan_name IN ('Basic Monthly', 'Pro Monthly') AND DATE_ADD(rp.payment_date, INTERVAL 1 MONTH) <= '2020-12-31') 
            OR (rp.plan_name IN ('Basic Annual', 'Pro Annual') AND DATE_ADD(rp.payment_date, INTERVAL 1 YEAR) <= '2020-12-31')
        )
)
SELECT  
    customer_id,
    plan_id,
    plan_name,
    payment_date,
    amount
FROM RecursivePayments
WHERE YEAR(payment_date) = 2020
ORDER BY customer_id, payment_date;
```

