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
	(CASE WHEN timeonsite IS NULL THEN '00:00:00'                                                                     --accurate time can be generated this way
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
	ELSE TO_CHAR((timeonsite || ' second')::interval, 'HH24:MI:SS') END) AS timeonsite, --this is a better way to do time on site, 
	pageviews,                                                                         -- if I didn't switch it on analytics I should
	type exec_type, 					--since type is a keyword, felt that changing it would be a good idea
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
~~~~
Regarding products and product related tables:
				
products - 7 columns, 1092 rows, each has unique productsku, nulls in sentiment_score and sentiment_magnitude
sales_by_sku - 2 columns, 462 rows, each unique with productsku and total_ordered
sales_report - 8 columns, match products except also has 'ratio'
				454 rows, each unique, contained in sales_by_sku AND duplicates with products, 
				FULL OUTER JOINING with products does not increase the number of rows so details match
				RE: the 8 extra SKUs in sales_by_sku NOT IN prodcuts or sales_report
				ONE of these skus matches a sale in the all_sessions and so must be included
				
Cleaning products, sales_by_sku and sales_report was MUCH more straightforward, the only sticking point being that sales_by_sku contains 8 additional skus

~~~~sql
SELECT s.productsku, s.total_ordered, sr.total_ordered	--shows that both tables match on total ordered, except for the 8 extra from sales_by_sku
FROM sales_by_sku s					--running again with the commented line in confirms that details are the same, no rows added
FULL OUTER JOIN sales_report sr
	ON sr.productsku = s.productsku
--WHERE sr.total_ordered <> s.total_ordered

SELECT p.productsku, p.productname, sr.productname 	-- also, sku and name match across products and sales_report
FROM products p
JOIN sales_report sr
	ON sr.productsku = p.productsku
--WHERE sr.productname <> p.productname

~~~~
Adding in the SKUs listed in the cleaned up all_sessions 

The rough part was trying to sort out the categories and skus, since there were a LOT of varieties associated with a single sku through the all_sessions table.
What I ended up doing was iterating between a temp table, viewing it, running it back through with a different REPLACE or TRIM function targeting a specific duplicate, or DELETE on a specific field, and saving my progress by incrementing the name.

This was not fun.
Here is a scrap of it I saved. Was probably not worth the effort.
Finally just created a 'categories' table with the results of my last iteration. For some reason I started working from the top at one point, but my workflow was good.
~~~~sql
CREATE TABLE categories AS
SELECT DISTINCT 
	productsku, 
	REPLACE(product_category,'Lifestyle/','') product_category
FROM prod15
ORDER BY productsku

SELECT DISTINCT *
FROM prod15
ORDER BY productsku

UPDATE prod15
SET product_category = 'Accessories'
WHERE productsku = 'GGOEGAAX0213'

DELETE FROM prod15
WHERE product_category IN ('(notset)')

SELECT *
FROM products
WHERE productsku = 'GGOEAOCB077499'
~~~~
