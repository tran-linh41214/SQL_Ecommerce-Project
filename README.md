# Explore E-Commerce Dataset with Google Analytics

# 📊 Project Title: [SQL_E-commerce Data]  
Author: [Linh Tran]    
Tools Used: SQL  

## Introduction
Welcome to my first project on e-commerce data exploration! This repository contains the SQL script **"SQL_E-commerce Data"**, which analyzes e-commerce data using Google Analytics samples. The objective is to uncover valuable insights into customer behavior, website performance, and transactional trends.

## 📂File Description
The script "SQL_E-commerce Data" comprises a series of SQL queries targeted at various aspects of e-commerce data analysis:
- **Monthly Trends Analysis:** Queries to scrutinize visits, pageviews, and transactions, segmented by month.
- **Traffic Source Analysis:** In-depth examination of various traffic sources, assessing total visits, bounce rates, and their overall impact on website engagement.
- **Revenue Analysis by Time and Source:** Advanced queries to analyze product revenue based on different time frames (monthly, weekly) and traffic sources. This helps in understanding which sources and time periods are most lucrative for the e-commerce business.

### Total visit, pageview, transaction for Jan, Feb AND March 2017

```sql
--Query 01: calculate total visit, pageview, transaction for Jan, Feb AND March 2017 (order by month)
SELECT
    FORMAT_DATE("%Y%m",PARSE_DATE('%Y%m%d',date)) AS month,
    SUM(totals.visits) AS visits,
    SUM(totals.pageviews) AS pageviews,
    SUM(totals.transactions) AS transactions,
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`
WHERE _table_suffix BETWEEN '0101' AND '0331'
GROUP BY month
ORDER BY month;
```

![image](https://github.com/user-attachments/assets/0136c47c-47bb-4f4e-8e5e-adcc3b823208)

### Bounce rate per traffic source in July 2017

```sql
--Query 02: Bounce rate per traffic source in July 2017 (Bounce_rate = num_bounce/total_visit) (order by total_visit DESC)
SELECT
    trafficSource.source,
    SUM(totals.visits) AS total_visits,
    SUM(totals.bounces) AS total_no_of_bounces,
    ROUND(SUM(totals.bounces)/SUM(totals.visits)*100, 3) AS Bounce_rate
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*` 
Where _table_suffix BETWEEN '0701' AND '0731'
GROUP BY trafficSource.source
ORDER BY total_visits DESC;
```
![image](https://github.com/user-attachments/assets/8a2dad05-d640-481b-b6c4-bf7332d647a3)

### Revenue by traffic source by week, by month in June 2017
```sql
--Query 3: Revenue by traffic source by week, by month in June 2017
with 
month_data as(
  SELECT
    "Month" as time_type,
    format_date("%Y%m", parse_date("%Y%m%d", date)) as month,
    trafficSource.source AS source,
    SUM(p.productRevenue)/1000000 AS revenue
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201706*`,
    unnest(hits) hits,
    unnest(product) p
  WHERE p.productRevenue is not null
  GROUP BY 1,2,3
  order by revenue DESC
),

week_data as(
  SELECT
    "Week" as time_type,
    format_date("%Y%W", parse_date("%Y%m%d", date)) as week,
    trafficSource.source AS source,
    SUM(p.productRevenue)/1000000 AS revenue
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201706*`,
    unnest(hits) hits,
    unnest(product) p
  WHERE p.productRevenue is not null
  GROUP BY 1,2,3
  order by revenue DESC
)

select * from month_data
union all
select * from week_data
order by time_type;
```
![image](https://github.com/user-attachments/assets/6fbd89be-b618-4532-9025-4f4512592551)

### Average number of pageviews by purchaser type (purchasers vs non-purchasers) in June, July 2017.
```sql
-- Query 04: Average number of pageviews by purchaser type (purchasers vs non-purchasers) in June, July 2017.
WITH 
purchaser_data AS (
  SELECT
      FORMAT_DATE("%Y%m", PARSE_DATE("%Y%m%d", date)) AS month,
      SUM(totals.pageviews) / COUNT(DISTINCT fullvisitorid) AS avg_pageviews_purchase
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`,
       UNNEST(hits) hits,
       UNNEST(product) product
  WHERE _table_suffix BETWEEN '0601' AND '0731'
    AND totals.transactions >= 1
    AND product.productRevenue IS NOT NULL
  GROUP BY month
),
non_purchaser_data AS (
  SELECT
      FORMAT_DATE("%Y%m", PARSE_DATE("%Y%m%d", date)) AS month,
      SUM(totals.pageviews) / COUNT(DISTINCT fullvisitorid) AS avg_pageviews_non_purchase
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`,
       UNNEST(hits) hits,
       UNNEST(product) product
  WHERE _table_suffix BETWEEN '0601' AND '0731'
    AND (totals.transactions IS NULL OR totals.transactions = 0)
    AND product.productRevenue IS NULL
  GROUP BY month
)
SELECT
    COALESCE(pd.month, npd.month) AS month,  -- Đảm bảo không bị mất dữ liệu nếu chỉ có non-purchase hoặc purchase trong tháng đó
    avg_pageviews_purchase,
    avg_pageviews_non_purchase
FROM purchaser_data pd
FULL JOIN non_purchaser_data npd
ON pd.month = npd.month
ORDER BY month;
```

![image](https://github.com/user-attachments/assets/e6e1cc27-8e06-44ef-8c4b-e646397cfb4c)

### Average number of transactions per user that made a purchase in July 2017
```sql
--Query 05: Average number of transactions per user that made a purchase in July 2017
SELECT
    FORMAT_DATE("%Y%m",PARSE_DATE('%Y%m%d',date)) AS month,
    SUM(totals.transactions)/COUNT(DISTINCT fullVisitorid) AS avg_total_transactions_per_user
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`
    ,UNNEST (hits) hits,
    UNNEST(product) product
Where 
    _table_suffix BETWEEN '0701' AND '0731'
    AND totals.transactions >= 1
    AND totals.totalTransactionRevenue IS NOT NULL
    AND product.productRevenue IS NOT NULL
GROUP BY month;
```

![image](https://github.com/user-attachments/assets/840010ca-056a-4459-98b3-fa609470fa8e)

### Average amount of money spent per session. Only include purchaser data in July 2017

```sql
--Query 06: Average amount of money spent per session. Only include purchaser data in July 2017
SELECT 
  FORMAT_DATE('%Y%m', PARSE_DATE('%Y%m%d', date)) month,
  ROUND((SUM(product.productRevenue) / SUM(totals.visits))/1000000,2) avg_revenue_by_user_per_visit
FROM 
  `bigquery-public-data.google_analytics_sample.ga_sessions_2017*`, 
  UNNEST(hits) hits, 
  UNNEST(hits.product) product
WHERE 
  _TABLE_SUFFIX BETWEEN '0701' AND '0731'
  AND product.productRevenue IS NOT NULL
  AND totals.transactions IS NOT NULL
GROUP BY month;
```

![image](https://github.com/user-attachments/assets/2940a4cb-0f21-468d-a0d2-987b207efd68)

### Other products purchased by customers who purchased product "YouTube Men's Vintage Henley" in July 2017.

```sql
--Query 07: Other products purchased by customers who purchased product "YouTube Men's Vintage Henley" in July 2017. Output should show product name and the quantity was ordered.
with buyer_list as(
    SELECT
        distinct fullVisitorId
    FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*`
    , UNNEST(hits) AS hits
    , UNNEST(hits.product) as product
    WHERE product.v2ProductName = "YouTube Men's Vintage Henley"
    AND totals.transactions>=1
    AND product.productRevenue is not null
)

SELECT
  product.v2ProductName AS other_purchased_products,
  SUM(product.productQuantity) AS quantity
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_201707*`
, UNNEST(hits) AS hits
, UNNEST(hits.product) as product
JOIN buyer_list using(fullVisitorId)
WHERE product.v2ProductName != "YouTube Men's Vintage Henley"
 and product.productRevenue is not null
GROUP BY other_purchased_products
ORDER BY quantity DESC;
```

![image](https://github.com/user-attachments/assets/f3f664fc-9fc9-45e0-afc1-1bf258043f96)

### Cohort map from product view to addtocart to purchase in Jan, Feb and March 2017.

```sql
--"Query 08: Calculate cohort map from product view to addtocart to purchase in Jan, Feb and March 2017. For example, 100% product view then 40% add_to_cart and 10% purchase.

--Cách 1:dùng CTE
with
product_view as(
  SELECT
    format_date("%Y%m", parse_date("%Y%m%d", date)) as month,
    count(product.productSKU) as num_product_view
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_*`
  , UNNEST(hits) AS hits
  , UNNEST(hits.product) as product
  WHERE _TABLE_SUFFIX BETWEEN '20170101' AND '20170331'
  AND hits.eCommerceAction.action_type = '2'
  GROUP BY 1
),

add_to_cart as(
  SELECT
    format_date("%Y%m", parse_date("%Y%m%d", date)) as month,
    count(product.productSKU) as num_addtocart
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_*`
  , UNNEST(hits) AS hits
  , UNNEST(hits.product) as product
  WHERE _TABLE_SUFFIX BETWEEN '20170101' AND '20170331'
  AND hits.eCommerceAction.action_type = '3'
  GROUP BY 1
),

purchase as(
  SELECT
    format_date("%Y%m", parse_date("%Y%m%d", date)) as month,
    count(product.productSKU) as num_purchase
  FROM `bigquery-public-data.google_analytics_sample.ga_sessions_*`
  , UNNEST(hits) AS hits
  , UNNEST(hits.product) as product
  WHERE _TABLE_SUFFIX BETWEEN '20170101' AND '20170331'
  AND hits.eCommerceAction.action_type = '6'
  and product.productRevenue is not null   
  group by 1
)

select
    pv.*,
    num_addtocart,
    num_purchase,
    round(num_addtocart*100/num_product_view,2) as add_to_cart_rate,
    round(num_purchase*100/num_product_view,2) as purchase_rate
from product_view pv
left join add_to_cart a on pv.month = a.month
left join purchase p on pv.month = p.month
order by pv.month;
```
```sql
--Cách 2: Dùng count(case when) hoặc sum(case when)

with product_data as(
select
    format_date('%Y%m', parse_date('%Y%m%d',date)) as month,
    count(CASE WHEN eCommerceAction.action_type = '2' THEN product.v2ProductName END) as num_product_view,
    count(CASE WHEN eCommerceAction.action_type = '3' THEN product.v2ProductName END) as num_add_to_cart,
    count(CASE WHEN eCommerceAction.action_type = '6' and product.productRevenue is not null THEN product.v2ProductName END) as num_purchase
FROM `bigquery-public-data.google_analytics_sample.ga_sessions_*`
,UNNEST(hits) as hits
,UNNEST (hits.product) as product
where _table_suffix between '20170101' and '20170331'
and eCommerceAction.action_type in ('2','3','6')
group by month
order by month
)

select
    *,
    round(num_add_to_cart/num_product_view * 100, 2) as add_to_cart_rate,
    round(num_purchase/num_product_view * 100, 2) as purchase_rate
from product_data;
```

![image](https://github.com/user-attachments/assets/5d768822-11a6-4291-9615-624175974681)
