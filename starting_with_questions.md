Answer the following questions and provide the SQL queries used to find the answer.

    
**Question 1: Which cities and countries have the highest level of transaction revenues on the site?**


SQL Queries:
MANY ASSUMPTIONS
First, if we assume that over the course of a year a particular user will no move cities, and that their orders will always count for the city they are logging in from when no shipping information is provided, then we can join the information from analytics (which contains no city information) with the city and country data from the all_sessions csv. Because I have elected to keep these two information sources mostly separate, following this assumption would require several steps.

~~~~SQL
--create a view, perhaps out of paranoia, to ensure that duplicates are not generated joining
    --data across tables
CREATE VIEW sales_by_city AS (
SELECT 
	se.city, 
	a.units_sold,
	a.unit_price
FROM order_sessions a
JOIN session_view se
	ON a.visitorid = se.visitorid
WHERE a.units_sold > 0
)
--call that view within a query that finds total sold and groups by city
SELECT 
	se.city,
	SUM(se.units_sold*se.unit_price) total_sold
FROM session_view se
JOIN sales_by_city s
	ON se.city = s.city
WHERE se.units_sold > 0
GROUP BY se.city
ORDER BY total_sold DESC
--LIMIT 1 --if you only want the top result, question unclear.

--this operation can be checked by running the data from both tables seperately, which does work.
~~~~

BUT if that isn’t true, and we should instead be relying on the columns ‘transaction_revenue’ and ‘revenue’ in the original data being the most correct, then we would write:
~~~~sql
--simple query except for the embedded CASE to sub out the revenue data from the analytics csv
SELECT 
	se.city,
	SUM(CASE WHEN se.transaction_revenue = 0 THEN a.revenue
	ELSE se.transaction_revenue END) revenue
FROM session_view se
JOIN order_sessions a
	ON se.visit_key = a.visit_key
WHERE se.transaction_revenue > 0 OR a.revenue > 0
GROUP BY se.city
ORDER BY revenue DESC
--I'll not waste space subbing out 'city' for 'country' here, you'll have to imagine it.
~~~~

Answer: These give different answers, which speaks to either the inconsistencies in the data (possible)
OR a fault in my creating the database.
Cities
1. Gives "Mountain View"	243650.70
2. Gives "Sunnyvale"	23418.62
Countries
1. Gives "United States"	15312451.41
2. Gives "United States"	34481.36

**Question 2: What is the average number of products ordered from visitors in each city and country?**



SQL Queries: 
Similar assumptions must be made about the quality of the data and what should be trusted/ignored.
The questions is also ambigous to me:
    Are we averaging the QUANTITY of products, or the NUMBER of individual products. The number of individual products is difficult, since order from the original analytics CSV do not have SKUs.
    
Again, using the same queries, but slightly modified, and making assumptions as above:
~~~~SQL
--We will use the same view structure, since the goal is to COUNT the QUANTITY of each order
CREATE VIEW sales_by_city AS (
SELECT 
	se.city, 
	a.units_sold,
	a.unit_price
FROM order_sessions a
JOIN session_view se
	ON a.visitorid = se.visitorid
WHERE a.units_sold > 0
)
--call that view within a query that finds total number of all products sold and groups that by city
SELECT 
	se.city,
	AVG(se.units_sold) avg_sold
FROM session_view se
JOIN sales_by_city s
	ON se.city = s.city
WHERE se.units_sold > 0
GROUP BY se.city
ORDER BY avg_sold DESC
--LIMIT 1 --if you only want the top result, question unclear.

~~~~


Answer:
gives    "Madrid"	10.0000000000000000
and       "Spain"	10.0000000000000000


**Question 3: Is there any pattern in the types (product categories) of products ordered from visitors in each city and country?**
There probably is, but given the state of the category data it is very hard to say.

SQL Queries:

~~~~sql
--query to TRY to tackle this would be something like
SELECT
    se.city,
    c.product_category,
    COUNT(c.product_category) num_cat
FROM session_view se
JOIN categories c
    ON se.productsku = c.productsku
	WHERE se.city NOT IN ('(not set)')
GROUP BY se.city, c.product_category
ORDER BY se.city, num_cat

--for countries
SELECT
    se.country,
    c.product_category,
    COUNT(c.product_category) num_cat
FROM session_view se
JOIN categories c
    ON se.productsku = c.productsku
WHERE se.country NOT IN ('(not set)')
GROUP BY se.country, c.product_category
ORDER BY se.country, num_cat
~~~~
Answer:

Pretty inconclusive at a glance, made more complicated by that the categories were very hard to clean, involving a lot of manual checking and editing



**Question 4: What is the top-selling product from each city/country? Can we find any pattern worthy of noting in the products sold?**


SQL Queries:
~~~~sql
SELECT 
	se.city, 
	p.productname, 
	SUM(se.units_sold) OVER (PARTITION BY se.productsku, se.city) products_sold
FROM session_view se
JOIN products p 
	ON se.productsku = p.productsku
WHERE se.productsku IS NOT NULL
	AND units_sold > 0
	AND se.city NOT IN ('(not set)')
ORDER BY se.city, products_sold DESC

--for countries
SELECT 
	se.country, 
	p.productname, 
	SUM(se.units_sold) OVER (PARTITION BY se.productsku, se.country) products_sold
FROM session_view se
JOIN products p 
	ON se.productsku = p.productsku
WHERE se.productsku IS NOT NULL
	AND units_sold > 0
	AND se.country NOT IN ('(not set)')
ORDER BY se.country, products_sold DESC

~~~~

Answer:
Since the US is the largest customer center, the most data is from here. Hard to draw any hard comparisons from a small set of data




**Question 5: Can we summarize the impact of revenue generated from each city/country?**

SQL Queries:



Answer:







