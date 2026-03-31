# Case Study 6 : Clique Bait

## 1. Enterprise Relationship Diagram
<img width="1842" height="788" alt="image" src="https://github.com/user-attachments/assets/8286ba3a-f9d2-4980-91c9-d24f19f5d9d8" />

## 2. Digital Analysis

**Question 1:** How many users are there?

```sql
SELECT COUNT(DISTINCT user_id) AS user_count FROM clique_bait.users;
```
<img width="244" height="106" alt="image" src="https://github.com/user-attachments/assets/65552fc1-a738-44c4-9eab-f7381a68085c" />

**Question 2:** How many cookies does each user have on average?
```sql
SELECT  ROUND((COUNT(cookie_id):: NUMERIC /(SELECT COUNT(DISTINCT user_id) FROM clique_bait.users)::NUMERIC),2)   as avegrage_cookie_count FROM clique_bait.users
```
<img width="286" height="112" alt="image" src="https://github.com/user-attachments/assets/6941c1d0-933c-4deb-8ba2-306531acc0c6" />

**Question 3:** What is the unique number of visits by all users per month?

```sql
SELECT EXTRACT( 'month' FROM event_time) as month,TO_CHAR(event_time,'Month') as month_name, COUNT(DISTINCT visit_id)  as unique_visit_count
FROM clique_bait.events e LEFT JOIN clique_bait.users u ON
e.cookie_id = u.cookie_id
GROUP BY EXTRACT( 'month' FROM event_time),TO_CHAR(event_time,'Month')
ORDER BY EXTRACT( 'month' FROM event_time)
```
<img width="1130" height="273" alt="image" src="https://github.com/user-attachments/assets/1e18f260-7921-49c1-ba06-e571330dcf70" />


**Question 4:** What is the number of events for each event type?
```sql
SELECT event_type, COUNT(*) as event_count FROM clique_bait.events 
GROUP BY event_type
ORDER BY event_type
```
<img width="1015" height="282" alt="image" src="https://github.com/user-attachments/assets/97f33df7-5707-4120-b2b3-82d596b1f39e" />

**Question 5:** What is the percentage of visits which have a purchase event?

```sql
SELECT  100*COUNT(CASE WHEN event_type=3 THEN visit_id END)/ (SELECT COUNT(DISTINCT visit_id) FROM clique_bait.events ) :: NUMERIC AS percentage_of_visits_for_purchase_event

FROM
clique_bait.events 

```
<img width="385" height="104" alt="image" src="https://github.com/user-attachments/assets/fda7daed-b308-4da3-b087-f9f42120cf34" />


**Question 6:** What is the percentage of visits which view the checkout page but do not have a purchase event?
```sql
```

**Question 7:** What are the top 3 pages by number of views?

```sql
SELECT  e.page_id ,page_name ,COUNT(visit_id) visitor_count ,
RANK() OVER( ORDER BY COUNT(visit_id) DESC) as rank
FROM clique_bait.events e LEFT JOIN clique_bait.page_hierarchy ph ON
e.page_id = ph.page_id
GROUP BY e.page_id,page_name
LIMIT 3
```
<img width="1528" height="191" alt="image" src="https://github.com/user-attachments/assets/0ba58084-267b-4a63-8fd3-ede639fc9092" />

**Question 8:** What is the number of views and cart adds for each product category?

```sql
SELECT  product_category,

COUNT(CASE WHEN e.event_type = 1  THEN 1 END) AS page_view_count, 
COUNT(CASE WHEN e.event_type = 2  THEN 1 END) AS cart_add_count

FROM clique_bait.events e LEFT JOIN clique_bait.page_hierarchy ph 
ON e.page_id = ph.page_id
LEFT JOIN clique_bait.event_identifier ei 
ON e.event_type=ei.event_type
WHERE product_category IS NOT NULL
GROUP BY product_category 

```
<img width="1485" height="198" alt="image" src="https://github.com/user-attachments/assets/43526b7f-ea3e-4f9d-aa2a-4cc19b0a2104" />


**Question 9:** What are the top 3 products by purchases?

```sql
--in this cte we get the distinct visit_id which led to the purchase event 
WITH cte AS
(
SELECT  DISTINCT visit_id  FROM clique_bait.events WHERE event_type =3
) 

-- From the cte we are querying page name and counting the visit ids by joing the events and cte together and grouping by pagename

SELECT page_name, COUNT(e.visit_id) as visit_count 
FROM clique_bait.events e
LEFT JOIN clique_bait.page_hierarchy ph ON
e.page_id = ph.page_id
LEFT JOIN cte ON
e.visit_id = cte.visit_id
WHERE ph.product_id IS NOT NULL
GROUP BY page_name
ORDER BY COUNT(e.visit_id) DESC
LIMIT 3
```
<img width="1238" height="239" alt="image" src="https://github.com/user-attachments/assets/a5e10d7a-c81f-406f-abb9-3685a1822604" />

---

## 3. Funnel Analysis
---
Using a single SQL query - create a new output table which has the following details:

How many times was each product viewed?
How many times was each product added to cart?
How many times was each product added to a cart but not purchased (abandoned)?
How many times was each product purchased?
Additionally, create another table which further aggregates the data for the above points but this time for each product category instead of individual products.

---

```sql
-- -- Product View Count and add to cart count 

WITH table1 AS
(
SELECT product_id, page_name as product, product_category, visit_id,

SUM(CASE WHEN event_type =1 THEN 1 ELSE 0 END) AS page_view_count ,
SUM(CASE WHEN event_type =2 THEN 1 ELSE 0 END) AS addtocart_count

FROM  clique_bait.events e LEFT JOIN
clique_bait.page_hierarchy ph ON
e.page_id = ph.page_id
WHERE  product_id IS NOT NULL
GROUP BY e.page_id,page_name , product_id,visit_id ,product_category
ORDER BY product_id
),

-- purchase events 

table2 AS (
  
  SELECT DISTINCT visit_id FROM clique_bait.events WHERE event_type = 3
),

table3 AS
(
SELECT product_id,product,product_category,t1.visit_id,page_view_count,addtocart_count, CASE WHEN t2.visit_id IS NOT NULL THEN 1 ELSE 0 END as purchase_count
FROM table1 t1 LEFT JOIN table2 t2 ON
t1.visit_id = t2.visit_id
),

product_count_analysis AS (
SELECT product_id ,product, product_category, 
    SUM(page_view_count) AS page_view_count,
    SUM(addtocart_count) AS addtocart_count, 
    SUM(CASE WHEN addtocart_count = 1 AND purchase_count = 0 THEN 1 ELSE 0 END) AS abandoned_count,
    SUM(CASE WHEN addtocart_count = 1 AND purchase_count = 1 THEN 1 ELSE 0 END) AS purchases_count
  FROM table3
  GROUP BY product_id, product ,product_category
  ORDER BY product_id
 ),
 
 product_category_analysis AS (
  
 SELECT  product_category, 
    SUM(page_view_count) AS page_view_count,
    SUM(addtocart_count) AS addtocart_count, 
    SUM(CASE WHEN addtocart_count = 1 AND purchase_count = 0 THEN 1 ELSE 0 END) AS abandoned_count,
    SUM(CASE WHEN addtocart_count = 1 AND purchase_count = 1 THEN 1 ELSE 0 END) AS purchases_count
  FROM table3
  GROUP BY product_category
)
```
<img width="1880" height="493" alt="image" src="https://github.com/user-attachments/assets/342a84af-19a0-4860-b8ac-298843acbec1" />

<img width="1679" height="191" alt="image" src="https://github.com/user-attachments/assets/005f79b0-d35b-4921-8949-d39a3e1597fb" />

---

Use your 2 new output tables - answer the following questions:
---
**Question 1 :** Which product had the most views, cart adds and purchases?

```sql
SELECT product,page_view_count FROM product_count_analysis
WHERE page_view_count = (SELECT MAX(page_view_count) FROM product_count_analysis ) ;

SELECT product,addtocart_count FROM product_count_analysis
WHERE addtocart_count = (SELECT MAX(addtocart_count) FROM product_count_analysis );

SELECT product,abandoned_count FROM product_count_analysis
WHERE abandoned_count = (SELECT MAX(abandoned_count) FROM product_count_analysis );

SELECT product,purchases_count FROM product_count_analysis
WHERE purchases_count = (SELECT MAX(purchases_count) FROM product_count_analysis );
```
<img width="879" height="357" alt="image" src="https://github.com/user-attachments/assets/a6e1be6b-644c-4df1-a696-ea2d12cdbaf9" />

Oyster had most views
Lobster has most add to carts  and purchases

---

**Question 2 :** Which product was most likely to be abandoned?

Russian Caviar was most likely to be abandoned.
---
**Question 3 :** Which product had the highest view to purchase percentage?

```sql
-- basically for each product that is viewd how many were purchased 
SELECT product, ROUND(100*(purchases_count/page_view_count),2) as view_to_purchase_percentage FROM product_count_analysis
ORDER BY view_to_purchase_percentage DESC
```
<img width="1027" height="491" alt="image" src="https://github.com/user-attachments/assets/4c67928f-886e-4a58-adb6-7f18b92a8d84" />

---
**Question 4 :** What is the average conversion rate from view to cart add?

```sql
SELECT ROUND(100*(SUM(addtocart_count)/SUM(page_view_count)),2) as avg_conversion_rate_from_view_to_cartadd FROM product_count_analysis
```
<img width="449" height="72" alt="image" src="https://github.com/user-attachments/assets/3d7b3fa9-d650-4a4e-a260-2dd7da2f5f08" />

---
**Question 5 :** What is the average conversion rate from cart add to purchase?

```sql
-- for every cart add how many led to purchases , basically on an average , out of all cart adds how many were turned to purchases.
SELECT ROUND(100*(SUM(purchases_count)/SUM(addtocart_count)),2) as avg_conversion_rate_from_cartadd_to_purchase FROM product_count_analysis
```
<img width="510" height="109" alt="image" src="https://github.com/user-attachments/assets/d8237292-c0c2-4602-855b-810cf2bf0b81" />

---
