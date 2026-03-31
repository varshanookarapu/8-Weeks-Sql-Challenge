# Data Bank

https://8weeksqlchallenge.com/case-study-4/

**Takeaway :**  Window Functions( Row number , lag ) , Running Totals , End of Month Balances , Min ,Max , Avg balances 

## A. Customer Nodes Exploration


**Question 1:** How many unique nodes are there on the Data Bank system?

---

## SQL Code

```sql
SELECT  DISTINCT node_id AS unique_nodes  FROM customer_nodes ORDER BY node_id;
```

<img width="789" height="301" alt="image" src="https://github.com/user-attachments/assets/df2e4936-5ede-45b8-9fcf-019618efe90a" />

---


**Question 2:** What is the number of nodes per region?

---

## SQL Code

```sql
SELECT cn.region_id,region_name, COUNT(node_id) as node_count FROM customer_nodes cn LEFT JOIN regions r 
ON cn.region_id = r.region_id
GROUP BY  cn.region_id,region_name
ORDER BY cn.region_id;
```
<img width="1364" height="316" alt="image" src="https://github.com/user-attachments/assets/0455b4a7-95f8-4c16-8397-327730d152f7" />

---

**Question 3:** How many customers are allocated to each region?

---

## SQL Code

```sql
SELECT cn.region_id,region_name, COUNT(DISTINCT customer_id) as customer_count FROM customer_nodes cn LEFT JOIN regions r 
ON cn.region_id = r.region_id
GROUP BY  cn.region_id,region_name
ORDER BY cn.region_id
```
<img width="1361" height="315" alt="image" src="https://github.com/user-attachments/assets/de13d849-2170-42c1-98f7-6b70a117b0ee" />

---

**Question 4:** How many days on average are customers reallocated to a different node?

---

## SQL Code

```sql

-- in this cte we are filtering all the rows where the end_date year is not equal to 9999  (there's about 500  rows )
WITH node_duration AS 
(
SELECT customer_id,node_id,start_date,end_date, (end_date - start_date) as duration
from 
customer_nodes
WHERE EXTRACT(YEAR FROM end_date) !=9999
),


-- In this cte we are checking next node id 
nd2 AS (
  
SELECT customer_id,node_id,duration,
LEAD(node_id) OVER(PARTITION BY customer_id ORDER BY start_date) as next_node_id 
FROM node_duration
)

-- In this statement we are filtering all the rows where there is reallocation from one node to another then calculating the average
SELECT 
    ROUND(AVG(duration), 2) AS average_days_until_reallocation
FROM nd2
WHERE 
node_id != next_node_id and next_node_id IS NOT NULL

```
<img width="476" height="160" alt="image" src="https://github.com/user-attachments/assets/a8e461cf-4b66-4e22-8a85-c763de315443" />

---

**Question 5:** What is the median, 80th and 95th percentile for this same reallocation days metric for each region?

---

## SQL Code 

```sql

WITH node_duration AS 
(
SELECT customer_id,cn.region_id,region_name,node_id,start_date,end_date, (end_date - start_date) as duration
from 
customer_nodes cn LEFT JOIN regions r ON
cn.region_id=r.region_id
WHERE EXTRACT(YEAR FROM end_date) !=9999
),



-- -- CTE to check the next nodes
nd2 AS (
  
SELECT region_name,node_id,duration,
LEAD(node_id) OVER(PARTITION BY customer_id ORDER BY start_date) as next_node_id 
FROM node_duration
)


SELECT 
    region_name,
    PERCENTILE_CONT(0.5)  WITHIN GROUP (ORDER BY duration) AS median_days,
    PERCENTILE_CONT(0.8)  WITHIN GROUP (ORDER BY duration) AS percentile_80,
    PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY duration) AS percentile_95
FROM nd2
WHERE next_node_id IS NOT NULL
  AND node_id != next_node_id
GROUP BY region_name
ORDER BY region_name;

```
<img width="1629" height="341" alt="image" src="https://github.com/user-attachments/assets/40031409-985e-4fa2-9ed7-29b05f3620ae" />

---


