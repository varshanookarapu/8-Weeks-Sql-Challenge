## 4. Campaigns Analysis


Generate a table that has 1 single row for every unique visit_id record and has the following columns:

user_id
visit_id
visit_start_time: the earliest event_time for each visit
page_views: count of page views for each visit
cart_adds: count of product cart add events for each visit
purchase: 1/0 flag if a purchase event exists for each visit
campaign_name: map the visit to a campaign if the visit_start_time falls between the start_date and end_date
impression: count of ad impressions for each visit
click: count of ad clicks for each visit
(Optional column) cart_products: a comma separated text value with products added to the cart sorted by the order they were added to the cart (hint: use the sequence_number)
Use the subsequent dataset to generate at least 5 insights for the Clique Bait team - bonus: prepare a single A4 infographic that the team can use for their management reporting sessions, be sure to emphasise the most important points from your findings.

---

```sql
SELECT user_id,visit_id,MIN(event_time) as visit_start_time , 

SUM(CASE WHEN e.event_type = 1 THEN 1 ELSE 0 END) as page_views,
SUM(CASE WHEN e.event_type = 2 THEN 1 ELSE 0 END) as cart_adds,
SUM(CASE WHEN e.event_type = 3 THEN 1 ELSE 0 END) as purchase,
campaign_name,
SUM(CASE WHEN e.event_type=4 THEN 1 ELSE 0 END) as impression,
SUM(CASE WHEN e.event_type=5 THEN 1 ELSE 0 END) as click,

STRING_AGG(CASE WHEN  e.event_type =2 THEN page_name ELSE Null END , ' , ' ORDER BY sequence_number) AS  cart_products


FROM clique_bait.events e LEFT JOIN clique_bait.users u ON e.cookie_id = u.cookie_id 
LEFT JOIN clique_bait.event_identifier ei ON e.event_type=ei.event_type
LEFT JOIN clique_bait.campaign_identifier ci ON e.event_time BETWEEN ci.start_date AND ci.end_date
LEFT JOIN clique_bait.page_hierarchy ph ON e.page_id = ph.page_id
GROUP BY user_id,visit_id,campaign_name
```
<img width="1919" height="429" alt="image" src="https://github.com/user-attachments/assets/592242fd-8b62-401d-a9a9-b7eb7ffba75a" />
<img width="1908" height="329" alt="image" src="https://github.com/user-attachments/assets/6b36ef79-bae3-4118-affc-7452947e9a94" />
<img width="1871" height="417" alt="image" src="https://github.com/user-attachments/assets/82e5b6cc-86a6-4412-b266-3b97bf921740" />


---
Some ideas you might want to investigate further include:

Identifying users who have received impressions during each campaign period and comparing each metric with other users who did not have an impression event

```sql
SELECT campaign_name, 
COUNT(DISTINCT CASE WHEN impression =1 THEN user_id END) as users_with_impressions_count ,
COUNT(DISTINCT CASE WHEN impression =0 THEN user_id END) as users_without_impressions_count
FROM campaign_analysis 
GROUP BY campaign_name

SELECT campaign_name, COUNT(DISTINCT user_id) as user_count, SUM(page_views) as page_view_count , SUM(cart_adds) as cart_add_count , SUM(purchase) as purchase_count
FROM campaign_analysis 
WHERE impression =1
GROUP BY campaign_name


SELECT campaign_name, COUNT(DISTINCT user_id) as user_count, SUM(page_views) as page_view_count , SUM(cart_adds) as cart_add_count , SUM(purchase) as purchase_count
FROM campaign_analysis 
WHERE impression =0
GROUP BY campaign_name
```
<img width="1500" height="254" alt="image" src="https://github.com/user-attachments/assets/d7f52c3b-2857-48d9-aead-c996bae82698" />
<img width="1665" height="267" alt="image" src="https://github.com/user-attachments/assets/a2059988-4443-4e7e-b3d6-1143c90cc3e8" />
<img width="1690" height="243" alt="image" src="https://github.com/user-attachments/assets/6c458d19-8c5f-4055-bc4a-64b3c08e4819" />
---

Does clicking on an impression lead to higher purchase rates?

```sql

WITH campaign_analysis AS (

SELECT user_id,visit_id,MIN(event_time) as visit_start_time , 

SUM(CASE WHEN e.event_type = 1 THEN 1 ELSE 0 END) as page_views,
SUM(CASE WHEN e.event_type = 2 THEN 1 ELSE 0 END) as cart_adds,
SUM(CASE WHEN e.event_type = 3 THEN 1 ELSE 0 END) as purchase,
campaign_name,
SUM(CASE WHEN e.event_type=4 THEN 1 ELSE 0 END) as impression,
SUM(CASE WHEN e.event_type=5 THEN 1 ELSE 0 END) as click,

STRING_AGG(CASE WHEN  e.event_type =2 THEN page_name ELSE Null END , ' , ' ORDER BY sequence_number) AS  cart_products


FROM clique_bait.events e LEFT JOIN clique_bait.users u ON e.cookie_id = u.cookie_id 
LEFT JOIN clique_bait.event_identifier ei ON e.event_type=ei.event_type
LEFT JOIN clique_bait.campaign_identifier ci ON e.event_time BETWEEN ci.start_date AND ci.end_date
LEFT JOIN clique_bait.page_hierarchy ph ON e.page_id = ph.page_id
GROUP BY user_id,visit_id,campaign_name
  
  )
  
SELECT campaign_name, 

ROUND(100* COUNT(DISTINCT CASE WHEN click =1 AND purchase =1 THEN user_id END) / COUNT(DISTINCT CASE WHEN click =1 THEN user_id END) ,2) as purchase_rate_when_clicked,

ROUND(100* COUNT(DISTINCT CASE WHEN click =0 AND purchase =1 THEN user_id END) / COUNT(DISTINCT CASE WHEN click =0 THEN user_id END) ,2) as purchase_rate_when_not_clicked
FROM campaign_analysis
WHERE campaign_name IS NOT NULL
GROUP BY campaign_name
```
After comparing the purchase rates when clicked vs when not clicked we see that the purchase rates are higher when clicked. Hence clicking on an impression does lead to higher purchase rates
<img width="1530" height="249" alt="image" src="https://github.com/user-attachments/assets/8b34d4ff-d24c-482b-b1ee-02901a436b21" />

What is the uplift in purchase rate when comparing users who click on a campaign impression versus users who do not receive an impression? What if we compare them with users who just an impression but do not click?
Uplift = the improvement in an outcome between two groups
```sql

SELECT campaign_name, 

ROUND(100* COUNT(DISTINCT CASE WHEN click =1 AND purchase =1 THEN user_id END) / COUNT(DISTINCT CASE WHEN click =1 THEN user_id END) ,2) as purchase_rate_when_clicked,

ROUND(100* COUNT(DISTINCT CASE WHEN click =0 AND purchase =1 THEN user_id END) / COUNT(DISTINCT CASE WHEN click =0 THEN user_id END) ,2) as purchase_rate_when_not_clicked ,

(ROUND(100* COUNT(DISTINCT CASE WHEN click =1 AND purchase =1 THEN user_id END) / COUNT(DISTINCT CASE WHEN click =1 THEN user_id END) ,2) - ROUND(100* COUNT(DISTINCT CASE WHEN click =0 AND purchase =1 THEN user_id END) / COUNT(DISTINCT CASE WHEN click =0 THEN user_id END) ,2)) as uplift


FROM campaign_analysis
WHERE campaign_name IS NOT NULL
GROUP BY campaign_name
```
<img width="1673" height="215" alt="image" src="https://github.com/user-attachments/assets/4e176423-48e7-4ed5-bb2e-d67f3b21ffa5" />

What metrics can you use to quantify the success or failure of each campaign compared to eachother?

Total purchases, purchase rate and uplift
