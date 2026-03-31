## C.Challenge Payment Question

The Foodie-Fi team wants you to create a new payments table for the year 2020 that includes amounts paid by each customer in the subscriptions table with the following requirements:

##### monthly payments always occur on the same day of month as the original start_date of any monthly paid plan
##### upgrades from basic to monthly or pro plans are reduced by the current paid amount in that month and start immediately
##### upgrades from pro monthly to pro annual are paid at the end of the current billing period and also starts at the end of the month period
##### once a customer churns they will no longer make payments

---

## SQL Code

```sql
WITH RECURSIVE 

base AS 
(
SELECT customer_id, s.plan_id,plan_name, 
LAG(s.plan_id) OVER(PARTITION BY customer_id ORDER BY start_date ASC ) as previous_plan_id,
LEAD(s.plan_id) OVER(PARTITION BY customer_id ORDER BY start_date ASC ) as next_plan_id,
start_date, 
LAG(start_date)OVER(PARTITION BY customer_id ORDER BY start_date ASC ) as previous_plan_date,
LEAD(start_date)OVER(PARTITION BY customer_id ORDER BY start_date ASC ) as next_plan_date,price,
LAG(price) OVER(PARTITION BY customer_id ORDER BY start_date ASC ) as previous_price
FROM subscriptions s LEFT JOIN plans p ON 
s.plan_id = p.plan_id

),  

-- WE ARE FINDING THE PERIOD END DATE HERE 
base2 AS 
(
SELECT * , 
(CASE WHEN next_plan_date IS NOT NULL THEN next_plan_date - (INTERVAL '1 DAY')
ELSE DATE '2020-12-31' 
END) :: DATE
AS 
period_end_date 
FROM base
),

-- payments table to have customer_id,plan_id,plan_name,payment_date,amount , payment_order
monthly_payments AS 
(
    -- anchor
    SELECT customer_id, plan_id, plan_name, 
  
    CASE WHEN plan_id = 2 AND previous_plan_id=1 THEN  (start_date + INTERVAL '1 Month') :: DATE 
    ELSE start_date 
    END
    AS payment_date, price AS amount, period_end_date
    FROM base2
    WHERE plan_id IN (1,2) 

    UNION ALL

    -- recursive
    SELECT customer_id, plan_id, plan_name, (payment_date + INTERVAL '1 MONTH')::date AS payment_date, amount, period_end_date
    FROM monthly_payments
    WHERE (payment_date + INTERVAL '1 MONTH') <= LEAST ( period_end_date , DATE '2020-12-31')
),
-- LEAST uses the earlier date of the two dates. 




--  When updgraded from basic to pro monthly plan , we are ensuring the price is updated
basictopromonthly AS
(
SELECT customer_id,plan_id,plan_name,start_date as payment_date, price - previous_price as amount, period_end_date 
FROM base2 
WHERE plan_id = 2 and previous_plan_id = 1 
),

protoannual AS 
(
SELECT customer_id,plan_id,plan_name,start_date as payment_date, price as amount, period_end_date 
FROM base2 
WHERE plan_id = 3 and previous_plan_id = 2 
 ) ,


all_payments AS 
(
SELECT * FROM monthly_payments 
UNION ALL
SELECT * FROM basictopromonthly
UNION ALL
SELECT * from protoannual
)


SELECT customer_id,plan_id,plan_name,payment_date,amount, period_end_date,
ROW_NUMBER() OVER(PARTITION BY customer_id ORDER BY payment_date) AS payment_order
FROM all_payments 
WHERE payment_date <= DATE '2020-12-31'
ORDER BY customer_id,payment_date


```

---
To address this challenge, I’m going to create three CTEs then i will combile all three to generate the payments table. In first step I am creating a base table, which contains all the information about the next plan, including the next plan dates, as well as the previous plans and their dates and previous price details.


```sql
WITH base AS 
(
SELECT customer_id,
s.plan_id,
plan_name,
LAG(s.plan_id) OVER(PARTITION BY customer_id ORDER BY start_date ASC) as previous_plan_id,
LEAD(s.plan_id) OVER(PARTITION BY customer_id ORDER BY start_date ASC) as next_plan_id,
start_date,
LAG(start_date) OVER(PARTITION BY customer_id ORDER BY start_date ASC) as previous_plan_date,  
LEAD(start_date) OVER(PARTITION BY customer_id ORDER BY start_date ASC) as next_plan_date,  
price,
LAG(price) OVER(PARTITION BY customer_id ORDER BY start_date ASC) as previous_price   
FROM subscriptions s LEFT JOIN plans p on
s.plan_id = p.plan_id
  
)

SELECT * FROM base ORDER BY customer_id, start_date
```

<img width="1894" height="830" alt="image" src="https://github.com/user-attachments/assets/8d66017a-18c3-4528-bf01-5ac7740ffdc7" />

---
Now i want to calculate period end date  so basically what we are going to do here is to check for the next plan date if it is not null then we take the next plan date then subtract 1 day from it which gives the current plan's period end date  before the next plan starts

``` sql
base2 AS (
  
SELECT *, CASE WHEN next_plan_date  IS NOT NULL THEN (next_plan_date - INTERVAL '1 DAY') :: DATE 
  ELSE DATE '2020-12-31' END 
  AS 
 period_end_date 
FROM base 
)
SELECT * FROM base2 ORDER BY customer_id, start_date
```

<img width="1894" height="824" alt="image" src="https://github.com/user-attachments/assets/be6d8e45-92e3-4341-9f57-f655d1bdaece" />

---
Now in this CTE we are going to create payment dates for basic monthly and pro monthly plans, which will recursively calculate the next payment date based on the start date and then will end at date on or before Dec 31 2020. 

```sql

WITH RECURSIVE
monthly_payments AS 
(
-- anchor
  
SELECT customer_id,
plan_id,
plan_name,
CASE WHEN plan_id =2 AND previous_plan_id = 1 THEN  
(start_date + INTERVAL '1 Month') :: DATE 
ELSE start_date 
END
AS payment_date,
price as amount,
period_end_date
FROM base2
WHERE plan_id IN (1,2)  
  
UNION ALL
  
-- recursive
SELECT customer_id,
plan_id,
plan_name,  
(payment_date + INTERVAL '1 Month') :: DATE AS payment_date,
amount,
period_end_date
FROM monthly_payments
WHERE  (payment_date + INTERVAL '1 Month') :: DATE  <= LEAST (period_end_date, DATE '2020-12-31' )
    
) 


SELECT * FROM monthly_payments WHERE customer_id =8 ORDER BY customer_id, payment_date

-- Notice how in the below example we skipped a payment date for August month , this is because before the end of the basic monthly payment period , the pro monthly period is starting and we need to account for price change which will be addressed in the next cte.
```
<img width="1703" height="390" alt="image" src="https://github.com/user-attachments/assets/ca764f48-5fc4-4c07-b688-2ad22fffd056" />

---
In this CTE we are going to be doing the price adjustment when we are upgrading from basic monthly to pro monthly or pro yearly  i.e going from plan 1 to plan 2 or plan 1 to plan 3

```sql
basictopromonthly AS 
(
SELECT customer_id,
plan_id,
plan_name,
start_date as payment_date,
(price-previous_price) AS amount,
period_end_date
FROM  base2 
WHERE (plan_id = 2 AND previous_plan_id = 1) OR  (plan_id = 3 AND previous_plan_id = 1) 
  
  
)  
SELECT * FROM basictopromonthly WHERE customer_id =8 
ORDER BY customer_id, payment_date

-- Notice in the previous cte when we missed a row  we are now resoliving that scenario with this CTE , where we did the price adjustment.
```
<img width="1696" height="187" alt="image" src="https://github.com/user-attachments/assets/a01351b1-9b8d-45e6-b1e3-0b622b4a5d51" />
<img width="1671" height="172" alt="image" src="https://github.com/user-attachments/assets/9980fa2c-705b-443e-8ef7-0f499cebc65a" />
---
Now i am going to address the upgrades from pro monthly to pro annual are paid at the end of the current billing period and also starts at the end of the month period

```sql
protoannual AS
(
SELECT 
  customer_id,
  plan_id,
  plan_name,
  start_date as payment_date, 
  price as amount, 
  period_end_date 
FROM base2 
WHERE plan_id = 3 and previous_plan_id = 2   
)

SELECT * FROM protoannual WHERE customer_id =19
ORDER BY customer_id, payment_date
```
<img width="1626" height="148" alt="image" src="https://github.com/user-attachments/assets/d404b795-cba6-4b4b-93a6-69f863aaa607" />

--- 
Finally we now combine all the CTES , add the payment_order column and then the final conditon to include only the dates till Dec 31 2020 

```sql
all_payments AS 
(
SELECT * FROM monthly_payments 
UNION ALL
SELECT * FROM basictopromonthly
UNION ALL
SELECT * from protoannual
)

SELECT
customer_id,
plan_id,
plan_name,
payment_date,
amount,
period_end_date,
ROW_NUMBER() OVER(PARTITION BY customer_id ORDER BY payment_date) AS payment_order
FROM all_payments 
WHERE payment_date <= DATE '2020-12-31'
ORDER BY customer_id,payment_date

```
---
Few output screenshots for refernce 

Plan id 1 to Plan id 2 
<img width="1893" height="500" alt="image" src="https://github.com/user-attachments/assets/7fd4ea6c-c8d9-4346-a26f-883a061107a7" />

Plan id 1 to Plan id 3
<img width="1903" height="366" alt="image" src="https://github.com/user-attachments/assets/e8003cf3-ac5b-44c0-acd0-da6cdfa0e96e" />

Plan id 2 to Plan id 3
<img width="1689" height="246" alt="image" src="https://github.com/user-attachments/assets/ea2fc9a7-24a6-4bc8-99e9-8ef77d77a1fa" />

Churn case 
<img width="1686" height="224" alt="image" src="https://github.com/user-attachments/assets/3dd079d8-0fc4-459e-af57-4ecccfcdaa24" />
