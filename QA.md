What are your risk areas? Identify and describe them.

There are two immediate risk areas here. One involves duplicate data between the all_sessions and analytics tables as initially presented. Since the data covers transactions over the same period, we should try to match them up, but because there is a lack of precision or missing data this becomes impossible.

An example of this would be finding a visitor's data included on both, seeing that they had placed an order, having a discrepancy between the two tables in revenue generated or number of products ordered. Such an incident could be attributed to either a mis-recorded product (perhaps it was left in the cart and not selected for final purchase) or imprecise or corrupted data. 

This issue is explicated below with specifics.

The second risk has to do with both variation and imprecision within a column, most often in product names and categories. 


QA Process:
Describe your QA process and include the SQL queries used to execute it.

QA is required throughout the process of cleaning to ensure that destructive processes are avoided, thus a great deal of my writing in the cleaning_data section is relevant. Rather than reproducing that here, I'll just add a few notes about the avenues I avoided do to QA concerns.

Perhaps the most frustrating thing, and something which highlights the need for rigorous QA, was trying to join product SKUs to orders. To begin:

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
We certainly expect there to be more visitids (here renamed 'sessionid') than fullvisitorids (here renamed 'visitorid'), so that isn't surprising, but the data taken from analytics contains many more of each, more than 8x as many. Given the overlapping dates for this data though, we should see what might happen if were we to try to join these two sources. 
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


My notes are a bit sloppy, but the intent is clear, I think. While the volume of data present in the analytics csv is large, because at least some of it is missing from the data from all_sessions, joining it is a very dangerous thing. Especially when we are unsure of the collection methods and the future uses.

We can see from that that there to be more visitids (renamed 'sessionid') than fullvisitorids (renamed 'visitorid') in the original data, so that we might have some trouble joining them is obvious, however even matching orders in the analytics data to lines in the all_sessions data in order to attempt to locate SKUs for particular rows we find missing data, revenue totals that don't add up, and other absent details that would help this make sense.

