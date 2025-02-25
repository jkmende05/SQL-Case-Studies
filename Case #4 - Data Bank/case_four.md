# Case Four: Data Bank
### Creating Dataframes and Tables
First, the tables and databases were created using the SQL code found on the website where these case studies were located. Due to the length of this code, it is excluded from this markdown file, but can be found from the following link: https://8weeksqlchallenge.com/case-study-4/

### Part A: Customer Nodes Exploration
Question One: Determining number of unique nodes
```SQL
SELECT
	COUNT(DISTINCT node_id) AS unique_nodes
FROM customer_nodes;
```

Question Two: Determining number of nodes per region
```SQL
SELECT
	r.region_name,
    COUNT(DISTINCT c.node_id) AS num_nodes
FROM customer_nodes AS c
JOIN regions AS r
	ON c.region_id = r.region_id
GROUP BY r.region_name;
```

Question Three: Determining number of customers allocated to each region
```SQL
SELECT
	region_id,
    COUNT(DISTINCT customer_id) AS num_customers
FROM customer_nodes
GROUP BY region_id
ORDER BY region_id;
```

Question Four: Calculating average number of days for each customer to reallocated.
```SQL
WITH node_days AS (
	SELECT
		customer_id,
        node_id,
        end_date - start_date AS days_in_node
	FROM customer_nodes
    WHERE end_date != '9999-12-31'
    GROUP BY customer_id, node_id, start_date, end_date
), total_node_days AS (
	SELECT
		customer_id,
        node_id,
        SUM(days_in_node) AS total_days_in_node
	FROM node_days
    GROUP BY customer_id, node_id
)
SELECT
	ROUND(AVG(total_days_in_node)) AS avg_reallocation_days
FROM total_node_days;
```

Question Five: Calculating median, 80th, and 95th percentile for the previously calculated average from question four.
```SQL
WITH node_days AS (
	SELECT
		customer_id,
        node_id,
        end_date - start_date AS days_in_node
	FROM customer_nodes
    WHERE end_date != '9999-12-31'
    GROUP BY customer_id, node_id, start_date, end_date
), total_node_days AS (
	SELECT
		customer_id,
        node_id,
        SUM(days_in_node) AS total_days_in_node
	FROM node_days
    GROUP BY customer_id, node_id
)
SELECT
	ROUND(0.50 * (COUNT(total_days_in_node) + 1)) AS median,
    ROUND(0.80 * (COUNT(total_days_in_node) + 1)) AS percentile_80th,
    ROUND(0.95 * (COUNT(total_days_in_node) + 1)) AS percentile_95th
FROM total_node_days;
```

### Part B: Customer Transactions
Question One: Determining unique count and total amount for each transaction type.
```SQL
SELECT
	txn_type,
    COUNT(customer_id) AS transaction_count,
    SUM(txn_amount) AS amount
FROM customer_transactions
GROUP BY txn_type;

```

Question Two: Calculating total historical deposit counts and amounts for all customers
```SQL
WITH deposits AS (
	SELECT
		customer_id,
        COUNT(customer_id) AS txn_count,
        AVG(txn_amount) AS avg_amount
    FROM customer_transactions
    WHERE txn_type = 'deposit'
    GROUP BY customer_id
)
SELECT
	ROUND(AVG(txn_count)) AS avg_deposit_counts,
    ROUND(AVG(avg_amount)) AS avg_deposit_amount
FROM deposits;
```

Question Three Determining how many customers made more than 1 deposit and either 1 purchase or withdrawal in a single month for each month
```SQL
WITH monthly_transactions AS (
	SELECT
		customer_id,
        MONTH(txn_date) AS mth,
		SUM(CASE WHEN txn_type = 'deposit' THEN 0 ELSE 1 END) AS deposit_count,
		SUM(CASE WHEN txn_type = 'purchase' THEN 0 ELSE 1 END) AS purchase_count,
		SUM(CASE WHEN txn_type = 'withdrawal' THEN 1 ELSE 0 END) AS withdrawal_count
    FROM customer_transactions
    GROUP BY customer_id, MONTH(txn_date)
)
SELECT
	mth,
    COUNT(DISTINCT customer_id) AS num_customers
FROM monthly_transactions
WHERE deposit_count >= 1 AND (purchase_count >= 1 OR withdrawal_count >= 1)
GROUP BY mth
ORDER BY mth;
```

Question Four: Determining closing balance for each customer at the end of each month.
```SQL
WITH monthly_balance AS (
	SELECT
		customer_id,
        txn_amount,
        MONTH(txn_date) AS txn_month,
        SUM(
			CASE
				WHEN txn_type = 'deposit' THEN txn_amount
                ELSE -txn_amount
			END
		) AS net_transaction_amount
	FROM customer_transactions
    GROUP BY customer_id, MONTH(txn_date), txn_amount
    ORDER BY customer_id
)
SELECT
	customer_id,
    txn_month, 
    net_transaction_amount,
    SUM(net_transaction_amount) OVER (PARTITION BY customer_id
    ORDER BY txn_month ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS closing_balance
FROM monthly_balance;
```

Question Five: Calculating the percentage of customers who increased their closing balance by at least 5%.
```SQL
WITH monthly_summary AS (
	SELECT 
		customer_id,
		MONTH(txn_date) AS txn_month,
        SUM(CASE WHEN txn_type = 'deposit' THEN txn_amount ELSE 0 END) AS total_deposit,
		SUM(CASE WHEN txn_type = 'purchase' THEN -txn_amount ELSE 0 END) AS total_purchase,
		SUM(CASE WHEN txn_type = 'withdrawal' THEN -txn_amount ELSE 0 END) AS total_withdrawal
	FROM customer_transactions
	GROUP BY customer_id, MONTH(txn_date)
), monthly_balance AS (
	SELECT
		customer_id,
        txn_month,
        (total_deposit + total_purchase + total_withdrawal) AS balance
	FROM monthly_summary
), key_nums AS (
	SELECT
		customer_id,
        txn_month, balance,
        FIRST_VALUE(balance) OVER (PARTITION BY customer_id ORDER BY customer_id) AS opening_balance,
        LAST_VALUE(balance) OVER (PARTITION BY customer_id ORDER BY customer_id) AS closing_balance
	FROM monthly_balance
), growth_rates as (
	SELECT *,
		((closing_balance - opening_balance) * 100 / ABS(opening_balance)) as growing_rate
	FROM key_nums
	WHERE ((closing_balance - opening_balance) * 100 / ABS(opening_balance)) >= 5 AND closing_balance > opening_balance
)
SELECT 
	CAST(COUNT(DISTINCT(customer_id)) AS FLOAT)*100
	/
	(SELECT CAST(COUNT(DISTINCT(customer_id)) AS FLOAT) FROM customer_transactions) AS Percent_Customer
FROM growth_rates
```
