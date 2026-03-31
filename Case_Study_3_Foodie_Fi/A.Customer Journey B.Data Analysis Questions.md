# SQL Case Study 3 : Foodie-Fi

https://8weeksqlchallenge.com/case-study-3/

## A. Customer Journey

**Question 1:** Based off the 8 sample customers provided in the sample from the subscriptions table, write a brief description about each customer’s onboarding journey.
Try to keep it as short as possible - you may also want to run some sort of join to make your explanations a bit easier!

---

## SQL Code

```sql
SELECT customer_id,s.plan_id,plan_name,start_date,price FROM subscriptions s 
LEFT JOIN plans p on 
s.plan_id = p.plan_id
WHERE customer_id IN  (1,2,11,13,15,16,18,19)
ORDER BY customer_id,plan_id
```

| customer_id | plan_id | plan_name     | start_date | price  |
| ----------- | ------- | ------------- | ---------- | ------ |
| 1           | 0       | trial         | 2020-08-01 | 0.00   |
| 1           | 1       | basic monthly | 2020-08-08 | 9.90   |
| 2           | 0       | trial         | 2020-09-20 | 0.00   |
| 2           | 3       | pro annual    | 2020-09-27 | 199.00 |
| 11          | 0       | trial         | 2020-11-19 | 0.00   |
| 11          | 4       | churn         | 2020-11-26 |        |
| 13          | 0       | trial         | 2020-12-15 | 0.00   |
| 13          | 1       | basic monthly | 2020-12-22 | 9.90   |
| 13          | 2       | pro monthly   | 2021-03-29 | 19.90  |
| 15          | 0       | trial         | 2020-03-17 | 0.00   |
| 15          | 2       | pro monthly   | 2020-03-24 | 19.90  |
| 15          | 4       | churn         | 2020-04-29 |        |
| 16          | 0       | trial         | 2020-05-31 | 0.00   |
| 16          | 1       | basic monthly | 2020-06-07 | 9.90   |
| 16          | 3       | pro annual    | 2020-10-21 | 199.00 |
| 18          | 0       | trial         | 2020-07-06 | 0.00   |
| 18          | 2       | pro monthly   | 2020-07-13 | 19.90  |
| 19          | 0       | trial         | 2020-06-22 | 0.00   |
| 19          | 2       | pro monthly   | 2020-06-29 | 19.90  |
| 19          | 3       | pro annual    | 2020-08-29 | 199.00 |

---
### Explanation 

**Customer ID 1:** Initially started with a 7-day free trial and downgraded to Basic Monthly after the free trial.

**Customer ID 2:** Initially started with a 7-day free trial and upgraded to the Pro Annual plan after the trial period.

**Customer ID 11:** Initially started with a 7-day free trial and canceled the plan after the 7-day trial.

**Customer ID 13:** Initially started with a 7-day free trial, then downgraded to the Basic Monthly plan for a week, then upgraded to Pro Monthly.

**Customer ID 15:** Initially started with a 7-day free trial, and the plan automatically upgraded to Pro Monthly as the customer did not cancel, downgrade, or upgrade; five days later, the customer canceled the subscription.

**Customer ID 16:** Initially started with a 7-day free trial, then downgraded to the Basic Monthly plan, stayed on the same plan for 4 months, then upgraded to the Pro Annual plan.

**Customer ID 18:** Initially started with a 7-day free trial, and the plan automatically upgraded to Pro Monthly as the customer did not cancel, downgrade, or upgrade.

**Customer ID 19:** Initially started with a 7-day free trial, and the plan automatically upgraded to Pro Monthly as the customer did not cancel, downgrade, or upgrade, stayed on the same plan for 2 months before upgrading to the Pro Annual plan.


---

## B. Data Analysis Questions 


**Question 1:** How many customers has Foodie-Fi ever had?

---

## SQL Code

```sql
SELECT COUNT(DISTINCT customer_id) as total_customers_from_foodie_fi FROM subscriptions;
```
<img width="586" height="146" alt="image" src="https://github.com/user-attachments/assets/076ab413-7d85-4df3-9c4e-003ed27a346f" />

---


**Question 2 :** What is the monthly distribution of trial plan start_date values for our dataset - use the start of the month as the group by value

---

## SQL Code

```sql
SELECT DATE_TRUNC('month' , start_date) :: DATE as month_start_date  , COUNT(customer_id)  as  customer_count_monthly_distribution 
FROM subscriptions                                                                  
WHERE plan_id = 0                                                                   
GROUP BY month_start_date 
ORDER BY month_start_date
```

<img width="1089" height="700" alt="image" src="https://github.com/user-attachments/assets/eb873a76-120b-4f5a-bfbe-7d5d53a04a23" />

---


**Question 3 :** What plan start_date values occur after the year 2020 for our dataset? Show the breakdown by count of events for each plan_name

---

## SQL Code

```sql
SELECT plan_name, COUNT(s.plan_id) as plan_count FROM subscriptions s 
LEFT JOIN plans p ON s.plan_id = p.plan_id
WHERE  EXTRACT (YEAR FROM start_date ) != 2020 
GROUP BY plan_name
ORDER BY plan_count ASC
```
<img width="1268" height="336" alt="image" src="https://github.com/user-attachments/assets/a1e9813b-d5da-4d4e-9c49-cdc462fc60aa" />

---

**Question 4 :** What is the customer count and percentage of customers who have churned rounded to 1 decimal place?

---

## SQL Code

```sql
SELECT 
COUNT(DISTINCT customer_id)  as total_customer_count , 
COUNT(DISTINCT CASE WHEN plan_id = 4 THEN customer_id END)  as customer_churn_count ,
ROUND ((COUNT(DISTINCT CASE WHEN plan_id = 4 THEN customer_id END) :: DECIMAL  / COUNT(DISTINCT customer_id)) *100 , 1 )  as churn_percentage
FROM subscriptions
```

<img width="1502" height="188" alt="image" src="https://github.com/user-attachments/assets/8171d4e9-8032-43f3-8fb0-3f9c1285d45f" />

---

**Question 5 :** How many customers have churned straight after their initial free trial - what percentage is this rounded to the nearest whole number?

---

## SQL Code

```sql
--To tackle this case, I used the LAG window function to check the previous row against the current row. Specifically, I examined whether plan_id = 4 had an immediately preceding row with plan_id = 0. That is how I 
--identified and counted the customers who churned straight after the initial free trial.

WITH cte AS 
(
SELECT customer_id, plan_id,start_date,
LAG(plan_id) OVER(PARTITION BY customer_id ORDER BY start_date) as previous_plan_id 
FROM subscriptions
ORDER BY customer_id,plan_id
)

SELECT COUNT(customer_id) as count_of_customers_churned ,
ROUND((COUNT(customer_id) :: DECIMAL  / (SELECT COUNT(DISTINCT customer_id) FROM subscriptions ) )  * 100 ) as percentage
FROM cte 
WHERE  plan_id =4 AND  previous_plan_id = 0

```
<img width="313" height="147" alt="image" src="https://github.com/user-attachments/assets/c61c1740-4360-4c73-8c9a-7f46624d5e8d" />
<img width="1456" height="140" alt="image" src="https://github.com/user-attachments/assets/a8704b6c-fd12-4c0b-9ea3-d09e115e70ce" />
<img width="1596" height="833" alt="image" src="https://github.com/user-attachments/assets/3fd36d0b-59fe-4f3c-8c03-d42aa688eff7" />

---



**Question 6 :** What is the number and percentage of customer plans after their initial free trial?

---

## SQL Code

```sql
WITH customer_plans AS
(
SELECT  customer_id,s.plan_id,plan_name,
LAG(s.plan_id) OVER(partition by customer_id  ORDER BY s.plan_id) AS previous_plan,
LEAD(s.plan_id) OVER(partition by customer_id  ORDER BY s.plan_id) AS next_plan
FROM subscriptions s 
LEFT JOIN plans p ON s.plan_id = p.plan_id
ORDER BY customer_id
)

-- SELECT COUNT(plan_id) as customer_plan_count_after_initial_free_trail
-- FROM customer_plans 
-- WHERE plan_id != 0

-- They asked for the count of post trail plans and how they are segregated. The total rows in subscriptions table is 2065 and  every customer has initial trail plan which amounts to 1000 so the remaining are the post trail plans which is 1065 

SELECT plan_id,plan_name, COUNT(plan_id) as customer_plan_count_after_initial_free_trail ,

ROUND ((COUNT(plan_id) :: DECIMAL / (SELECT COUNT(plan_id) as customer_plan_count_after_initial_free_trail FROM customer_plans WHERE plan_id != 0  ) ) * 100 ,2 ) AS plan_percentage

FROM customer_plans 
WHERE previous_plan IS NOT NULL  
GROUP BY plan_name,plan_id
ORDER BY plan_id

```
Customer Plans CTE
<img width="1889" height="821" alt="image" src="https://github.com/user-attachments/assets/dacbf638-81f7-48bc-a60f-f396a1bd6812" />

<img width="466" height="198" alt="image" src="https://github.com/user-attachments/assets/1cf7cb5a-e440-4659-be4a-ea9e388c69cf" />

<img width="1523" height="299" alt="image" src="https://github.com/user-attachments/assets/e8849d13-5bfe-4444-ab07-84e36f8f4d74" />

---

**Question 7 :** What is the customer count and percentage breakdown of all 5 plan_name values at 2020-12-31?

---

## SQL Code

```sql
WITH customer_plans AS
(
SELECT  customer_id,s.plan_id,plan_name,start_date,
ROW_NUMBER() OVER (partition by customer_id ORDER BY start_date DESC ) as row_number
FROM subscriptions s 
LEFT JOIN plans p ON s.plan_id = p.plan_id
WHERE start_date <= '2020-12-31'

)

-- in the CTE we used row number window function to check for the latest date before Dec 31  , 2020 for when the subscription is active
--SELECT * FROM customer_plans ORDER BY customer_id

SELECT plan_id,plan_name, COUNT(customer_id) ,
ROUND((COUNT(customer_id) :: DECIMAL / (SELECT COUNT(DISTINCT customer_id) FROM subscriptions ) ) * 100 , 2)  AS percentage_breakdown
FROM customer_plans
WHERE row_number =1 
GROUP BY plan_id,plan_name 
ORDER BY plan_id
```
<img width="1664" height="366" alt="image" src="https://github.com/user-attachments/assets/a6e3768f-adde-4979-b424-33df57044a79" />

---



**Question 8 :** How many customers have upgraded to an annual plan in 2020?

---

## SQL Code

```sql
SELECT COUNT(customer_id) as customers_upgraded_to_annual_plan FROM 

(
SELECT  customer_id,plan_id,start_date FROM subscriptions 
WHERE plan_id = 3 AND  EXTRACT(YEAR FROM start_date )=2020
ORDER BY customer_id
)  s
```

<img width="404" height="151" alt="image" src="https://github.com/user-attachments/assets/45c29cce-a84a-4739-a509-71e79a397139" />

---




**Question 9 :** How many days on average does it take for a customer to an annual plan from the day they join Foodie-Fi?
---

## SQL Code

```sql
--- The idea was to create two ctes one with trail plan start date and the other with annual plan start date , join those two ctes , do the date difference and then calculate the average days 
WITH trial_cte as 
(
SELECT customer_id, s.plan_id,plan_name, start_date as trial_start_date
  FROM 
subscriptions s LEFT JOIN plans p ON
s.plan_id = p.plan_id
WHERE s.plan_id = 0
ORDER BY customer_id , plan_id
),

annual_cte as 
(
SELECT customer_id, s.plan_id,plan_name, start_date as annual_plan_start_date
 FROM 
subscriptions s LEFT JOIN plans p ON
s.plan_id = p.plan_id
WHERE s.plan_id = 3
ORDER BY customer_id , plan_id
),

joined_cte AS
(
SELECT tc.customer_id, trial_start_date,annual_plan_start_date,
(annual_plan_start_date - trial_start_date ) AS days_taken_to_join_annual_plan
FROM trial_cte tc
INNER JOIN annual_cte ac ON
tc.customer_id = ac.customer_id
)


SELECT ROUND(AVG(days_taken_to_join_annual_plan)) as AVG_Days_taken_to_Join_Annual_plan FROM joined_cte

```
---
```sql
-- MIN - Logic - cleaner 
WITH CTE AS
(
SELECT customer_id,
-- here we are picking the earliest trial date and annual plan date hence we use min 
MIN(CASE WHEN plan_id=0 THEN start_date END ) as trial_date,  
MIN (CASE WHEN plan_id=3 THEN start_date END  ) as annual_date
FROM subscriptions
GROUP BY customer_id
ORDER BY customer_id
)


SELECT ROUND(AVG(annual_date - trial_date))
FROM cte
```





<img width="493" height="146" alt="image" src="https://github.com/user-attachments/assets/3186ae29-d13a-4c9f-a5a9-45552d7dd670" />


---


**Question 10 :** Can you further breakdown this average value into 30 day periods (i.e. 0-30 days, 31-60 days etc)

---

## SQL Code

```sql
WITH cte AS 
( SELECT customer_id, 
 MIN(CASE WHEN plan_id = 0 THEN start_date END ) AS trial_date,
 MIN(CASE WHEN plan_id = 3 THEN start_date END ) AS annual_date 
 FROM subscriptions GROUP BY customer_id 
 ORDER BY customer_id 
),


days_taken  AS 
(
SELECT *, (annual_date - trial_date ) AS days_taken_to_upgrade FROM  cte
WHERE trial_date IS NOT NULL AND annual_date IS NOT NULL
),


-- Here we are dividing the days taken to upgrade to different buckets  for instance 0- 30 days bucket is 0 
-- 31-60 days is bucket 1 etc  , this is acheiveved by diving the days taken to uprade , divide it by 30 then floor ( rounds down to nearest whole number )  

buckets_cte AS 
(

  SELECT customer_id, days_taken_to_upgrade, floor(days_taken_to_upgrade/30) as bucket 
  FROM days_taken

)


SELECT    (bucket*30 + 1   || '-' || bucket*30 + 30) as period ,bucket , COUNT(customer_id) as customers_count , ROUND(AVG(days_taken_to_upgrade)) as avg_days_to_upgrade
FROM
buckets_cte
GROUP BY  bucket
ORDER BY bucket 
  
```
<img width="1646" height="732" alt="image" src="https://github.com/user-attachments/assets/58a9e91b-77b0-4bbb-85fd-de320c00664e" />

---

**Question 11 :** How many customers downgraded from a pro monthly to a basic monthly plan in 2020?

---

## SQL Code

```sql
WITH CTE as 
(
SELECT customer_id, s.plan_id,plan_name ,
LEAD(s.plan_id) OVER(partition by customer_id  ORDER BY s.plan_id ASC) as next_plan 
FROM subscriptions s LEFT JOIN plans p
ON s.plan_id = p.plan_id
WHERE EXTRACT(YEAR FROM start_date) = 2020
ORDER BY customer_id,s.plan_id
)

select COUNT(customer_id) as customers_downgraded FROM CTE  WHERE plan_id = 2 AND next_plan = 1 

```

<img width="404" height="189" alt="image" src="https://github.com/user-attachments/assets/48f9136d-9802-4f3f-9b4f-9aba7a197700" />


