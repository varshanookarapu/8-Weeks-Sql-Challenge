
**Question 1:** What is the unique count and total amount for each transaction type?

```sql
SELECT txn_type, COUNT(txn_type) as unique_count , SUM(txn_amount) as total_amount FROM customer_transactions
GROUP BY txn_type
ORDER BY txn_type
```
<img width="1565" height="213" alt="image" src="https://github.com/user-attachments/assets/c4c5260c-e2af-4014-87da-6ae07a1e6588" />

---

**Question 2:** What is the average total historical deposit counts and  average amounts for all customers
```sql
WITH customer_deposits AS 
(
SELECT customer_id, COUNT(txn_type) AS deposit_count, SUM (txn_amount) as deposit_amount
FROM customer_transactions
WHERE txn_type = 'deposit'
GROUP BY customer_id
)

SELECT  ROUND(AVG(deposit_count)) as average_deposit_Count , ROUND(AVG(deposit_amount)) as avg_deposit_amount FROM customer_deposits
```
<img width="1492" height="155" alt="image" src="https://github.com/user-attachments/assets/069107a3-b8b6-4e5a-a48d-c8a0801b6f8a" />

---

**Question 3:** For each month - how many Data Bank customers make more than 1 deposit and either 1 purchase or 1 withdrawal in a single month?

```sql
WITH customer_activity_counts AS ( 
SELECT customer_id, EXTRACT(MONTH from txn_date) as month,to_char(txn_date,'month') as month_name,
     COUNT (CASE WHEN txn_type='deposit' THEN 1 END ) AS deposit_count,
     COUNT (CASE WHEN txn_type='purchase' THEN 1 END ) AS purchase_count,
     COUNT (CASE WHEN txn_type='withdrawal' THEN 1 END ) AS withdrawal_count
FROM customer_transactions
GROUP BY customer_id ,EXTRACT(MONTH from txn_date) , to_char(txn_date,'month')
ORDER BY customer_id ,EXTRACT(MONTH from txn_date) , to_char(txn_date,'month')
)


SELECT month,month_name, COUNT(customer_id) FROM customer_activity_counts 
WHERE deposit_count > 1 AND ( purchase_count >= 1  OR withdrawal_count >=1   )
GROUP BY month ,month_name
ORDER BY month
```

<img width="1669" height="318" alt="image" src="https://github.com/user-attachments/assets/7f441706-792b-4f1e-9552-b09b8e8ee67a" />


---


**Question 4:** What is the closing balance for each customer at the end of the month? 

```sql
WITH customer_amounts AS
(
SELECT customer_id, txn_date, EXTRACT ('month'FROM txn_date)    AS txn_month,
SUM ((CASE WHEN txn_type = 'deposit' THEN txn_amount ELSE 0 END ) - (CASE WHEN txn_type <> 'deposit' THEN txn_amount ELSE 0 END )) as balance
FROM customer_transactions 
GROUP BY customer_id ,txn_date ,EXTRACT ('month'FROM txn_date)
),

balance AS
(
SELECT 
  customer_id,
  txn_date,
  txn_month,
  balance,
  SUM(balance) OVER(PARTITION BY customer_id ORDER BY txn_date) AS running_sum,
  ROW_NUMBER() OVER(PARTITION BY customer_id,txn_month  ORDER BY txn_date DESC) AS row_num
FROM customer_amounts 
  
)

SELECT customer_id, (date_trunc('month',txn_date) + INTERVAL '1 month - 1 day ' ) :: DATE as end_of_month,
txn_month,running_sum as closing_balance
FROM balance
WHERE customer_id =429 AND row_num=1

-- remove the filter for customer_id to get values for all the other customers
```
<img width="1660" height="304" alt="image" src="https://github.com/user-attachments/assets/ea75ea0d-75e3-4b1e-bcb4-f1b3731c87ff" />

---

**Question 5:** What is the percentage of customers who increase their closing balance by more than 5%?

```sql

WITH customer_amounts AS
(
SELECT customer_id, txn_date, EXTRACT ('month'FROM txn_date)    AS txn_month,
SUM ((CASE WHEN txn_type = 'deposit' THEN txn_amount ELSE 0 END ) - (CASE WHEN txn_type <> 'deposit' THEN txn_amount ELSE 0 END )) as balance
FROM customer_transactions 
GROUP BY customer_id ,txn_date ,EXTRACT ('month'FROM txn_date)
),

balance AS
(
SELECT 
  customer_id,
  txn_date,
  txn_month,
  balance,
  SUM(balance) OVER(PARTITION BY customer_id ORDER BY txn_date) AS running_sum,
  ROW_NUMBER() OVER(PARTITION BY customer_id,txn_month  ORDER BY txn_date DESC) AS row_num
FROM customer_amounts 
  
),

-- From this CTE we are calculating the closing blances for every month for each customer
running_sum AS 
(
SELECT customer_id, (date_trunc('month',txn_date) + INTERVAL '1 month - 1 day ' ) :: DATE as end_of_month,
txn_month,running_sum as closing_balance,
LAG(running_sum) OVER(PARTITION BY customer_id ORDER BY txn_month) AS previous_month_balance
FROM balance
WHERE row_num=1
),

-- From this CTE we are calculating the percentage change between current months closing balance and previous months closing balance
cbpg AS(
  
SELECT customer_id,txn_month,closing_balance,previous_month_balance, 
 CASE WHEN previous_month_balance IS NULL OR previous_month_balance = 0 THEN NULL 
ELSE ROUND(((closing_balance - previous_month_balance ) / previous_month_balance ) * 100 , 2) END  AS percentage_change
FROM running_sum 
  )
  
 -- SELECT COUNT(DISTINCT customer_id) FROM cbpg
 -- WHERE percentage_change > 5 AND previous_month_balance > 0 

-- Here we  are filerting all the  customers whose percentage change is greater than 5 and calculating the percentage of customers

SELECT (COUNT(DISTINCT customer_id) :: NUMERIC/ (SELECT COUNT(distinct customer_id) FROM customer_transactions) :: NUMERIC) * 100
AS percentageofcustomers FROM cbpg WHERE percentage_change > 5 

```
<img width="439" height="166" alt="image" src="https://github.com/user-attachments/assets/efc8c45d-0c60-4a64-9dcc-c4f942362486" />
