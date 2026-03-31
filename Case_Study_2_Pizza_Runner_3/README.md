# Pizza Runner :  C . Ingredient Optimization

## Prequsite

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


**Question 1:** What are the standard ingredients for each pizza? - ( I was a bit confused by the question I initally assumed they were asking for the topping names but its the common ingredients used between two pizzas ) 

---

## SQL Code

```sql
WITH normalized_pizza_recipes AS
(
SELECT pizza_id, CAST (TRIM(topping) AS INTEGER) as topping_id
FROM pizza_recipes ,
  UNNEST(STRING_TO_ARRAY (toppings, ',')) AS  topping
) 


--Normalizing the Table

SELECT pizza_id,npr.topping_id,topping_name 
 FROM normalized_pizza_recipes npr 
LEFT JOIN pizza_toppings pt ON
 npr.topping_id = pt.topping_id
 ORDER BY pizza_id,topping_id;


-- De-normalizing the table
SELECT pizza_id, 
STRING_AGG(npr.topping_id :: TEXT , ',' ORDER BY npr.topping_id ) as topping_id ,
STRING_AGG(topping_name , ' , ' ORDER BY npr.topping_id) as topping_name
FROM normalized_pizza_recipes npr 
LEFT JOIN pizza_toppings pt ON
npr.topping_id = pt.topping_id
GROUP BY npr.pizza_id
ORDER BY npr.pizza_id

```
**Normalized Table** 

| pizza_id | topping_id | topping_name |
| -------- | ---------- | ------------ |
| 1        | 1          | Bacon        |
| 1        | 2          | BBQ Sauce    |
| 1        | 3          | Beef         |
| 1        | 4          | Cheese       |
| 1        | 5          | Chicken      |
| 1        | 6          | Mushrooms    |
| 1        | 8          | Pepperoni    |
| 1        | 10         | Salami       |
| 2        | 4          | Cheese       |
| 2        | 6          | Mushrooms    |
| 2        | 7          | Onions       |
| 2        | 9          | Peppers      |
| 2        | 11         | Tomatoes     |
| 2        | 12         | Tomato Sauce |

---
**Denormalized Table**

| pizza_id | topping_id       | topping_name                                                                 |
| -------- | ---------------- | ---------------------------------------------------------------------------- |
| 1        | 1,2,3,4,5,6,8,10 | Bacon , BBQ Sauce , Beef , Cheese , Chicken , Mushrooms , Pepperoni , Salami |
| 2        | 4,6,7,9,11,12    | Cheese , Mushrooms , Onions , Peppers , Tomatoes , Tomato Sauce              |

---

```sql
WITH pr AS 
(
SELECT pizza_id, CAST(TRIM(topping) AS INTEGER ) as topping_id
FROM pizza_recipes ,
UNNEST(STRING_TO_ARRAY(toppings, ',')) AS topping
)

SELECT pr.topping_id, topping_name  FROM
pr LEFT JOIN pizza_toppings pt ON
pr.topping_id = pt.topping_id 
WHERE pr.topping_id  IN (SELECT topping_id FROM pr GROUP BY 
  topping_id
  HAVING count(distinct pizza_id) > 1 )
GROUP BY pr.topping_id ,topping_name
ORDER BY topping_id
```

| topping_id | topping_name |
| ---------- | ------------ |
| 4          | Cheese       |
| 6          | Mushrooms    |

---





**Question 2:** What was the most commonly added extra?

---

## SQL Code

```sql
WITH normalized_pizza_recipes AS
(
SELECT pizza_id, CAST (TRIM(topping) AS INTEGER) as topping_id
FROM pizza_recipes ,
  UNNEST(STRING_TO_ARRAY (toppings, ',')) AS  topping
), 


npr_pt AS
(
SELECT pizza_id,npr.topping_id,topping_name 
FROM normalized_pizza_recipes npr 
LEFT JOIN pizza_toppings pt ON
npr.topping_id = pt.topping_id
ORDER BY pizza_id,topping_id
)
,

nco AS 
(  
SELECT order_id,customer_id,pizza_id, CAST(TRIM(extra) AS INTEGER) as extras
FROM customer_orders ,
UNNEST(STRING_TO_ARRAY(extras,',')) AS extra
ORDER BY order_id
)


SELECT extras,COUNT(*)  as max_extras,topping_name  FROM nco 
LEFT JOIN npr_pt ON
nco.extras = npr_pt.topping_id
GROUP BY extras,npr_pt.topping_name
ORDER BY max_extras DESC 
LIMIT 1

```

<img width="1531" height="132" alt="image" src="https://github.com/user-attachments/assets/961c9f4f-56f8-4d06-8af5-97c82549877d" />



**Question 3:** What was the most common exclusion?

---

## SQL Code

```sql
WITH normalized_pizza_recipes AS
(
SELECT pizza_id, CAST (TRIM(topping) AS INTEGER) as topping_id
FROM pizza_recipes ,
  UNNEST(STRING_TO_ARRAY (toppings, ',')) AS  topping
), 


npr_pt AS
(
SELECT pizza_id,npr.topping_id,topping_name 
FROM normalized_pizza_recipes npr 
LEFT JOIN pizza_toppings pt ON
npr.topping_id = pt.topping_id
ORDER BY pizza_id,topping_id
)
,

nco AS 
(  
SELECT order_id,customer_id,pizza_id, CAST(TRIM(extra) AS INTEGER) as extras
FROM customer_orders ,
UNNEST(STRING_TO_ARRAY(extras,',')) AS extra
ORDER BY order_id
)


SELECT extras,COUNT(*)  as max_extras,topping_name  FROM nco 
LEFT JOIN npr_pt ON
nco.extras = npr_pt.topping_id
GROUP BY extras,npr_pt.topping_name
ORDER BY max_extras DESC 
LIMIT 1
```

<img width="1672" height="127" alt="image" src="https://github.com/user-attachments/assets/d7c4a03b-4d15-4996-ac27-aad01f61d9ec" />

**Question 4:** Generate an order item for each record in the customers_orders table in the format of one of the following:
Meat Lovers
Meat Lovers - Exclude Beef
Meat Lovers - Extra Bacon
Meat Lovers - Exclude Cheese, Bacon - Extra Mushroom, Peppers

---

## SQL Code

``` sql
WITH nco AS
(
SELECT order_id,pizza_id,CAST (TRIM(exclusion) AS INTEGER ) as exclusions, extras,order_time 
FROM customer_orders,
UNNEST(STRING_TO_ARRAY(exclusions,',')) AS  exclusion
ORDER BY order_id
),

EXCLUSIONS AS 
(
SELECT order_id,nco.pizza_id,STRING_AGG(DISTINCT topping_name , ',' )  as exclusion_topping_name
FROM nco LEFT JOIN pizza_names pn ON nco.pizza_id = pn.pizza_id 
LEFT JOIN pizza_toppings pt ON
nco.exclusions = pt.topping_id
GROUP BY order_id,nco.pizza_id
ORDER BY order_id
),

nco_extras AS
(
SELECT order_id,customer_id,pizza_id,CAST (TRIM(extra) AS INTEGER ) as extras,order_time 
FROM customer_orders,
UNNEST(STRING_TO_ARRAY(extras,',')) AS  extra
ORDER BY order_id
),

EXTRAS AS
(
SELECT order_id,nco_extras.pizza_id,STRING_AGG(DISTINCT topping_name , ',' )  as extra_topping_name
FROM nco_extras LEFT JOIN pizza_names pn ON nco_extras.pizza_id = pn.pizza_id 
LEFT JOIN pizza_toppings pt ON
nco_extras.extras = pt.topping_id
GROUP BY order_id,nco_extras.pizza_id
ORDER BY order_id
)

--  Created Two CTES for Extras and Exclusions to include  order id ,pizza id and topping names in a comma separated list
-- From those two CTES I joined Customer ORders table and then used string concatenation to get the results in the desired format. 

SELECT co.order_id,
CONCAT( pizza_name, (' -  Exclude  ' || exclusion_topping_name) ,(' -  Extra  ' || extra_topping_name ) ) as order_item
FROM customer_orders co 
LEFT JOIN pizza_names pn ON co.pizza_id = pn.pizza_id
LEFT JOIN EXTRAS e ON co.order_id=e.order_id AND co.pizza_id = e.pizza_id
LEFT JOIN EXCLUSIONS ex ON co.order_id=ex.order_id AND co.pizza_id = ex.pizza_id
ORDER BY order_id

```
<img width="1371" height="263" alt="image" src="https://github.com/user-attachments/assets/869258ed-1abe-4bc1-a9fa-3f653a02589d" />

<img width="1197" height="258" alt="image" src="https://github.com/user-attachments/assets/57ffdc51-313b-48df-9799-607edd598525" />

<img width="930" height="670" alt="image" src="https://github.com/user-attachments/assets/600ea960-50e7-4943-86bd-d87ec67e8f28" />




