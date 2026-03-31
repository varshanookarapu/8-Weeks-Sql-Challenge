# Pizza Runner :  B.Runner and Customer Experience

## Prerequesite - Cleaning Runner Orders Table

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


SELECT * FROM runner_orders ORDER BY order_id ;

```

---

**Question 1:** How many runners signed up for each 1-week period? (i.e. week starts 2021-01-01)

---

## SQL Code

```sql
```



**Question 2:**  What was the average time in minutes it took for each runner to arrive at the Pizza Runner HQ to pickup the order?

---

## SQL Code

```sql

-- epoch is a fixed starting point in time.For PostgreSQL (and most systems), the epoch is: 1970-01-01 00:00:00 UTC .This is called the Unix epoch.
WITH customer_orders_runner_orders AS 
(
SELECT co.order_id,customer_id,pn.pizza_id,pizza_name, exclusions,extras,order_time,runner_id,pickup_time,distance,duration,cancellation FROM customer_orders co LEFT JOIN runner_orders ro ON
co.order_id = ro.order_id 
LEFT JOIN pizza_names pn ON
co.pizza_id = pn.pizza_id
ORDER BY co.order_id
)
SELECT runner_id, 
FLOOR (AVG(EXTRACT(EPOCH FROM (pickup_time - order_time) / 60) ))
AS time_taken_to_arrive_at_pizza_hq 
FROM customer_orders_runner_orders
GROUP BY runner_id
ORDER BY runner_id

```

<img width="1248" height="226" alt="image" src="https://github.com/user-attachments/assets/cadaa3a6-2f54-4b0f-952a-1984c0cf04bc" />


**Question 3:** Is there any relationship between the number of pizzas and how long the order takes to prepare?

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
),

-- Reasoning is that I checked for each order , what is the count of the pizzas delivered, and what is the maximum time taken to prepare the pizza.
--for instance if order 3 made two pizzas one pizza took 20 minute and the other took 25 minutes I chekecked the maximum pizza prep time,
--Then I proceeded to find the pizzas delivered and their average pizza prep time. 

pizza_delivery_time_stats as 
(
SELECT order_id,COUNT(order_id) as pizzas_delivered , FLOOR(MAX(EXTRACT(EPOCH FROM (pickup_time -order_time)/60))) as time_taken_to_prepare_pizza
FROM customer_orders_runner_orders
WHERE cancellation  IS  null
GROUP BY order_id 
ORDER BY order_id
)

SELECT pizzas_delivered, AVG(time_taken_to_prepare_pizza) as avg_time_taken_to_prepare_pizza 
FROM pizza_delivery_time_stats
GROUP BY pizzas_delivered
ORDER BY pizzas_delivered
```
<img width="960" height="223" alt="image" src="https://github.com/user-attachments/assets/f6e754a9-7ab4-4c2d-8ea0-91916805e97f" />



**Question 4:** What was the average distance travelled for each customer?

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

SELECT customer_id, ROUND(AVG(distance),2) as avg_distance_travelled FROM customer_orders_runner_orders
GROUP BY customer_id
ORDER BY customer_id
```

<img width="1161" height="320" alt="image" src="https://github.com/user-attachments/assets/451fcfb0-3d44-415e-9ed4-3de846bd6468" />

**Question 5:** What was the difference between the longest and shortest delivery times for all orders?
---

## SQL Code

```sql
SELECT MAX(duration)-MIN(duration) as difference_bw_longest_and_shortest_delivery_durations FROM runner_orders
```

<img width="436" height="132" alt="image" src="https://github.com/user-attachments/assets/5052e541-6503-46f0-9604-970e9e43fe31" />


**Question 6:** What was the average speed for each runner for each delivery and do you notice any trend for these values?
---

## SQL Code

```sql
SELECT 
	runner_id,
	order_id,
	ROUND(AVG(distance*1000/(duration*60)) *18/5,2)
    AS avg_speed_kmph
FROM runner_orders
WHERE cancellation IS NULL
GROUP BY runner_id,order_id
ORDER BY runner_id, order_id
```
### The trends I noticed are  that runner 2 has covered both the max- 93.60 kmph and min-35.10 kmph speeds among all the runners

<img width="1462" height="472" alt="image" src="https://github.com/user-attachments/assets/a6938771-e43d-44a4-87f0-664e922d229d" />

**Question 7:** What is the successful delivery percentage for each runner?
---

## SQL Code

```sql
WITH cte_delivery AS
(
SELECT runner_id,
SUM(CASE WHEN cancellation IS NULL THEN 1 ELSE 0 END) as successful_orders_delivered,
SUM(CASE WHEN cancellation IS NOT NULL THEN 1 ELSE 0 END) as orders_not_delivered,
COUNT(order_id) as all_orders
FROM runner_orders
GROUP BY runner_id
ORDER BY runner_id
)

SELECT runner_id, TRUNC((successful_orders_delivered/ CAST(all_orders AS DECIMAL))*100) as successful_delivery_percentage
FROM cte_delivery 
```

<img width="1044" height="235" alt="image" src="https://github.com/user-attachments/assets/9c97878a-cacd-4385-86d2-1bf55975d623" />


