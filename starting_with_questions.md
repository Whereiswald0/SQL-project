Answer the following questions and provide the SQL queries used to find the answer.

    
**Question 1: Which cities and countries have the highest level of transaction revenues on the site?**


SQL Queries:

MANY ASSUMPTIONS
First, if we assume that over the course of a year a particular user will no move cities, and that their orders will always count for the city they are logging in from when no shipping information is provided, then we can join the information from analytics (which contains no city information) with the city and country data from the all_sessions csv. Because I have elected to keep these two information sources mostly separate, following this assumption would require several steps.

~~~~SQL
--create a view, perhaps out of paranoia, to ensure that duplicates are not generated joining
    --data across tables
DROP VIEW sales_by_city	
	
CREATE VIEW sales_by_city AS (
SELECT 
	se.city, 
	a.units_sold,
	a.unit_price
FROM analytics a
JOIN sessions se
	ON a.visitorid = se.visitorid
WHERE a.units_sold > 0
)
--call that view within a query that finds total sold and groups by city
SELECT 
	se.city,
	SUM(se.units_sold*se.unit_price) total_sold
FROM sessions se
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
FROM sessions se
JOIN analytics a
	ON se.sessionid = a.sessionid
	AND se.visitorid = a.visitorid
WHERE se.transaction_revenue > 0 OR a.revenue > 0
GROUP BY se.city
ORDER BY revenue DESC
--I'll not waste space subbing out 'city' for 'country' here, you'll have to imagine it.
~~~~

Answer: These give different answers, which speaks to either the inconsistencies in the data (possible)
OR a fault in my creating the database.
Cities
1. Gives 
	"N/A"		10014870.40
	"Mountain View"	334037.25
	"New York"	245041.92
	"Sunnyvale"	138330.00
2. Gives
	"Sunnyvale"	263126.12
	"N/A"		18614.72
	"San Francisco"	5434.90
	"New York"	4058.55		

Countries
1. Gives 
	"United States"	37671671.25
	"Canada"	6540.75
	"Ireland"	899.91
	"France"	559.68
2. Gives 
	"United States"	298977.43
	"Switzerland"	305.82

**Question 2: What is the average number of products ordered from visitors in each city and country?**



SQL Queries: 
Similar assumptions must be made about the quality of the data and what should be trusted/ignored.
The questions is also ambigous to me:
    Are we averaging the QUANTITY of products, or the NUMBER of individual products. The number of individual products is difficult, since order from the original analytics CSV do not have SKUs.
    
Again, using the same queries, but slightly modified, and making assumptions as above:
~~~~SQL
--We will use the same view as above, since the goal is to COUNT the QUANTITY of each order we only need units_sold over city/country, which we already retrieved

--call that view within a query that finds total number of all products sold and averages that by city
SELECT 
	se.city,
	AVG(se.units_sold) avg_sold
FROM sessions se
JOIN sales_by_city s
	ON se.city = s.city
WHERE se.units_sold > 0
--	AND se.city NOT IN('N/A') --CONSIDERED removing 'N/A' as an option, since it isn't a city
GROUP BY se.city			--but it does give a better picture of the missing data
ORDER BY avg_sold DESC

~~~~


Answer:
For city:    
	"Madrid"	10.0000000000000000
	"N/A"		9.6562500000000000
	"Salem"		8.0000000000000000
	"Atlanta"	4.0000000000000000
	"Houston"	2.0000000000000000
	"New York"	1.1111111111111111
	"Los Angeles"	1.00000000000000000000
	"Mountain View"	1.00000000000000000000
	"Palo Alto"	1.00000000000000000000
	"San Francisco"	1.00000000000000000000
	"San Jose"	1.00000000000000000000
	"Seattle"	1.00000000000000000000
	"(not set)"	1.00000000000000000000
	"Sunnyvale"	1.00000000000000000000
	"Ann Arbor"	1.00000000000000000000
	"Chicago"	1.00000000000000000000
	"Dallas"	1.00000000000000000000
	"Detroit"	1.00000000000000000000
	"Dublin"	1.00000000000000000000

 and for country:
	 "Spain"	10.0000000000000000
	"United States"	6.3333333333333333
	"Finland"	1.00000000000000000000
	"France"	1.00000000000000000000
	"Canada"	1.00000000000000000000
	"Ireland"	1.00000000000000000000
	"Mexico"	1.00000000000000000000
	"India"		1.00000000000000000000
	"Colombia"	1.00000000000000000000


**Question 3: Is there any pattern in the types (product categories) of products ordered from visitors in each city and country?**
There probably is, but given the state of the category data it is very hard to say.

SQL Queries:

~~~~sql
--query to TRY to tackle this would be something like
SELECT
    se.city,
    c.product_category,
    COUNT(c.product_category) num_cat
FROM sessions se
JOIN categories c
    ON se.productsku = c.productsku
GROUP BY se.city, c.product_category
ORDER BY se.city, num_cat

--for countries
SELECT
    se.country,
    c.product_category,
    COUNT(c.product_category) num_cat
FROM sessions se
JOIN categories c
    ON se.productsku = c.productsku
GROUP BY se.country, c.product_category
ORDER BY se.country, num_cat
~~~~
Answer:

Pretty inconclusive at a glance, made more complicated by that the categories were very hard to clean, involving a lot of manual checking and editing.



**Question 4: What is the top-selling product from each city/country? Can we find any pattern worthy of noting in the products sold?**


SQL Queries:
~~~~sql
SELECT 
	se.city, 
	p.productname, 
	SUM(se.units_sold) OVER (PARTITION BY se.productsku, se.city) products_sold
FROM sessions se
JOIN products p 
	ON se.productsku = p.productsku
WHERE se.productsku IS NOT NULL
	AND units_sold > 0
ORDER BY se.city, products_sold DESC

--for countries
SELECT 
	se.country, 
	p.productname, 
	SUM(se.units_sold) OVER (PARTITION BY se.productsku, se.country) products_sold
FROM sessions se
JOIN products p 
	ON se.productsku = p.productsku
WHERE se.productsku IS NOT NULL
	AND units_sold > 0
	AND se.country NOT IN ('(not set)')
ORDER BY se.country, products_sold DESC
~~~~

Answer:
Note that since I was unable to join skus to the data from analytics with any degree of certainty, that data is not included here.

Since the US is the largest customer center, the most data is from here. Hard to draw any hard comparisons from a small set of data about countries

The main thing I can draw from this question is a reduced faith in the adequacy of either the data itself or my efforts at cleaning it, since it looks like a lot of repeated order might be showing up here..


**Question 5: Can we summarize the impact of revenue generated from each city/country?**

Again, we must define away any ambiguities in the question.
The impact of revenue can be understood in many ways, but mostly with regards to the business goals.
Here we can say that the revenue generated in the US obviously means that that is where the biggest existing customer base is. Does that mean that efforts should be made to grow there? Or is the business at it's cap in the US and should be looking for opportunities elsewhere?

What we can do is generate indications of sales numbers from the data

SQL Queries:

~~~~sql
--returning to our query from the 1st question
CREATE VIEW sales_by_city AS (
SELECT 
	se.city, 
	a.units_sold,
	a.unit_price
FROM analytics a
JOIN sessions se
	ON a.visitorid = se.visitorid
WHERE a.units_sold > 0
)
--call that view within a query that finds total sold and groups by city
SELECT 
	se.city,
	SUM(se.units_sold*se.unit_price) total_sold
FROM sessions se
JOIN sales_by_city s
	ON se.city = s.city
WHERE se.units_sold > 0
GROUP BY se.city
ORDER BY total_sold DESC
~~~~
And for countries
~~~~sql
CREATE VIEW sales_by_country AS (
SELECT 
	se.country, 
	a.units_sold,
	a.unit_price
FROM analytics a
JOIN sessions se
	ON a.visitorid = se.visitorid
WHERE a.units_sold > 0
)

SELECT 
	se.country,
	SUM(se.units_sold*se.unit_price) total_sold
FROM sessions se
JOIN sales_by_country s
	ON se.country = s.country
WHERE se.units_sold > 0
GROUP BY se.country
ORDER BY total_sold DESC
~~~~
Answer:
Any number of things could be shown by this data, without further context we would only be guessing.

Data without context can tell us only so much.






