# Case Study Two: Pizza Runner
### Creating Tables
```SQL
CREATE SCHEMA pizza_runner;
USE pizza_runner;

DROP TABLE IF EXISTS runners;
CREATE TABLE runners (
  runner_id INTEGER,
  registration_date DATE
);
INSERT INTO runners
  (runner_id, registration_date)
VALUES
  (1, '2021-01-01'),
  (2, '2021-01-03'),
  (3, '2021-01-08'),
  (4, '2021-01-15');


DROP TABLE IF EXISTS customer_orders;
CREATE TABLE customer_orders (
  order_id INTEGER,
  customer_id INTEGER,
  pizza_id INTEGER,
  exclusions VARCHAR(4),
  extras VARCHAR(4),
  order_time TIMESTAMP
);

INSERT INTO customer_orders
  (order_id, customer_id, pizza_id, exclusions, extras, order_time)
VALUES
  ('1', '101', '1', '', '', '2020-01-01 18:05:02'),
  ('2', '101', '1', '', '', '2020-01-01 19:00:52'),
  ('3', '102', '1', '', '', '2020-01-02 23:51:23'),
  ('3', '102', '2', '', NULL, '2020-01-02 23:51:23'),
  ('4', '103', '1', '4', '', '2020-01-04 13:23:46'),
  ('4', '103', '1', '4', '', '2020-01-04 13:23:46'),
  ('4', '103', '2', '4', '', '2020-01-04 13:23:46'),
  ('5', '104', '1', 'null', '1', '2020-01-08 21:00:29'),
  ('6', '101', '2', 'null', 'null', '2020-01-08 21:03:13'),
  ('7', '105', '2', 'null', '1', '2020-01-08 21:20:29'),
  ('8', '102', '1', 'null', 'null', '2020-01-09 23:54:33'),
  ('9', '103', '1', '4', '1, 5', '2020-01-10 11:22:59'),
  ('10', '104', '1', 'null', 'null', '2020-01-11 18:34:49'),
  ('10', '104', '1', '2, 6', '1, 4', '2020-01-11 18:34:49');


DROP TABLE IF EXISTS runner_orders;
CREATE TABLE runner_orders (
  order_id INTEGER,
  runner_id INTEGER,
  pickup_time VARCHAR(19),
  distance VARCHAR(7),
  duration VARCHAR(10),
  cancellation VARCHAR(23)
);

INSERT INTO runner_orders
  (order_id, runner_id, pickup_time, distance, duration, cancellation)
VALUES
  ('1', '1', '2020-01-01 18:15:34', '20km', '32 minutes', ''),
  ('2', '1', '2020-01-01 19:10:54', '20km', '27 minutes', ''),
  ('3', '1', '2020-01-03 00:12:37', '13.4km', '20 mins', NULL),
  ('4', '2', '2020-01-04 13:53:03', '23.4', '40', NULL),
  ('5', '3', '2020-01-08 21:10:57', '10', '15', NULL),
  ('6', '3', 'null', 'null', 'null', 'Restaurant Cancellation'),
  ('7', '2', '2020-01-08 21:30:45', '25km', '25mins', 'null'),
  ('8', '2', '2020-01-10 00:15:02', '23.4 km', '15 minute', 'null'),
  ('9', '2', 'null', 'null', 'null', 'Customer Cancellation'),
  ('10', '1', '2020-01-11 18:50:20', '10km', '10minutes', 'null');


DROP TABLE IF EXISTS pizza_names;
CREATE TABLE pizza_names (
  pizza_id INTEGER,
  pizza_name TEXT
);
INSERT INTO pizza_names
  (pizza_id, pizza_name)
VALUES
  (1, 'Meatlovers'),
  (2, 'Vegetarian');


DROP TABLE IF EXISTS pizza_recipes;
CREATE TABLE pizza_recipes (
  pizza_id INTEGER,
  toppings TEXT
);
INSERT INTO pizza_recipes
  (pizza_id, toppings)
VALUES
  (1, '1, 2, 3, 4, 5, 6, 8, 10'),
  (2, '4, 6, 7, 9, 11, 12');

DROP TABLE IF EXISTS pizza_toppings;
CREATE TABLE pizza_toppings (
  topping_id INTEGER,
  topping_name TEXT
);
INSERT INTO pizza_toppings
  (topping_id, topping_name)
VALUES
  (1, 'Bacon'),
  (2, 'BBQ Sauce'),
  (3, 'Beef'),
  (4, 'Cheese'),
  (5, 'Chicken'),
  (6, 'Mushrooms'),
  (7, 'Onions'),
  (8, 'Pepperoni'),
  (9, 'Peppers'),
  (10, 'Salami'),
  (11, 'Tomatoes'),
  (12, 'Tomato Sauce');
```

### Cleaning Data
```SQL
-- Data Cleaning: runner_orders
CREATE TABLE runner_orders_temp AS
SELECT
	order_id,
    runner_id,
    CASE
		WHEN pickup_time IS NULL OR pickup_time LIKE 'null' THEN NULL
        ELSE pickup_time
	END AS pickup_time,
    CASE
		WHEN distance IS NULL OR distance LIKE 'null' THEN NULL
		WHEN distance LIKE '%km' THEN TRIM('km' FROM distance)
        ELSE distance
	END AS distance,
    CASE
		WHEN duration IS NULL OR duration LIKE 'null' THEN NULL
        WHEN duration LIKE '%mins' THEN TRIM('mins' FROM duration)
        WHEN duration LIKE '%minute' THEN TRIM('minute' FROM duration)
        WHEN duration LIKE '%minutes' THEN TRIM('minutes' FROM duration)
        ELSE duration
	END AS duration,
    CASE
		WHEN cancellation IS NULL OR cancellation LIKE 'null' THEN ''
        ELSE cancellation
	END AS cancellation
FROM runner_orders;

ALTER TABLE runner_orders_temp
MODIFY COLUMN pickup_time DATETIME,
MODIFY COLUMN distance FLOAT,
MODIFY COLUMN duration INT;

-- Data Cleaning: customer_orders
CREATE TABLE customer_orders_temp AS
SELECT
	order_id,
    customer_id,
    pizza_id,
    CASE
		WHEN exclusions IS NULL OR exclusions LIKE 'null' THEN ''
        ELSE exclusions
	END AS exclusions,
    CASE
		WHEN extras IS NULL OR extras LIKE 'null' THEN ''
        ELSE extras
	END AS extras
FROM customer_orders;
  
```

## Part A: Pizza Metrics
### Question One: Determining how many pizzas were ordered
```SQL
SELECT
	COUNT(order_id) AS pizza_order_count
FROM customer_orders_temp;
```

### Question Two: Determining how many unique customer orders were made
```SQL
SELECT
	COUNT(DISTINCT order_id) AS total_unique_orders
FROM customer_orders_temp;  
```

### Question Three: Determining how many successful orders were delivered by each runner
```SQL
SELECT
	runner_id,
    COUNT(order_id) AS successful_orders
FROM runner_orders_temp
WHERE distance IS NOT NULL
GROUP BY runner_id;
```

### Question Four: Determining how many pizzas of each type were delievered
```SQL
SELECT
	p.pizza_name,
    COUNT(c.pizza_id) AS delivered_pizzas
FROM customer_orders_temp AS c
JOIN runner_orders_temp As r
	ON c.order_id = r.order_id
JOIN pizza_names AS p
	ON c.pizza_id = p.pizza_id
WHERE r.distance IS NOT NULL
GROUP BY p.pizza_name;
```

### Question Five: Determining how many pizzas of each type were ordered by each customer
```SQL
SELECT
	c.customer_id,
    p.pizza_name,
    COUNT(p.pizza_name) AS order_count
FROM customer_orders_temp AS c
JOIN pizza_names AS p
	ON c.pizza_id = p.pizza_id
GROUP BY c.customer_id, p.pizza_name
ORDER BY c.customer_id;
```

### Question Six: Determining the max number of pizzas delivered in one order
```SQL
WITH pizza_order_counts AS
(
	SELECT
		c.order_id,
        COUNT(c.pizza_id) AS num_pizzas
    FROM customer_orders_temp AS c
    JOIN runner_orders_temp AS r
		ON c.order_id = r.order_id
	WHERE r.distance IS NOT NULL
	GROUP BY c.order_id
)
SELECT *
FROM pizza_order_counts;
```

### Question Seven: Determining how many pizzas had one change and how many had no changes
```SQL
SELECT
	c.customer_id,
    SUM(
		CASE
			WHEN c.exclusions <> '' OR c.extras <> '' THEN 1
			ELSE 0
		END) AS made_changes,
	SUM(
		CASE
			WHEN c.exclusions = '' AND c.extras = '' THEN 1
			ELSE 0
		END) AS no_changes
FROM customer_orders_temp AS c
JOIN runner_orders_temp AS r
	ON c.order_id = r.order_id
WHERE r.distance IS NOT NULL
GROUP BY c.customer_id
ORDER BY c.customer_id;
```

### Question Eight: Determining how many delivered pizzas had both exclusions and extras
```SQL
SELECT
	COUNT(c.pizza_id) AS exclusions_and_extras
FROM customer_orders_temp AS c
JOIN runner_orders_temp AS r
	ON c.order_id = r.order_id
WHERE r.distance IS NOT NULL
	AND c.exclusions <> ''
    AND c.extras <> '';
```

### Question Nine: Calculating total volume of pizzas ordered each hour of the day
```SQL
SELECT
	HOUR(order_time) AS hour_of_day,
    COUNT(order_id) AS num_pizzas
FROM customer_orders AS c
GROUP BY HOUR(order_time)
ORDER by hour_of_day;
```

### Question Ten: Calculating total volume of pizzas ordered each day of the week
```SQL
SELECT
	DAYNAME(DATE_ADD(order_time, INTERVAL 2 DAY)) AS day_of_week,
    COUNT(order_id) AS num_pizzas
FROM customer_orders AS c
GROUP BY DAYNAME(DATE_ADD(order_time, INTERVAL 2 DAY))
ORDER BY FIELD(day_of_week, 
               'Monday', 'Tuesday', 'Wednesday', 'Thursday', 'Friday', 'Saturday', 'Sunday');
```

## Part B: Runner and Customer Experience
### Question One: Determining how many runners signed up for each one week period
```SQL
SELECT
	FLOOR(DATEDIFF(registration_date, '2021-01-01') / 7) + 1 AS week_number,
    COUNT(runner_id) AS num_runners
FROM
    runners
GROUP BY
    week_number
ORDER BY
    week_number;
```

### Question Two: Calculating average time for each runner to arrive to pickup the order
```SQL
WITH time_taken AS
(
	SELECT
		c.order_id,
        c.order_time,
        r.pickup_time,
        TIMESTAMPDIFF(MINUTE, c.order_time, r.pickup_time) AS pickup_minutes
    FROM customer_orders AS c
    JOIN runner_orders_temp AS r
		ON c.order_id = r.order_id
	WHERE r.distance IS NOT NULL
    GROUP BY c.order_id, c.order_time, r.pickup_time
)
SELECT
	AVG(pickup_minutes) AS avg_pickup_time_min
FROM time_taken;
```

### Question Three: Looking for relationship between time to prepare order and number of pizzas ordered
```SQL
WITH prepare_time AS
(
	SELECT
		c.order_id,
        COUNT(c.order_id) AS num_pizzas,
        c.order_time,
        r.pickup_time,
        TIMESTAMPDIFF(MINUTE, c.order_time, r.pickup_time) AS prep_minutes
    FROM customer_orders AS c
    JOIN runner_orders_temp AS r
		ON c.order_id = r.order_id
	WHERE r.distance IS NOT NULL
    GROUP BY c.order_id, c.order_time, r.pickup_time
)
SELECT
	num_pizzas,
    AVG(prep_minutes) AS avg_prep_minutes
FROM prepare_time
GROUP BY num_pizzas;
  
```

### Question Four: Calculating average distance travelled for each order
```SQL
SELECT
    c.customer_id,
    AVG(r.distance) AS avg_distance_km
FROM customer_orders_temp AS c
JOIN runner_orders_temp AS r
	ON c.order_id = r.order_id
WHERE r.distance IS NOT NULL
GROUP BY c.customer_id;
```

### Question Five: Calculating difference between longest and shortest delivery times
```SQL
SELECT
	MAX(duration) - MIN(duration) AS delivery_time_diff
FROM runner_orders_temp
WHERE duration IS NOT NULL;
```

### Question Six: Determining average speed for each delivery
```SQL
SELECT
	r.runner_id,
    c.customer_id,
    r.order_id,
    r.distance,
	(r.duration / 60) AS duration_h,
    ROUND((r.distance / r.duration * 60), 2) AS avg_speed
FROM runner_orders_temp AS r
JOIN customer_orders_temp AS c
	ON r.order_id = c.order_id
WHERE r.distance IS NOT NULL
GROUP BY r.runner_id, c.customer_id, r.order_id, r.distance, r.duration
ORDER BY r.order_id;
```

### Question Seven: Calculating successful delivrey percentage for each runner
```SQL
SELECT
	runner_id,
    ROUND(100 * SUM(
		CASE
			WHEN distance IS NULL THEN 0
            ELSE 1
		END) / COUNT(*), 0
    ) AS success_rate
FROM runner_orders_temp AS r
GROUP BY r.runner_id;
```

## Part C: Ingredient Optimization
### Question One: Determining most common/standard ingredient
```SQL
WITH toppings AS (
  SELECT
    pizza_id,
    TRIM(value) AS topping_id
  FROM pizza_recipes,
  JSON_TABLE(CONCAT('[', toppings, ']'), '$[*]' COLUMNS (value INT PATH '$')) AS t
)

SELECT
	pn.pizza_id,
    pn.pizza_name,
	toppings.topping_id,
    pt.topping_name
FROM toppings
JOIN pizza_toppings AS pt
	ON toppings.topping_id = pt.topping_id
JOIN pizza_names AS pn
	ON toppings.pizza_id = pn.pizza_id
ORDER BY pn.pizza_id, toppings.topping_id;
```

### Question Two: Determining most commonly added extra
```SQL
SELECT 
    pt.topping_name, 
    COUNT(*) AS extra_count
FROM customer_orders_temp AS c
JOIN pizza_toppings AS pt 
    ON FIND_IN_SET(pt.topping_id, c.extras) > 0
WHERE c.extras <> ''
GROUP BY pt.topping_name
ORDER BY extra_count DESC
LIMIT 1;
```

### Question Three: Determining most common exclusion
```SQL
SELECT 
    pt.topping_name, 
    COUNT(*) AS extra_count
FROM customer_orders_temp AS c
JOIN pizza_toppings AS pt 
    ON FIND_IN_SET(pt.topping_id, c.exclusions) > 0
WHERE c.exclusions <> ''
GROUP BY pt.topping_name
ORDER BY extra_count DESC
LIMIT 1;
```

### Question Four: Generating a record of order instructions (Ex. Meat Lovers - Exclude Cheese, Bacon - Extra Mushroom, Peppers)
```SQL
SELECT
	c.order_id,
    c.customer_id,
    CONCAT(
		pn.pizza_name,
        IF(c.exclusions IS NOT NULL AND c.exclusions != '', CONCAT(' - Exclude: ', GROUP_CONCAT(DISTINCT t.topping_name SEPARATOR ', ')), ''),
        IF(c.extras IS NOT NULL AND c.extras != '', CONCAT(' - Extra: ', GROUP_CONCAT(DISTINCT t_two.topping_name SEPARATOR ', ')), '')
    ) AS pizza_instructions
FROM customer_orders_temp AS c
JOIN pizza_names AS pn
	ON c.pizza_id = pn.pizza_id
JOIN pizza_recipes AS pr
	ON c.pizza_id = pr.pizza_id
LEFT JOIN pizza_toppings AS t
	ON FIND_IN_SET(t.topping_id, REPLACE(c.exclusions, ' ', '')) > 0
LEFT JOIN pizza_toppings AS t_two
	ON FIND_IN_SET(t_two.topping_id, REPLACE(c.extras, ' ', '')) > 0
GROUP BY c.order_id, c.customer_id, c.pizza_id, pn.pizza_name, pr.toppings, c.exclusions, c.extras;
```

### Question Five: Generating an alphabetically ordered ingredient list for each order and adding a 2x in front of relevant ingredients
```SQL
SELECT
    c.order_id,
    c.customer_id,
    pn.pizza_name,
CONCAT(
    GROUP_CONCAT(DISTINCT 
        CASE 
            -- If the topping is not in exclusions
            WHEN FIND_IN_SET(REPLACE(t.topping_id, ' ', ''), REPLACE(c.exclusions, ' ', '')) = 0 
            THEN 
                CASE
					WHEN FIND_IN_SET(REPLACE(t.topping_id, ' ', ''), REPLACE(c.extras, ' ', '')) > 0
                    THEN CONCAT('2x', t.topping_name)
                    ELSE t.topping_name
				END
            ELSE NULL
        END
    SEPARATOR ', ')
    ) AS pizza_ingredients
FROM customer_orders_temp AS c
JOIN pizza_names AS pn
	ON c.pizza_id = pn.pizza_id
JOIN pizza_recipes AS pr
	ON c.pizza_id = pr.pizza_id
LEFT JOIN pizza_toppings AS t
	ON FIND_IN_SET(t.topping_id, REPLACE(pr.toppings, ' ', '')) > 0
LEFT JOIN pizza_toppings AS t_two
	ON FIND_IN_SET(t_two.topping_id, REPLACE(c.exclusions, ' ', '')) > 0
LEFT JOIN pizza_toppings AS t_three
	ON FIND_IN_SET(t_three.topping_id, REPLACE(c.extras, ' ', '')) > 0
GROUP BY c.order_id, c.customer_id, c.pizza_id, pn.pizza_name, pr.toppings, c.exclusions, c.extras;
```

### Question Six: Determining how many times each ingredient was used
```SQL
WITH Ingredients_CTE AS (
    SELECT 
        SUBSTRING_INDEX(SUBSTRING_INDEX(pizza_ingredients, ', ', n.n), ', ', -1) AS ingredient
    FROM (
        SELECT 
            c.order_id,
            c.pizza_id,
            pn.pizza_name,
            pr.toppings,
            c.exclusions,
            c.extras,
            CONCAT(
                GROUP_CONCAT(DISTINCT 
                    CASE 
                        WHEN FIND_IN_SET(REPLACE(t.topping_id, ' ', ''), REPLACE(c.exclusions, ' ', '')) = 0 
                        THEN t.topping_name 
                        ELSE NULL 
                    END
                SEPARATOR ', '),
                IF(c.extras IS NOT NULL AND c.extras != '', CONCAT(', ', GROUP_CONCAT(DISTINCT t_three.topping_name SEPARATOR ', ')), '')
            ) AS pizza_ingredients
        FROM customer_orders_temp AS c
        JOIN runner_orders_temp AS r ON c.order_id = r.order_id
        JOIN pizza_names AS pn ON c.pizza_id = pn.pizza_id
        JOIN pizza_recipes AS pr ON c.pizza_id = pr.pizza_id
        LEFT JOIN pizza_toppings AS t ON FIND_IN_SET(t.topping_id, REPLACE(pr.toppings, ' ', '')) > 0
        LEFT JOIN pizza_toppings AS t_two ON FIND_IN_SET(t_two.topping_id, REPLACE(c.exclusions, ' ', '')) > 0
        LEFT JOIN pizza_toppings AS t_three ON FIND_IN_SET(t_three.topping_id, REPLACE(c.extras, ' ', '')) > 0
        WHERE r.distance > 0
        GROUP BY c.order_id, c.customer_id, c.pizza_id, pn.pizza_name, pr.toppings, c.exclusions, c.extras
    ) p
    JOIN (SELECT 1 n UNION SELECT 2 UNION SELECT 3 UNION SELECT 4 UNION SELECT 5 UNION SELECT 6 UNION SELECT 7 UNION SELECT 8 UNION SELECT 9 UNION SELECT 10) n 
        ON CHAR_LENGTH(p.pizza_ingredients) - CHAR_LENGTH(REPLACE(p.pizza_ingredients, ', ', '')) >= n.n - 1
)

SELECT 
    ingredient,
    COUNT(*) AS ingredient_count
FROM Ingredients_CTE
WHERE ingredient IS NOT NULL
GROUP BY ingredient
ORDER BY ingredient_count DESC;
```

## Part D: Pricing and Ratings
### Question One: Determining how much the company has made based on pizza prices and no delivery fees
```SQL
SELECT 
    SUM(CASE WHEN pizza_id = 1 THEN num_pizzas * 12 ELSE 0 END) +
    SUM(CASE WHEN pizza_id = 2 THEN num_pizzas * 10 ELSE 0 END) AS total_value
FROM (
    SELECT 
        c.pizza_id,
        COUNT(*) AS num_pizzas
    FROM customer_orders_temp AS c
    JOIN runner_orders_temp AS r ON r.order_id = c.order_id
    WHERE r.distance > 0
    GROUP BY c.pizza_id
) AS subquery;
```

### Question Two: Determining how much the company has made with a $1 change for pizza extras
```SQL
SELECT 
    SUM(CASE WHEN pizza_id = 1 THEN num_pizzas * 12 ELSE 0 END) +
    SUM(CASE WHEN pizza_id = 2 THEN num_pizzas * 10 ELSE 0 END) +
    SUM(extra_count) AS total_value
FROM (
    SELECT 
        c.pizza_id,
        COUNT(*) AS num_pizzas,
        SUM(CASE WHEN c.extras IS NOT NULL AND c.extras <> '' THEN 1 ELSE 0 END) AS extra_count
    FROM customer_orders_temp AS c
    JOIN runner_orders_temp AS r ON r.order_id = c.order_id
    WHERE r.distance > 0
    GROUP BY c.pizza_id
) AS subquery;
```

### Question Three: Creating table for ratings
```SQL
DROP TABLE IF EXISTS ratings;
CREATE TABLE ratings (
  order_id INTEGER,
  rating INTEGER CHECK (rating BETWEEN 1 AND 5)
);
INSERT INTO ratings
  (order_id, rating)
VALUES
  (1, 1),
  (2, 5),
  (3, 4),
  (4, 2),
  (5, 3),
  (6, 1),
  (7, 5),
  (8, 4),
  (9, 2),
  (10, 3);
```

### Question Four: Joining multiple tables into one to create summary table
```SQL
SELECT
	c.order_id,
	c.customer_id,
    r.runner_id,
    rt.rating,
    c.order_time,
    r.pickup_time,
    TIMESTAMPDIFF(MINUTE, c.order_time, r.pickup_time) AS pickup_minutes,
    r.duration,
    ROUND((r.distance / r.duration * 60), 2) AS avg_speed,
    COUNT(c.order_id) AS num_pizzas
FROM customer_orders AS c
JOIN runner_orders_temp AS r
	ON c.order_id = r.order_id
JOIN ratings AS rt
	ON c.order_id = rt.order_id
WHERE r.distance > 0
GROUP BY c.order_id, customer_id, r.runner_id, rt.rating, c.order_time, r.pickup_time, r.duration, r.distance;
```

### Question Five: Calculating money made after paying each runner for the deliveries
```SQL
SELECT 
	ROUND(
		SUM(CASE WHEN pizza_id = 1 THEN num_pizzas * 12 ELSE 0 END) +
		SUM(CASE WHEN pizza_id = 2 THEN num_pizzas * 10 ELSE 0 END) -
		SUM(extra_count), 
    2) AS total_money
FROM (
    SELECT 
        c.pizza_id,
        COUNT(*) AS num_pizzas,
        COALESCE(runners.extra_count, 0) AS extra_count
    FROM customer_orders_temp AS c
    JOIN (
        SELECT 
            r.order_id,
            SUM(r.distance) * 0.3 AS extra_count  -- Sum distance per order_id
        FROM runner_orders_temp AS r
        WHERE r.distance > 0
        GROUP BY r.order_id
    ) AS runners ON c.order_id = runners.order_id
    GROUP BY c.pizza_id, runners.extra_count
) AS subquery;
```

## Bonus - Editing pizza_recipes and pizza_toppings to add a Supreme Pizza to the menu
```SQL
INSERT INTO pizza_names
VALUES (3, 'Supreme');

SELECT *
FROM pizza_names;

INSERT INTO pizza_recipes
VALUES (3, '1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12');

SELECT *
FROM pizza_recipes;
```
