What issues will you address by cleaning the data?

The two biggest ones will be data normalization and removing duplicates.
This is especially true for the columns that appear unformatted, such as prices, and columns which may represent intervals or times.



Queries:
Below, provide the SQL queries you used to clean your data.

--first, generate a temp table of all fields in the analytics csv
~~~sql
CREATE TEMP TABLE clean_analytics AS (
SELECT DISTINCT			
	visitid AS sessionid, --visitid renamed to sessionid to try to differenciate from other column headers
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
Call this whole table, examine the results checking for missed nulls, unusual looking information, etc.
Then move on to all_sessions. 32 columns, lots of data.

While committed to preserving as much data as possible, the column sessionqualitydim contains 13,906 NULL entries and will be discarded. 

Since the 'time' column contains LESS information than the visitorid (when CAST as a timestmp), that column is also discarded.

'productrefundamount', 'itemquantity', 'itemrevenue', 'productrefundamount', and 'searchkeyword' are entirely NULL and discarded.

'transactionid' is NULL in all but 9 cases ONLY two of which appear in analytics with units_order positive
~~~sql
  		SELECT se.transactionid, se.fullvisitorid, se.visitid, COUNT(*)
		FROM all_sessions se
		JOIN analytics a
			ON se.fullvisitorid = a.fullvisitorid 
			AND
			se.visitid = a.visitid
		WHERE transactionid IS NOT NULL
			AND a.units_sold > 0
		GROUP BY se.transactionid, se.fullvisitorid, se.visitid
		ORDER BY transactionid
~~~
and so is discarded as well

The results with certain transformations will be:

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
	ONE of these skus matches a sale recorded in the all_sessions and so must be included
				
Cleaning products, sales_by_sku and sales_report was MUCH more straightforward, the only sticking point being that sales_by_sku contains 8 additional skus

~~~~sql
SELECT s.productsku, s.total_ordered, sr.total_ordered	--shows that both tables match on total ordered, except for the 8 extra from sales_by_sku
FROM sales_by_sku s					--running again with the commented line in confirms that details are the same, no rows added
FULL OUTER JOIN sales_report sr
	ON sr.productsku = s.productsku
--WHERE sr.total_ordered <> s.total_ordered  		--this commented-out line can be deployed to poll results where the total_ordered does not match

SELECT p.productsku, p.productname, sr.productname 	-- also, sku and name match across products and sales_report
FROM products p
JOIN sales_report sr
	ON sr.productsku = p.productsku
--WHERE sr.productname <> p.productname			--this commented-out line can be deployed to poll results where the total_ordered does not match
~~~~
Examining the data:
~~~~sql
SELECT DISTINCT productsku
FROM products		--returns 1092

SELECT productsku, productname 	
FROM products 		--returns 1092, shows that each sku is unique and has a name

SELECT DISTINCT productsku, productname
FROM sales_report
WHERE productsku NOT IN (
SELECT DISTINCT p.productsku
FROM products p
) 						--zero results

SELECT DISTINCT productsku--, productname --commented out since productnames do not appear in sales by sku
FROM sales_by_sku
WHERE productsku NOT IN (
SELECT DISTINCT p.productsku
FROM products p
) 						--8 results

~~~~
These queries show that there is 1:1 overlap of productskus between the products and sales_report tables, while the sales_by_sku produces 8 additional. Thus the total numerber of SKUs we should be looking for is 1100.


Creating a list of all skus used in this data looks like:
~~~~sql
CREATE VIEW all_skus AS
(SELECT productsku
FROM sales_by_sku
UNION
SELECT productsku
FROM products)
~~~~
An issue arrises when looking at the data from all_sessions:
~~~~sql
SELECT DISTINCT se.productsku
FROM clean_sessions se		--results in 536 uniques

SELECT DISTINCT se.productsku
FROM clean_sessions se
WHERE se.productsku IN (
	SELECT * 
	FROM all_skus)		--BUT only 390 are in the other tables
~~~~

So now we can join these to finally have a list of all SKUs
~~~~sql
CREATE TEMP TABLE all_skus1 AS
SELECT * 
FROM all_skus
UNION
SELECT DISTINCT se.productsku
FROM clean_sessions se
WHERE se.productsku NOT IN  -- this WHERE is probably unnecessary
	(
	SELECT * 
	FROM all_skus)	
~~~~
After dropping the previous table and renaming our updated one we have list of primary keys which allows us to begin incorporating the data from the other tables.

Product names appear in three tables, products, sales_report and all_sessions. Disregarding sales_report (since those skus are in products): 
~~~~sql
CREATE TEMP TABLE step_sku_name AS
SELECT DISTINCT s.productsku, 
	(CASE WHEN p.productname IS NOT NULL THEN p.productname	--this will select the productname first from products table, but if null will select from se
	 WHEN p.productname IS NULL THEN se.productname
	 ELSE 'N/A' END) productname
FROM all_skus1 s
LEFT JOIN products p
	ON s.productsku = p.productsku
LEFT JOIN clean_sessions se
	ON s.productsku = se.productsku				--results 1246, as expected, however the 7 SKUs present in sales_by_sku and not in products  										--WILL NOT HAVE NAMES, must be addressed later

~~~~
Next we'll begin formatting these names for consistency and appearance
~~~~sql
-- these next blocks of code demonstrate the hacky way I found to commit multiple find and replaces, by iterating across multiple tables using REPLACE and TRIM functions

CREATE TEMP TABLE sku_n2 AS
SELECT 
	productsku,
	REPLACE(productname, 'Android','') productname
FROM step_sku_name

SELECT * 	--examine results, select additional edits to be made and increment the temp table name
FROM sku_n2
~~~~
This result was then written to a table to be title 'Products' once we have archived the existing products table.

Next we'll work on joining the other date from the product-related tables regarding supply levels
~~~~sql
SELECT 
	s.productsku,
	s.total_ordered
FROM sales_report s
FULL OUTER JOIN sales_by_sku sb
	ON s.productsku = sb.productsku		--results in 462, the expected number given uniques in sales_by_sku
--this result tells us we can ignore the sales_by_sku data, since it is contained in the sales_report

SELECT 				
	p.productsku,
	s.total_ordered,
	p.ordered_quantity
FROM products p
FULL OUTER JOIN sales_report s
	ON p.productsku = s.productsku		--This query shows that there is a large discrepency between the total_ordered and ordered_quantity
~~~~
This result goes along with the common english reading of the different between "Products" and "Sales report." From this parsing I will assume that total_ordered is derrived from the data in all_sessions (the only table which directly links skus and individual sales).

Thus, 'total_ordered' will be discarded and we will incorporate the other columns into a supply_levels table.




~~~~
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
