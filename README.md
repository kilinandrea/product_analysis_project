## Introduction
This is part of a product analysis project to evaluate users' time spent on an ecommerce website.
I am to analyse a specfic subset of customers: users who are first time visitors and make a purchase on the same day.
_Limitations_: 
- Sessions occurred during midnight are missed.
- Limited to one purchase per day per user.
- Inactive time of a user is calculated in the session duration, which can lead to very long sessions and mislead the results interpretation.

**Two hypotheses were considered:**
- Hypothesis A: time spent in the purchasing funnel has a correlation with the value of the purchase of first-time customers.
- Hypothesis B: Hypothesis B: value of purchase can be forecasted through time spent in the purchasing funnel and devices used by first-time customers.

### Data preparation

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
### Hypothesis A:

Correlation and linear regression -purchase value and time spent in funnel

**Correlation** between spend and time to purchase is 0.042302
This means:
- very weak positive linear relationship
- other factors, not in this correlation, are likely to be more significant in predicting

**Linear regression** analysis shows:
-  a weak R square of 0.001789460942
- a lower than 0.05 P-value for our independent variable of 0.02865186412
- coefficient: an estimated increase of 0.0178 units in the amount spent

**Result**
Looking only at time spent in the purchasing funnel on our website for new customers does not provide any reliable information about customer purchasing behavior nor predictability. 

### Hypothesis B:
Correlation and linear regression -purchase value, time spent in funnel + devices used

**Multiple linear regression** analysis shows:
- the p-value of all independent variables are higher than 0.05, except for ‘time_to_purchase’ which is 0.0410 indicating a relationship between it and the dependent variable
- R square is as low as 0.002 which is only marginally higher than the simple linear regression

Equation: value = -0.0770 + 0.0990 * desktop + 0.0514 * mobile + 0.0404 * time_to_purchase
n = 2563, R-squared = 0.002 

**Result** 
We found that, despite good p-value, the value of purchase doesn’t have a relevant relationship with neither time spent, nor devices used by customers.

### Summary
Analysing this subset of customers, both hypotheses were rejected, there is no relevant relationship between the analyzed variables of purchase spend, time spend and device used.
