
## C. Data Allocation Challenge
To test out a few different hypotheses - the Data Bank team wants to run an experiment where different groups of customers would be allocated data using 3 different options:

Option 1: data is allocated based off the amount of money at the end of the previous month
Option 2: data is allocated on the average amount of money kept in the account in the previous 30 days
Option 3: data is updated real-time
For this multi-part challenge question - you have been requested to generate the following data elements to help the Data Bank team estimate how much data will need to be provisioned for each option:

running customer balance column that includes the impact each transaction
customer balance at the end of each month
minimum, average and maximum values of the running balance for each customer
Using all of the data available - how much data would have been required for each option on a monthly basis?

---

## Option 1: data is allocated based off the amount of money at the end of the previous month

Basically the question is aksing us how much data is needed  to check the end of month  balance for every customer  i.e  one row for every customer for every month , we get the end of month balance , then we count the rows  for every month which gives the data that is needed 

```sql
WITH running_balance AS (
    SELECT
        customer_id,
        txn_date,
        SUM(CASE WHEN txn_type='deposit' THEN txn_amount ELSE -txn_amount END) 
            OVER (PARTITION BY customer_id ORDER BY txn_date) AS balance
    FROM customer_transactions
),
month_end AS (
    SELECT
        customer_id,
        EXTRACT ('month' FROM txn_date) AS month,
        MAX(balance) AS end_of_month_balance
    FROM running_balance
    GROUP BY customer_id, EXTRACT ('month' FROM txn_date)
    ORDER BY customer_id ,EXTRACT ('month' FROM txn_date)
)


SELECT month, COUNT(*) as rows_required
FROM month_end
GROUP BY month
ORDER BY month
```

<img width="1149" height="333" alt="image" src="https://github.com/user-attachments/assets/025c5a99-a21d-46d6-9fca-8119a9aeda54" />

---

## Option 2: data is allocated on the average amount of money kept in the account in the previous 30 days

In this question we will have to check the day by day data for every customer  since they are asking us to check the avg amount of money kept in account for previous 30 days , which means we need daily snapshot of the blances

```sql
WITH running_balance AS (
    SELECT
        customer_id,
        txn_date,
        SUM(CASE WHEN txn_type='deposit' THEN txn_amount ELSE -txn_amount END) 
            OVER (PARTITION BY customer_id ORDER BY txn_date) AS running_balance
    FROM customer_transactions
),


daily_snapshot AS (
    SELECT
        customer_id,
        txn_date,
        MAX(running_balance) AS end_of_day_balance
    FROM running_balance
    GROUP BY customer_id, txn_date
    ORDER BY customer_id ,txn_date
)


SELECT EXTRACT ('month' FROM txn_date) AS month, COUNT(*) as rows_required
FROM daily_snapshot
GROUP BY EXTRACT ('month' FROM txn_date)
ORDER BY EXTRACT ('month' FROM txn_date)
```

<img width="1024" height="318" alt="image" src="https://github.com/user-attachments/assets/a4d8e2b7-ada5-481f-9231-90c5921ccef7" />

---

## Option 3: data is updated real-time

here we are counting every transaction for every customer

```sql
WITH running_balance AS (
    SELECT
        customer_id,
        txn_date,
        SUM(CASE WHEN txn_type='deposit' THEN txn_amount ELSE -txn_amount END) 
            OVER (PARTITION BY customer_id ORDER BY txn_date) AS running_balance
    FROM customer_transactions
)



SELECT EXTRACT ('month' FROM txn_date) AS month, COUNT(*) as rows_required
FROM running_balance
GROUP BY EXTRACT ('month' FROM txn_date)
ORDER BY EXTRACT ('month' FROM txn_date)
```
---

<img width="863" height="345" alt="image" src="https://github.com/user-attachments/assets/76bc2501-84d7-4333-926b-9dc345cc296b" />


---


## running customer balance column that includes the impact each transaction


```sql
SELECT customer_id,txn_type,txn_date, txn_amount ,
SUM
(CASE WHEN txn_type = 'deposit' THEN txn_amount ELSE -txn_amount END ) 
OVER ( PARTITION BY customer_id ORDER BY txn_date ) 
AS running_balance FROM customer_transactions
ORDER BY customer_id, txn_date
```
<img width="1742" height="818" alt="image" src="https://github.com/user-attachments/assets/a4a89a27-ed68-4098-ab07-a25fd2cbff04" />

---

## customer balance at the end of each month
```sql

WITH balance AS
(
SELECT customer_id,txn_date, EXTRACT ( 'month' FROM txn_date ) as txn_month ,
SUM ( CASE WHEN txn_type = 'deposit' THEN txn_amount ELSE -txn_amount END )  AS balance
FROM customer_transactions 
GROUP BY customer_id,txn_date
),

running_total AS
(
 SELECT * , 
 SUM(balance) OVER(PARTITION BY customer_id ORDER BY txn_date) :: NUMERIC AS running_balance,
 ROW_NUMBER() OVER(PARTITION BY customer_id,txn_month ORDER BY txn_date DESC) as row_num
 FROM balance 
)  



-- closing balance at the end of every month 

SELECT customer_id, txn_month , (date_trunc('month',txn_date) + INTERVAL '1 month - 1 day ' ) :: DATE as end_of_month ,running_balance as customer_balance_at_eom
FROM running_total WHERE row_num =1
ORDER BY customer_id,txn_month

```

<img width="1645" height="882" alt="image" src="https://github.com/user-attachments/assets/6702305c-50ec-48c2-97ce-7bf7b0492bc1" />

---

## minimum, average and maximum values of the running balance for each customer

```sql

WITH CTE AS 
(
SELECT customer_id,txn_type,txn_date, txn_amount ,
SUM
(CASE WHEN txn_type = 'deposit' THEN txn_amount ELSE -txn_amount END ) 
OVER ( PARTITION BY customer_id ORDER BY txn_date ) 
AS running_balance FROM customer_transactions
ORDER BY customer_id, txn_date
)


SELECT customer_id , MIN(running_balance) as min_running_balance, MAX(running_balance) as max_running_balance , ROUND(AVG(running_balance),2)  as  avg_running_balance
FROM CTE 
GROUP BY customer_id
ORDER BY customer_id
```

<img width="1668" height="845" alt="image" src="https://github.com/user-attachments/assets/9986d6f2-a113-4a41-8019-dec2f22aa4f2" />

