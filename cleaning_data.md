What issues will you address by cleaning the data?

The two biggest ones will be data normalization and removing duplicates.
This is especially true for the columns that appear unformatted, such as prices, and columns which may represent intervals or times.



Queries:
Below, provide the SQL queries you used to clean your data.

--first, generate a temp table of all fields in the analytics csv
~~~sql
CREATE TEMP TABLE clean_analytics AS (
SELECT 
	DISTINCT visitid AS sessionid, --distinct is called to reduce the number of duplicates, visitid renamed to sessionid to try to differenciate from other fields
	TO_TIMESTAMP(visitid) AS visit_date, --since visitid is generated from the users timestamp (which can be seen when comparing this cast to the date), a more       
	(CASE WHEN timeonsite IS NULL THEN '00:00:00'                                                                       accurate time can be generated this way
	ELSE timeonsite END) timeonsite, 
	visitnumber,
	fullvisitorid AS visitorid, --fullvisitorid was a bit unwieldly, since visitid was changed to session ID, dropped the 'full'.
	channelgrouping,
	socialengagementtype,
	(CASE WHEN bounces IS NULL THEN '0'  -- the following casts are just to remove null values
	ELSE bounces END) bounces,
	(CASE WHEN units_sold IS NULL THEN 0
	ELSE units_sold END) units_sold,
	(CASE WHEN unit_price IS NULL THEN '0' 
	 ELSE ROUND((unit_price/1000000),2) END) AS unit_price, --creates unit_price in a better, more legible format
	(CASE WHEN revenue IS NULL THEN '0' 
	ELSE ROUND((revenue/1000000),2) END) AS revenue --same
FROM analytics
ORDER BY sessionid)
~~~~
Call this whole table, examine the results, then move on to all_sessions. 32 columns, lots of data
~~~sql
CREATE TEMP TABLE clean_sessions AS
(
	SELECT 
	DISTINCT visitid AS sessionid, --first few are formatted to match changes from analytics
	fullVisitorId visitorid,
	TO_TIMESTAMP(visitid) AS visit_date,
	channelgrouping,
	(CASE WHEN country = '(not set)' THEN 'N/A'
	 ELSE country END),
	city,
	(CASE WHEN timeonsite IS NULL THEN '0'
	ELSE TO_CHAR((timeonsite || ' second')::interval, 'HH24:MI:SS') END) AS timeonsite, --this is a better way to do time on site, if I didn't switch it on analytics
	pageviews,                                                                                                                                      I should
	type exec_type, --since type is a keyword, felt that changing it would be a good idea
	(CASE WHEN productQuantity IS NULL THEN '0' --CASE used to reduce nulls
	ELSE productQuantity END) AS product_quantity,
	(CASE WHEN productPrice IS NULL THEN '0'
	ELSE (ROUND((productPrice/1000000),2)) END) as unit_price,
	(CASE WHEN productrevenue IS NULL THEN '0'
	ELSE (ROUND((productrevenue/1000000),2)) END) AS product_revenue ,
	productsku,
	ProductName product_name,
	v2ProductCategory product_category, --to match the categories existing in products
	productvariant,
	(CASE WHEN currencycode IS NULL then 'unspecified'
	ELSE currencycode END) AS currencycode,
	(CASE WHEN totalTransactionRevenue IS NULL THEN '0'
	ELSE (ROUND((totalTransactionRevenue/1000000),2)) END) transaction_revenue,
	(CASE WHEN transactions IS NULL THEN '0'
	ELSE transactions END) AS transactions,
	pageTitle,
	pagePathLevel1 page_path_level,
	eCommerceAction_type,
	eCommerceAction_step,
	(CASE WHEN eCommerceAction_option IS NULL THEN 'N/A'
	 ELSE eCommerceAction_option END) AS eCommerceAction_option
FROM all_sessions
ORDER BY visitid DESC
)

