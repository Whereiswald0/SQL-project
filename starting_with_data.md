Question 1: Over which dates were orders placed.

SQL Queries:

Answer: 



Question 2: When vistors return, are they more likely to make a purchase

SQL Queries:

Answer:



Question 3: Are frequent visitors better customers?

SQL Queries:
SELECT DISTINCT 
	a.visitorid,
	MAX(a.visitnumber)high_visit,
	MAX(a.visit_date) high_date
FROM clean_analytics a
GROUP BY a.visitorid
ORDER BY high_visit DESC
Answer:



Question 4: 

SQL Queries:

Answer:



Question 5: 

SQL Queries:

Answer:
