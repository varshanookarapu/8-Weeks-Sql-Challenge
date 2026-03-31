# SQL Case Study 1: Danny's Diner

*Brushing up my SQL skills through the 8 Weeks SQL challenges by Danny Ma (https://8weeksqlchallenge.com/) , I have explored the following concepts: Basic Aggregation, CTEs, and Ranking functions.*

**Question 1:** What is the total amount each customer spent at the restaurant?

---

## SQL Code

```sql
SELECT customer_id, SUM(price) AS Total_Amount_Spent FROM Sales LEFT JOIN Menu ON
sales.product_id = menu.product_id
GROUP BY customer_id 
ORDER BY customer_id ;
```
<img width="1665" height="228" alt="image" src="https://github.com/user-attachments/assets/1e3bf06a-d1d0-4ecc-bc75-fc65f5fac14e" />


**Question 2:** How many days has each customer visited the restaurant?

---

## SQL Code

```sql
SELECT sales.customer_id,COUNT(DISTINCT sales.order_date)AS days_customer_visited 
FROM Sales LEFT JOIN Members ON
sales.customer_id = members.customer_id
GROUP BY sales.customer_id,members.join_date
ORDER BY sales.customer_id
```
<img width="1685" height="252" alt="image" src="https://github.com/user-attachments/assets/3845cd89-d439-4403-845e-cf84ebc160ae" />


**Question 3:** What was the first item from the menu purchased by each customer?

---

## SQL Code

```sql
WITH first_item_purchased AS 
(
SELECT customer_id,order_date,sales.product_id,product_name,price,
dense_rank() over (partition by sales.customer_id order by order_date asc) as product_rank 
FROM Sales LEFT JOIN Menu ON
sales.product_id = menu.product_id 
)

SELECT customer_id,product_name,product_rank FROM first_item_purchased
WHERE product_rank =1 
ORDER BY customer_id ;
```

<img width="1684" height="341" alt="image" src="https://github.com/user-attachments/assets/fe73a652-8640-424b-b8dc-aba0ad80df06" />

**Question 4:** What is the most purchased item on the menu and how many times was it purchased by all customers?

---

## SQL Code

```sql
SELECT  product_name, COUNT(sales.product_id) as times_purchased FROM Sales LEFT JOIN Menu ON
sales.product_id = menu.product_id 
GROUP BY  product_name
ORDER BY  times_purchased DESC;
```
<img width="1694" height="231" alt="image" src="https://github.com/user-attachments/assets/f127c359-d2f9-4d65-b64f-350fbce847ca" />

**Question 5:** Which item was the most popular for each customer?

---

## SQL Code

```sql
WITH popular_item AS 
(
SELECT customer_id,product_name, COUNT(sales.product_id) as times_purchased ,
RANK() OVER (partition by customer_id ORDER BY COUNT(sales.product_id)  DESC) as rank,
ROW_NUMBER () OVER (partition by customer_id ORDER BY COUNT(sales.product_id)  DESC) as row_number
FROM Sales LEFT JOIN Menu ON
sales.product_id = menu.product_id 
GROUP BY  product_name , customer_id
ORDER BY customer_id, times_purchased DESC
  )


SELECT customer_id,product_name,times_purchased FROM popular_item WHERE rank = 1 and row_number=1
```
<img width="1687" height="235" alt="image" src="https://github.com/user-attachments/assets/fe26d797-4b48-4964-9b58-11d6e261c84f" />

**Question 6:** Which item was purchased first by the customer after they became a member?

---

## SQL Code

```sql
WITH first_item_purchased_after_membership AS 
(
SELECT sales.customer_id,order_date,sales.product_id,product_name,price,join_date ,
RANK() OVER(partition by sales.customer_id order by order_date ) as product_rank
FROM Sales LEFT JOIN Menu ON sales.product_id = menu.product_id 
LEFT JOIN Members ON sales.customer_id = members.customer_id
WHERE order_date >= join_date
)

SELECT customer_id,product_name,order_date,join_date 
FROM first_item_purchased_after_membership 
WHERE product_rank = 1

```

<img width="1668" height="203" alt="image" src="https://github.com/user-attachments/assets/575ab7cf-526b-485f-ab14-48c593c85d40" />

**Question 7:** Which item was purchased just before the customer became a member?

---

## SQL Code

```sql
WITH cte AS
(
SELECT sales.customer_id,sales.product_id,product_name,price,order_date,join_date ,
RANK() OVER(partition by sales.customer_id order by order_date DESC ) as product_rank,
ROW_NUMBER() OVER(partition by sales.customer_id order by order_date DESC) as row_number
FROM Sales LEFT JOIN Menu ON sales.product_id = menu.product_id 
LEFT JOIN Members ON sales.customer_id = members.customer_id
WHERE order_date < join_date
)

SELECT customer_id,product_name FROM cte WHERE product_rank =1;

```
<img width="1691" height="239" alt="image" src="https://github.com/user-attachments/assets/0e35d3ea-5588-483e-a8de-1c92f50d43c8" />

**Question 8:** What is the total items and amount spent for each member before they became a member?

---

## SQL Code

```sql
SELECT sales.customer_id,COUNT(product_name) as total_items_bought,SUM(price) as total_amount
FROM Sales LEFT JOIN Menu ON sales.product_id = menu.product_id 
LEFT JOIN Members ON sales.customer_id = members.customer_id
WHERE order_date < join_date
GROUP BY sales.customer_id
ORDER BY sales.customer_id

```
<img width="1051" height="133" alt="image" src="https://github.com/user-attachments/assets/6bb34c22-c8a4-438d-a8ff-54dd339503a2" />

**Question 9:** If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?

---

## SQL Code

```sql
WITH points_multiplier AS 
(
SELECT customer_id,sales.product_id, product_name, SUM(price) AS total_amount_spent 
FROM Sales LEFT JOIN Menu ON
sales.product_id = menu.product_id
GROUP BY customer_id ,product_name,sales.product_id
ORDER BY customer_id,sales.product_id 
),

total_points AS (
SELECT customer_id,product_name,total_amount_spent,
CASE WHEN product_id = 1  THEN total_amount_spent*20
ELSE total_amount_spent*10 
END AS total_points  
FROM points_multiplier
)

SELECT customer_id, SUM(total_points) as cumulative_points
FROM total_points 
GROUP BY customer_id
ORDER BY customer_id

```
<img width="1696" height="239" alt="image" src="https://github.com/user-attachments/assets/2c4d6e41-f418-4445-8896-4eb01b0597e2" />

**Question 10:** In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?

---

## SQL Code

```sql
WITH points_multiplier_2 AS (
    SELECT 
        sales.customer_id,
        sales.product_id, 
        product_name, 
        price,
        order_date,
        join_date,
        (join_date + INTERVAL '6 days') :: date as week_after_join_date,
        DATE '2021-01-31' as end_of_january,
      
  CASE 
            WHEN order_date BETWEEN join_date AND join_date + INTERVAL '6 days' THEN price*10*2
            WHEN product_name = 'sushi' THEN price*10*2
            ELSE price*10
        END AS cumulative_points 
  
  FROM Sales 
    INNER JOIN Menu ON sales.product_id = menu.product_id
    INNER JOIN members ON sales.customer_id = members.customer_id
)

SELECT customer_id, SUM(cumulative_points) as total_points
FROM points_multiplier_2
GROUP BY customer_id
ORDER BY customer_id

```
<img width="1668" height="217" alt="image" src="https://github.com/user-attachments/assets/1342aba0-ee04-445c-920d-e6faafda15fa" />


**BONUS Question 11:** Join All Tables and Check if member or not 

---

## SQL Code

```sql
SELECT sales.customer_id,sales.order_date,menu.product_name,menu.price,
CASE WHEN sales.order_date >= members.join_date THEN 'Y'
ELSE 'N'
END as member
FROM  Sales LEFT JOIN Menu ON sales.product_id = menu.product_id
LEFT JOIN  members ON sales.customer_id = members.customer_id
```
<img width="1911" height="799" alt="image" src="https://github.com/user-attachments/assets/f3b885e7-a540-4a92-aff5-3afe8da4d1ec" />

**BONUS Question 12:** Rank All Things 

---

## SQL Code

```sql
SELECT sales.customer_id,sales.order_date,menu.product_name,menu.price,
CASE WHEN sales.order_date >= members.join_date THEN 'Y'
ELSE 'N'
END as member,
 CASE 
    WHEN sales.order_date >= members.join_date THEN
        DENSE_RANK() OVER (
            PARTITION BY sales.customer_id
            ORDER BY 
                CASE 
                    WHEN sales.order_date >= members.join_date 
                    THEN sales.order_date 
                END
        )
END AS ranking
FROM  Sales LEFT JOIN Menu ON sales.product_id = menu.product_id
LEFT JOIN  members ON sales.customer_id = members.customer_id
ORDER BY sales.customer_id,sales.order_date,menu.product_name ```

```
<img width="1917" height="826" alt="image" src="https://github.com/user-attachments/assets/f72b413c-57ea-43ea-9018-8406ba0059d4" />



