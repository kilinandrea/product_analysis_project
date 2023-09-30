###Introduction
This is a product analysis project to identify how much time it takes for a user to make a purchase on our website.
I am to analyse a specfic subset of customers' time spent in the purchase funnel: users who are first time visitors and make a purchase on the same day.

Main areas of analysis:
- Correlation between spend and minutes to purchase
- 

Limitations: 
- Sessions occurred during midnight are missed.
- Limited to one purchase per day per user.
- Inactive time of a user is calculated in the session duration, which can lead to very long sessions and mislead the results interpretation.

query results and subsequent calculations found in https://docs.google.com/spreadsheets/d/1TujZc-2OeMilE6kSKh501uH4i3PduyeyJgucMhvXT4M/edit?usp=sharing


#### SQL code:
``` SQL
WITH
  user_first_touch AS (
  SELECT
    user_pseudo_id,
    MIN(PARSE_DATE('%Y%m%d', event_date)) AS event_date,
    MIN(TIMESTAMP_MICROS(event_timestamp)) AS first_touch_timestamp
  FROM
    `turing_data_analytics.raw_events`
  GROUP BY
    user_pseudo_id ),
  user_first_purchase AS (
  SELECT
    user_pseudo_id,
    purchase_revenue_in_usd AS value,
    country,
    category as device,
    MIN(PARSE_DATE('%Y%m%d', event_date)) AS event_date,
    MIN(TIMESTAMP_MICROS(event_timestamp)) AS first_purchase_timestamp,
    ROW_NUMBER() OVER (PARTITION BY user_pseudo_id ORDER BY TIMESTAMP_MICROS(event_timestamp)) AS purchase_row_number
  FROM
    `turing_data_analytics.raw_events`
  WHERE
    event_name = 'purchase'
  GROUP BY
    user_pseudo_id,
    purchase_revenue_in_usd,
    event_timestamp,
    country,
    device )
SELECT
  uft.user_pseudo_id,
  uft.event_date AS purchase_date,
  ufp.value AS value,
  ufp.device,
  uft.first_touch_timestamp,
  ufp.first_purchase_timestamp,
  DATETIME_DIFF(ufp.first_purchase_timestamp, uft.first_touch_timestamp, MINUTE) AS time_to_purchase_minutes
FROM
  user_first_touch uft
JOIN (
  SELECT
    user_pseudo_id,
    value,
    device,
    event_date,
    first_purchase_timestamp
  FROM
    user_first_purchase
  WHERE
    purchase_row_number = 1 ) ufp
ON
  uft.user_pseudo_id = ufp.user_pseudo_id
  AND uft.event_date = ufp.event_date
ORDER BY
  purchase_date;
```
