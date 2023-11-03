Question 1: Over which dates were orders placed? Which months?

SQL Queries:

~~~sql


CREATE VIEW sales_2016 AS (
SELECT 
	EXTRACT('Month' FROM visit_date) AS Month, 
	'2016' AS Year,
	SUM(units_sold*unit_price) total_sold
FROM sessions
WHERE visit_date BETWEEN '01-01-2016' AND '12-31-2016'
GROUP BY EXTRACT('Month' FROM visit_date) 
ORDER BY EXTRACT('Month' FROM visit_date)
		)

CREATE VIEW sales_2017 AS (
SELECT 
	EXTRACT('Month' FROM visit_date) AS Month, 
	'2017' AS Year,
	SUM(units_sold*unit_price) total_sold
FROM sessions
WHERE visit_date BETWEEN '01-01-2017' AND '12-31-2017'
GROUP BY EXTRACT('Month' FROM visit_date) 
ORDER BY EXTRACT('Month' FROM visit_date)
		)
--These two views give us the monthly sales totals for the years we have data from

SELECT * FROM sales_2016 --joining the data like this allows an easier view of how sales change month-to-month and from one year to the nxt
UNION ALL
SELECT * FROM sales_2017

--one could also join them and order by month to better compare year-over-year sales, though we only have the last third of 2016 to examine
~~~

Answer: 
8	"2016"	0.00
9	"2016"	1278.72  --2nd highest monthly sales right after the data begins, will need to see if this matches the following Sept.
10	"2016"	632.07
11	"2016"	525.00
12	"2016"	681.16
1	"2017"	679.98
2	"2017"	945.00
3	"2017"	1715.54  --March seems like an odd time to have a sales spike, rather than Nov. or Dec, which are flat
4	"2017"	404.94
5	"2017"	369.93
6	"2017"	786.99
7	"2017"	302.95
8	"2017"	0.00


Question 2: When vistors return, are they more likely to make a purchase?

SQL Queries:
~~~sql
SELECT visitnumber, COUNT(DISTINCT sessionid) 
FROM analytics
WHERE units_sold > 0
GROUP BY visitnumber
ORDER BY visitnumber
~~~
Answer:

Sales mostly occur on the first visit, and then fall off, down to double digits by the mid-twenties and then single digits in the range of 30th-40th visit.
While this data is only stretching across a single year, working on retaining and building repeat customers should certainly be a goal.


Question 3: 
Are sales coming from many users making small purchases or fewer users making large purchases?

SQL Queries:
~~~~sql
WITH order_temp AS ( 	
SELECT 
	visitorid,
	(CASE WHEN order_amt < 100 THEN '0-99'
	WHEN order_amt <=300 then '100-300'
	WHEN order_amt <=500 THEN '301-500'
	WHEN order_amt > 500 THEN '500+' END) order_vol
FROM sales_by_user
ORDER BY order_vol)
SELECT order_vol, COUNT(visitorid)
FROM order_temp
GROUP BY order_vol
~~~~
Answer:
"0-99"		8841
"100-300"	3252
"301-500"	1056
"500+"		2914
Most users have purchased under $100 worth of goods over the course of the year.


Question 4: 

SQL Queries:

Answer:



Question 5: 

SQL Queries:

Answer:
