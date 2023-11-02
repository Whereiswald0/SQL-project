What are your risk areas? Identify and describe them.



QA Process:
Describe your QA process and include the SQL queries used to execute it.
QA is required throught the process of cleaning to ensure that destructive processes are avoid.

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
