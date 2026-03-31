# SQL Case Study 2: Pizza Runner

https://8weeksqlchallenge.com/case-study-2/ 

## Data Cleaning Customer Orders Table 

```sql

--Cleaning Customer Orders Table Exclustions and Extras Columns - updating blank vlaues to nulls
-- Checking for null string value, blank values and actual null and replacing them all with nulls

UPDATE customer_orders
SET exclusions = null
WHERE exclusions IS null OR TRIM(exclusions) = '' OR exclusions = 'null';
   

UPDATE customer_orders
SET extras = null
WHERe extras IS NULL  OR TRIM(extras)='' OR extras  = 'null';

-- Checking Duplicates 
SELECT order_id,customer_id,pizza_id,exclusions,extras,order_time, COUNT(*)
FROM customer_orders
GROUP BY order_id,customer_id,order_time,pizza_id,exclusions,extras
HAVING COUNT(*) > 1;


--Deleting the duplicate records
--ctid → a unique physical identifier for each row  in PostgresSQL

DELETE FROM customer_orders 
WHERE ctid NOT IN ( SELECT MIN(ctid) FROM customer_orders GROUP BY order_id,customer_id,order_time,pizza_id,exclusions,extras ) ;
            
-- Final cleaned customer_orders Table
SELECT * FROM customer_orders ORDER BY order_id;

```

---

## Data Cleaning Runner Orders Table

```sql
--cancellation column

UPDATE runner_orders 
SET cancellation = null 
WHERe cancellation = 'null' OR  TRIM(cancellation) = '' OR 
cancellation = 'NaN' ;

--pickup_time
UPDATE runner_orders 
SET pickup_time = null 
WHERe pickup_time = 'null' OR  TRIM(pickup_time) = '' OR 
pickup_time = 'NaN' ;

--distance
UPDATE runner_orders 
SET distance = null 
WHERe distance = 'null' OR  TRIM(distance) = '' OR 
distance = 'NaN' ;
--duration 
UPDATE runner_orders 
SET duration = null 
WHERe duration = 'null' OR  TRIM(duration) = '' OR 
duration = 'NaN' ;

-- removing km and mins from distance and duration columns

UPDATE runner_orders
SET distance = REGEXP_REPLACE( distance, '[^0-9.]', '', 'g' ) ;


UPDATE runner_orders
SET duration = REGEXP_REPLACE (duration , '[^0-9.]','','g');


-- Changing the data types of pickup_time, distance and duration columns

ALTER TABLE runner_orders
ALTER COLUMN distance TYPE decimal USING distance :: decimal; 

ALTER TABLE runner_orders
ALTER duration TYPE integer USING duration ::integer;

ALTER TABLE runner_orders
ALTER pickup_time TYPE timestamp USING  pickup_time :: timestamp; 


SELECT * FROM runner_orders ORDER BY order_id

```


## A. Pizza Metrics

**Question 1:** How many pizzas were ordered?

---

## SQL Code

```sql
SELECT COUNT(order_id) AS pizzas_ordered
FROM customer_orders;
```

| pizzas_ordered |
| -------------- |
| 13             |

---


**Question 2:** How many unique customer orders were made?

---

## SQL Code
```sql
SELECT COUNT(DISTINCT order_id) AS unique_customer_orders
FROM customer_orders;
```

| unique_customer_orders |
| ---------------------- |
| 10                     |

---

[View on DB Fiddle](https://www.db-fiddle.com/f/7VcQKQwsS3CTkGRFG7vu98/65)

**Question 3:** How many successful orders were delivered by each runner?

---
## SQL Code

```sql
SELECT runner_id, COUNT(pickup_time) as successful_orders FROM runner_orders WHERE 
pickup_time IS NOT NULL AND cancellation IS null 
GROUP BY runner_id

```
<img width="1681" height="222" alt="image" src="https://github.com/user-attachments/assets/65359b21-dd15-4c6c-a085-3b60b35bfac6" />


**Question 4:** How many of each type of pizza was delivered?

---

## SQL Code

```sql
WITH customer_orders_runner_orders AS 
(
SELECT co.order_id,customer_id,pn.pizza_id,pizza_name, exclusions,extras,order_time,runner_id,pickup_time,distance,duration,cancellation FROM customer_orders co LEFT JOIN runner_orders ro ON
co.order_id = ro.order_id 
LEFT JOIN pizza_names pn ON
co.pizza_id = pn.pizza_id
ORDER BY co.order_id
)
SELECT pizza_id, pizza_name, COUNT(pizza_id) AS total_pizzas_delivered
FROM customer_orders_runner_orders
WHERE cancellation IS NULL
GROUP BY pizza_id,pizza_name;
```
<img width="1671" height="201" alt="image" src="https://github.com/user-attachments/assets/5f5230ef-ae46-42cd-9f21-a37e401a82f9" />


**Question 5**: How many Vegetarian and Meatlovers were ordered by each customer? - Join customer_orders and pizza_names tables

---

## SQL Code

```sql
WITH customer_orders_pizza_names AS (
    SELECT order_id,
           customer_id,
           co.pizza_id,
           exclusions,
           extras,
           order_time,
           pizza_name
    FROM customer_orders co
    LEFT JOIN pizza_names pn ON co.pizza_id = pn.pizza_id
)
SELECT customer_id,
       pizza_name,
       COUNT(pizza_id) AS pizzas_ordered
FROM customer_orders_pizza_names
GROUP BY customer_id, pizza_id, pizza_name
ORDER BY customer_id;
```
<img width="1908" height="454" alt="image" src="https://github.com/user-attachments/assets/e738de11-20df-4f47-9969-8effdb8a3fa9" />


**Question 6** : What was the maximum number of pizzas delivered in a single order?
---

## SQL Code

```sql
WITH customer_orders_runner_orders AS 
(
SELECT co.order_id,customer_id,pizza_id,exclusions,extras,order_time,runner_id,pickup_time,distance,duration,cancellation FROM customer_orders co LEFT JOIN runner_orders ro ON
co.order_id = ro.order_id 
ORDER BY co.order_id
)


SELECT MAX(pizzas_delivered) as max_pizzas_delivered_in_single_order
FROM
(
SELECT order_id , COUNT(pizza_id) as pizzas_delivered FROM  customer_orders_runner_orders 
WHERE cancellation IS NULL
GROUP BY order_id
ORDER BY order_id
) as pizzas_per_order

```
<img width="995" height="138" alt="image" src="https://github.com/user-attachments/assets/34554ac4-2f51-4e37-a4d0-174ccfccd7fc" />


**Question 7** : For each customer, how many delivered pizzas had at least 1 change and how many had no changes?
---

## SQL Code

```sql
WITH customer_orders_runner_orders AS 
(
SELECT co.order_id,customer_id,pizza_id,exclusions,extras,order_time,runner_id,pickup_time,distance,duration,cancellation FROM customer_orders co LEFT JOIN runner_orders ro ON
co.order_id = ro.order_id 
ORDER BY co.order_id
)

SELECT customer_id, count(pizza_id) as pizzas_delivered,
SUM(CASE WHEN   (exclusions IS NOT NULL OR  extras IS NOT NULL ) THEN  1 ELSE 0 END ) AS pizzas_with_atleast_one_change,
SUM(CASE WHEN   (exclusions IS  NULL AND extras IS NULL ) THEN  1 ELSE 0 END ) AS pizzas_with_no_change
FROM customer_orders_runner_orders 
WHERE cancellation IS NULL 
GROUP BY customer_id
ORDER BY customer_id

```
<img width="1670" height="325" alt="image" src="https://github.com/user-attachments/assets/1f225396-cd57-42ce-a0df-0d5e6a3010a2" />


**Question 8**: How many pizzas were delivered that had both exclusions and extras?

---

## SQL Code

```sql
WITH customer_orders_runner_orders AS 
(
SELECT co.order_id,customer_id,pizza_id,exclusions,extras,order_time,runner_id,pickup_time,distance,duration,cancellation FROM customer_orders co LEFT JOIN runner_orders ro ON
co.order_id = ro.order_id 
ORDER BY co.order_id
)

SELECT COUNT(order_id) as  pizzas_delivered_with_exclusions_and_extras
FROM  customer_orders_runner_orders WHERE exclusions IS NOT NULL AND extras IS NOT NULL AND pickup_time IS NOT NULL
```
<img width="1670" height="126" alt="image" src="https://github.com/user-attachments/assets/402b7672-c6a8-4a95-9d88-49c8338d0095" />

**Question 9** : What was the total volume of pizzas ordered for each hour of the day?
---

## SQL Code

```sql
SELECT EXTRACT(HOUR FROM order_time) as hour, count(pizza_id) as volume_of_pizzas_ordered_for_each_hour_of_day FROM customer_orders
GROUP BY hour
ORDER BY hour

```
<img width="1389" height="366" alt="image" src="https://github.com/user-attachments/assets/fd581da2-1973-4c47-a521-d0542226a0f7" />  


**Question 10**: What was the volume of orders for each day of the week?  - could be ambiguous but we need to count all the orders based on the day of the week ( like all orders on monday irrespective of order_date ) 

---

## SQL Code

```sql

--TO_CHAR() is a conversion/formatting function that converts a date, timestamp, or number into a string

SELECT  TO_CHAR(order_time ,'Day' )   as day_of_week, COUNT( order_id) as volume_of_orders FROM customer_orders 
GROUP BY  TO_CHAR (order_time , 'Day')
ORDER BY TO_CHAR (order_time , 'Day') DESC
```

<img width="1299" height="286" alt="image" src="https://github.com/user-attachments/assets/91b5ceb5-09e0-4649-9d3a-5d2f8acb3e77" />


