What are your risk areas? Identify and describe them.

There are two immediate risk areas here. One involves duplicate data between the all_sessions and analytics tables as initially presented. Since the data covers transactions over the same period, we should try to match them up, but because there is a lack of precision or missing data this becomes impossible.

An example of this would be finding a visitor's data included on both, seeing that they had placed an order, having a discrepancy between the two tables in revenue generated or number of products ordered. Such an incident could be attributed to either a mis-recorded product (perhaps it was left in the cart and not selected for final purchase) or imprecise or corrupted data. 

This issue is explicated below with specifics.

The second risk has to do with both variation and imprecision within a column, most often in product names and categories. 


QA Process:
Describe your QA process and include the SQL queries used to execute it.

QA is required throughout the process of cleaning to ensure that destructive processes are avoided, thus a great deal of my writing in the cleaning_data section is relevant. 

Some additional notes:

For me this process involved the liberal use of temporary tables before committing to writing a new table, preserving existing data in archived sections so that old-queries could be re-run, and verifying that correct transformations have been done without losing information. This process is necessarily rigorous and an example is to start by just examining what we might take as primary keys
~~~~sql
SELECT DISTINCT visitorid
FROM clean_analytics --120,018 results

SELECT DISTINCT visitorid
FROM clean_sessions --14,223 results

SELECT DISTINCT sessionid
FROM clean_analytics --148,642 results

SELECT DISTINCT sessionid
FROM clean_sessions --14,556 results
~~~~
We certainly expect there to be more visitids (here renamed 'sessionid') than fullvisitorids (here renamed 'visitorid'), so that isn't surprising, but the data taken from analytics contains many more of each, more than 8x as many. 

This is already suggesting that there are holes in our data, but if we should proceed with trying to identify a primary key and join our data:
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
If after cleaning the all_sessions and analytics tables we simply joined them assuming that the sessionid  + visitorid would provide a link between the two datasets we could be destroying information particular to each table. Joining on just sessionid produces 48,827 results, while joining on both session and visitor ids produces 47,702. This indicates that some visitors and sessions are not recorded in one or the other tables and we must decide a course of action.

Drilling down into this would be a longer process than I was able to accomplish, so I elected to maintain potentially overlapping tables and to remind myself that the data is incomplete and further action will be required.


This same process when applied to the other suspected primary key in the data was much more successful.
~~~~sql
SELECT DISTINCT p.productsku, p.productname
FROM products p
FULL OUTER JOIN clean_sessions se
	ON p.productsku = se.productsku
	WHERE p.productsku IS NOT NULL	--produces 1093, 1 null
~~~~
This shows that all productskus in the products table are also present in the cleaned all_sessions data, bar one.

This missing sku was then found in the sales_by_sku table and appended along with the 7 other skus.
