# Case Study One: Danny's Diner
### Creating Dataframes and Tables
First, the following code was used to create sample databases and frames to be used to complete the rest of the case study.

``` sql
CREATE SCHEMA dannys_diner;
SET search_path = dannys_diner;

USE dannys_diner;

CREATE TABLE sales (
  customer_id VARCHAR(1),
  order_date DATE,
  product_id INTEGER
);

INSERT INTO sales
  (customer_id, order_date, product_id)
VALUES
  ('A', '2021-01-01', '1'),
  ('A', '2021-01-01', '2'),
  ('A', '2021-01-07', '2'),
  ('A', '2021-01-10', '3'),
  ('A', '2021-01-11', '3'),
  ('A', '2021-01-11', '3'),
  ('B', '2021-01-01', '2'),
  ('B', '2021-01-02', '2'),
  ('B', '2021-01-04', '1'),
  ('B', '2021-01-11', '1'),
  ('B', '2021-01-16', '3'),
  ('B', '2021-02-01', '3'),
  ('C', '2021-01-01', '3'),
  ('C', '2021-01-01', '3'),
  ('C', '2021-01-07', '3');
 

CREATE TABLE menu (
  product_id INTEGER,
  product_name VARCHAR(5),
  price INTEGER
);

INSERT INTO menu
  (product_id, product_name, price)
VALUES
  ('1', 'sushi', '10'),
  ('2', 'curry', '15'),
  ('3', 'ramen', '12');
  

CREATE TABLE members (
  customer_id VARCHAR(1),
  join_date DATE
);

INSERT INTO members
  (customer_id, join_date)
VALUES
  ('A', '2021-01-07'),
  ('B', '2021-01-09');
```

### Question One: Determining the total amount spent by each customer
```
SELECT
	sales.customer_id,
    SUM(menu.price) AS total_sales
FROM sales
JOIN menu
	ON sales.product_id = menu.product_id
GROUP BY sales.customer_id
ORDER BY sales.customer_id ASC;
```

### Question Two: Determining how many times each customer has visited the restaurant
```
SELECT
	customer_id,
    COUNT(order_date) AS total_visits
FROM sales
GROUP BY customer_id;
```

### Qustion Three: Determining the first item purchased by each customer
```
WITH ordered_sales AS (
	SELECT
		sales.customer_id,
        sales.order_date,
        menu.product_name,
        DENSE_RANK() OVER (
			PARTITION BY sales.customer_id
            ORDER BY sales.order_date) AS menu_rank
	FROM sales
    JOIN menu
		ON sales.product_id = menu.product_id
)
SELECT customer_id, product_name
FROM ordered_sales
WHERE menu_rank = 1
GROUP BY customer_id, product_name;
```

### Question Four: Determining the most popular item on the menu for all customers and how many times it has been bought by all customers
```
SELECT
	menu.product_name,
    COUNT(sales.product_id) AS num_purchases
FROM sales
JOIN menu
	ON sales.product_id = menu.product_id
GROUP BY menu.product_name
ORDER BY num_purchases DESC
LIMIT 1;
```

### Question Five: Determining the most popular item for each customer
```
WITH most_purchased AS (
	SELECT
		menu.product_name,
        sales.customer_id,
        COUNT(menu.product_id) AS order_count,
        DENSE_RANK() OVER (
			PARTITION BY sales.customer_id
            ORDER BY COUNT(sales.customer_id) DESC) AS purchase_count
    FROM menu
    JOIN sales
		ON menu.product_id = sales.product_id
	GROUP BY menu.product_name, sales.customer_id
)
SELECT
	customer_id,
    product_name,
    order_count
FROM most_purchased
WHERE purchase_count = 1;
```

### Question Six: Determining the item first purchased by a customer after they became a member
```
WITH joined_as_member AS (
	SELECT
		members.customer_id,
        sales.product_id,
        sales.order_date,
        ROW_NUMBER() OVER (
			PARTITION BY members.customer_id
            ORDER BY sales.order_date
        ) AS purchase_order
    FROM members
    JOIN sales
		ON members.customer_id = sales.customer_id
        AND sales.order_date > members.join_date
)
SELECT
	customer_id,
    joined_as_member.product_id,
    menu.product_name
FROM joined_as_member
JOIN menu
	ON joined_as_member.product_id = menu.product_id
WHERE purchase_order = 1
ORDER BY customer_id;
```

### Question Seven: Determining the last item purchased by a customer before they becamse a member
```
WITH joined_as_member AS (
	SELECT
		members.customer_id,
        sales.product_id,
        sales.order_date,
        ROW_NUMBER() OVER (
			PARTITION BY members.customer_id
            ORDER BY sales.order_date
        ) AS purchase_order
    FROM members
    JOIN sales
		ON members.customer_id = sales.customer_id
        AND sales.order_date < members.join_date
)
SELECT
	customer_id,
    joined_as_member.product_id,
    menu.product_name
FROM joined_as_member
JOIN menu
	ON joined_as_member.product_id = menu.product_id
WHERE purchase_order = 1
ORDER BY customer_id;
```

### Question Eight: Determining the total number of items and amount spent by each member before they bought a membership
```
SELECT
	sales.customer_id,
    COUNT(sales.product_id) AS total_items,
    SUM(menu.price) AS total_spent
FROM sales
JOIN members
	ON sales.customer_id = members.customer_id
    AND sales.order_date < members.join_date
JOIN menu
	ON sales.product_id = menu.product_id
GROUP BY sales.customer_id
ORDER BY sales.customer_id;
```

### Question Nine: Calculating the number of points each customer has if every $1 spent is equal to 10 points and sushi is worth double the points
```
WITH total_points AS (
	SELECT
		product_id,
        CASE
			WHEN product_id = 1 THEN price * 20
            ELSE price * 10
		END AS points
    FROM menu
)
SELECT
	sales.customer_id,
    SUM(total_points.points) AS	points
FROM sales
JOIN total_points
	ON sales.product_id = total_points.product_id
GROUP BY sales.customer_id
ORDER BY sales.customer_id;
```

### Question Ten: Calculating the number of points each member has in the month of January if the first week after buying a membership is worth double the points
```
WITH dates AS (
	SELECT
		customer_id,
        join_date,
		DATE_ADD(join_date, INTERVAL 6 DAY) AS valid_date, 
		DATE_ADD('2021-01-31', INTERVAL 0 DAY) AS last_date
    FROM members
)
SELECT
	sales.customer_id,
	SUM(
		CASE
			WHEN menu.product_name = 'sushi' THEN 2 * 10 * menu.price
            WHEN sales.order_date BETWEEN dates.join_date AND dates.valid_date THEN 2 * 10 * menu.price
            ELSE 10 * menu.price
            END
    ) AS points
FROM sales
JOIN dates
	ON sales.customer_id = dates.customer_id
    AND dates.join_date <= sales.order_date
    AND sales.order_date <= dates.last_date
JOIN menu
	ON sales.product_id = menu.product_id
GROUP BY sales.customer_id
ORDER BY sales.customer_id;
```

### Additional Question: Joining multiple tables and ranking products based on order purchased once having a membership
```
WITH customers_data AS (
	SELECT
		sales.customer_id,
		order_date,
		product_name,
		price,
		CASE
			WHEN members.join_date > sales.order_date THEN 'N'
			WHEN members.join_date <= sales.order_date THEN 'Y'
			ELSE 'N'
		END AS member
	FROM sales
	LEFT JOIN members
		ON sales.customer_id = members.customer_id
	JOIN menu
		ON sales.product_id = menu.product_id
	ORDER BY sales.customer_id ASC, sales.order_date ASC, menu.price DESC
)
SELECT
	*,
    CASE
		WHEN member = 'N' THEN NULL
        ELSE RANK () OVER (
			PARTITION BY customer_id, member
            ORDER BY order_date
        )
	END AS ranking
FROM customers_data;
```
