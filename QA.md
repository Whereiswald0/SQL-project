What are your risk areas? Identify and describe them.

There are two immediate risk areas here. One involves duplicate data between the all_sessions and analytics tables as initially presented. Since the data covers transactions over the same period, we should try to match them up, but because there is a lack of precision or missing data this becomes impossible.

An example of this would be finding a visitor's data included on both, seeing that they had placed an order, having a discrepancy between the two tables in revenue generated or number of products ordered. Such an incident could be attributed to either a mis-recorded product (perhaps it was left in the cart and not selected for final purchase) or imprecise or corrupted data. 

This issue is explicated below with specifics.

The second risk has to do with both variation and imprecision within a column, most often in product names and categories. 


QA Process:
Describe your QA process and include the SQL queries used to execute it.

QA is required throughout the process of cleaning to ensure that destructive processes are avoided, thus a great deal of my writing in the cleaning_data section is relevant. Rather than reproducing that here, I'll just add a few notes about the avenues I avoided due to QA concerns.

FIrstly when looking at the data from analytics and all_sessions we presume there to be some overlap because of the data in the data fields. We can check this as follows:

~~~sql
SELECT *
FROM clean_analytics a
FULL OUTER JOIN clean_sessions se
	ON a.visitorid = se.visitorid 
WHERE se.visitorid IS NULL
-- 1,656,058 rows produced IN analytics, MISSING from all_sessions
AND units_sold >0
--58,529 with units sold
AND a.revenue>0 
--12,934 where positive revenue is recorded

SELECT *
FROM clean_analytics a
LEFT JOIN clean_sessions se
	ON a.sessionid = se.sessionid 
WHERE se.sessionid IS NULL
--1,691,948 rows produced IN analytics, MISSING from all_sessions
AND units_sold >0
--60,501 produced with with units sold
AND a.revenue>0 
--13,356 where positive revenue is recorded
~~~
This suggests that there are holes in our data, given the number of nulls produced by these queries. Moreover we can say that these holes are significant because they involve orders (rows which contain units sold and revenue totals). This was enough to give me cause to halt the creation of a table joining these two data sets, which was a major issue since certain information about orders was contained in one set and not the other. But, if we were able to identify a common primary key (perhaps a combination of visitid and fullvisitorid, we could make a bridge table and join them. 
~~~~sql
SELECT 
a.visitnumber,
se.sessionid,
se.visitorid
FROM clean_sessions se
JOIN clean_analytics a
	ON se.sessionid = a.sessionid --48,827 results
	AND se.visitorid = a.visitorid --47,702 results
~~~~
The discrepency between joining on just the visitid (renamed sessionid here) and including the visitorid as a criteria suggests that this cannot join our two tables directly, since a common key would produce a good deal of nulls. I did end up setting up one, but at the state the data is in, it is not going to pull in the best results.

The second tool of QA I used involved checking uniques, combining results into a temporary table and making sure the resulting table returned the expected number of rows.

~~~~sql
SELECT DISTINCT productsku
FROM products 	--returns 1092

SELECT productsku, productname
FROM products 	--returns 1092

SELECT DISTINCT productsku
FROM clean_sessions 	--returns 536

SELECT DISTINCT p.productsku
FROM products p
FULL OUTER JOIN clean_sessions se
	ON p.productsku = se.productsku		--returns 1093	
~~~~
This process shows that one additional SKU is contained in the data from all_sessions.

This same process is then done for all tables containing productskus, ensuring that no data is lost in the process of joining these tables to create a list of products and their associated details.
