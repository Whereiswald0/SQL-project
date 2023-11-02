What are your risk areas? Identify and describe them.



QA Process:
Describe your QA process and include the SQL queries used to execute it.
QA is required throught the process of cleaning to ensure that destructive processes are avoided.

This involves the liberal use of temporary tables before committing to writing a new table, preserving existing data in archived sections so that old-queries can be re-run, and verifying that correct transformations have been done without losing information. This process is necessarilty rigorous and an example is:  

Let's start by just examining what we might take as primary keys
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
	ON se.sessionid = a.sessionid --48,827
	AND se.visitorid = a.visitorid --47,702
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
This shows that all productskus in the products table are also present in in the cleaned all_sessions data, bar one.

This missing sku was then found in the sales_by_sku table and appended along with the 7 other skus
