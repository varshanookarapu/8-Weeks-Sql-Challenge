# SQL Case Study 2: Pizza Runner  

## D.Pricing and Ratings

**Question 1:** If a Meat Lovers pizza costs $12 and Vegetarian costs $10 and there were no charges for changes - how much money has Pizza Runner made so far if there are no delivery fees?

---

## SQL Code

```sql

-- Clean runner_orders table cancellation column - make sure the condition includes trimming white spaces ,checking for nulls and Nan.

UPDATE runner_orders 
SET cancellation = null 
WHERe cancellation = 'null' OR  TRIM(cancellation) = '' OR 
cancellation = 'NaN' ;

WITH pizza_runner_costs AS
(
SELECT  co.pizza_id,pizza_name,count(co.pizza_id) as pizzas_ordered,
CASE WHEN co.pizza_id = 1 THEN count(co.pizza_id)*12
ELSE count(co.pizza_id)*10
END AS total_pizza_cost 
FROM customer_orders co 
LEFT JOIN runner_orders ro ON co.order_id = ro.order_id
LEFT JOIN pizza_names pn  ON co.pizza_id = pn.pizza_id
WHERE cancellation IS NULL 
GROUP BY co.pizza_id,pizza_name
ORDER BY co.pizza_id 

) 

SELECT SUM(total_pizza_cost) AS money_pizza_runner_made FROM pizza_runner_costs

```
pizza_runner_costs CTE TABLE

| pizza_id | pizza_name | pizzas_ordered | total_pizza_cost |
| -------- | ---------- | -------------- | ---------------- |
| 1        | Meatlovers | 9              | 108              |
| 2        | Vegetarian | 3              | 30               |

---


| money_pizza_runner_made |
| ----------------------- |
| 138                     |

---

**Question 2:** What if there was an additional $1 charge for any pizza extras? Add cheese is $1 extra

---

## SQL Code

```sql
UPDATE runner_orders 
SET cancellation = null 
WHERe cancellation = 'null' OR  TRIM(cancellation) = '' OR 
cancellation = 'NaN' ;

UPDATE customer_orders
SET extras = null 
WHERe extras = 'null' OR  TRIM(extras) = '' OR 
extras = 'NaN' ;

-- used this function cardinality(string_to_array(extras,',')) to count the number of extras and then multiplied with the charge of extra 
WITH pizza_runner_costs AS
(
SELECT  co.pizza_id,pizza_name,count(co.pizza_id) as pizzas_ordered,
CASE WHEN  cardinality(string_to_array(extras,',')) IS NULL THEN  0 
ELSE cardinality(string_to_array(extras,',')) * 1 
END AS  extras_charge,
CASE WHEN co.pizza_id = 1 THEN count(co.pizza_id)*12
ELSE count(co.pizza_id)*10
END AS total_pizza_cost 
FROM customer_orders co 
LEFT JOIN runner_orders ro ON co.order_id = ro.order_id
LEFT JOIN pizza_names pn  ON co.pizza_id = pn.pizza_id
WHERE cancellation IS NULL 
GROUP BY co.pizza_id,pizza_name, extras
ORDER BY co.pizza_id 
)

SELECT  SUM(total_pizza_cost)+SUM(extras_charge) AS money_pizza_runner_made FROM pizza_runner_costs
```

---
| money_pizza_runner_made |
| ----------------------- |
| 142                     |

---


**Question 5:** If a Meat Lovers pizza was $12 and Vegetarian $10 fixed prices with no cost for extras and each runner is paid $0.30 per kilometre traveled - how much money does Pizza Runner have left over after these deliveries?

---

## SQL Code

```sql
UPDATE runner_orders 
SET cancellation = null 
WHERe cancellation = 'null' OR  TRIM(cancellation) = '' OR 
cancellation = 'NaN' ;

WITH prc AS
(
SELECT co.pizza_id,pizza_name , COUNT(co.pizza_id) as pizzas_ordered ,
CASE WHEN co.pizza_id=1 THEN COUNT(co.pizza_id) * 12
ELSE COUNT(co.pizza_id) *10 END  AS 
pizza_charge
FROM customer_orders co LEFT JOIN runner_orders ro
on co.order_id = ro.order_id
LEFT JOIN pizza_names pn ON
co.pizza_id = pn.pizza_id
WHERE cancellation IS NULL
GROUP BY co.pizza_id,pn.pizza_name
ORDER BY co.pizza_id 
) ,

rc AS 
(
SELECT order_id, NULLIF(REGEXP_REPLACE(distance ,'[^0-9.]','','g'), ' ' ) :: DECIMAL  AS distance_km
from runner_orders WHERE cancellation IS NULL
ORDER BY order_id
) 



SELECT 
(SELECT SUM(pizza_charge)  FROM prc ) AS total_pizza_cost,
(SELECT SUM(distance_km) *0.30 FROM rc ) as runner_charge ,
(SELECT SUM(pizza_charge)  FROM prc ) - (SELECT SUM(distance_km) *0.30 FROM rc ) as profit_pizza_runner_made

```
prc -cte
<img width="1686" height="190" alt="image" src="https://github.com/user-attachments/assets/39aa7117-7f22-4fb7-adc9-ca6f4c3101bc" />

rc - cte
<img width="1349" height="451" alt="image" src="https://github.com/user-attachments/assets/dc8b8c57-b482-4a08-946a-af4997fbe62a" />

| total_pizza_cost | runner_charge | profit_pizza_runner_made |
| ---------------- | ------------- | ------------------------ |
| 138              | 43.560        | 94.440                   |

---

