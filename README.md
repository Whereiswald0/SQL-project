# Final-Project-Transforming-and-Analyzing-Data-with-SQL

## Project/Goals
To clean and organize the data presented, ensuring that the dataset become legible and easy to use.
Secondary goals include making note of key information which is missing and preserving data which can be made useful at a later time or with additional information.

## Process
### Loading data, making note of datatypes -
  This stage involved opening the unprocessed CSVs to retrieve headers and deducing the datatypes based on initial values. To load the data an SQL query was written in PGAdmin, first to programmatically format the headers and then to create a table. In general, numerical data was imported as NUMERIC, while text data was imported as VARCHAR. Fields labelled ‘data’ or similar were imported as DATE types, while fields labeled ‘time’ were imported as INTEGERS to maintain ease of CASTing, in order to not preclude the possibility of those fields being easily formatted as INTERVAL or HH:MM:SS.

### Examining data as initially formatted
  At this point, with all of the received data imported, the process of examining it in detail could begin by calling each column with aggregate functions. This is done to check for the number of unique values vs. duplicates, as well as to get an idea of the data ranges, number of NULL or ‘0’ values, and any outliers or completely empty columns.

### Formatting, CAST, identifying sets across tables
  Now we can move on to making some assumptions about that data set we have received and how we can begin making it more legible. This includes some preliminary analysis and test-casting from one datatype to another.

### Draw up preliminary datasets
  Having examined the data thoroughly and examined the initial structure, we should begin testing that structure to see if alternatives will be more useful and clear.

### Re-evaluate data with additional structures in mind. 
  The number of tables, their relationships, primary keys and the way they will interact must now be understood.

### Implementing those structures 
  At this stage we can begin writing and testing joins, making sure that all remaining NULLs, duplicates and anomalous data is accounted for.

### Devise sample queries
  Do these queries indicate a need for additional tables, or fewer tables? Do the results look like what is expected?

### REMEMBER THAT YOU FORGOT TO DOCUMENT EVERYTHING
  But you did save the queries used to generate your tables, so that’s easy to go back to.

### Complete ERD, draw conclusions

### 

## Results
(fill in what you discovered this data could tell you and how you used the data to answer those questions)

  The data can be broken into two large groups - one set spread across the all_sessions and analytics CVS’ has to do with user-specific webtraffic and purchases, while the second set contains information about products and their supply level. 

  As far as what could be said using this data, assuming it was all collected from a single merchant, they operate with a heavy emphasis on US-based customers, sell a variety of products, and have been tracking website traffic between August 2016 and August 2017, with an additional collection model applied from May to August of  2017. 

  This data can indicated the frequency of orders with broad categories, but not with regards to specific products.

## Challenges 
(discuss challenges you faced in the project)
  The lack of an acceptable primary key within the data for customers was a struggle initially. Because the visitid is derived directly from the time and date of the visit, it cannot function as a primary key to act as a primary key for each individual visit. While this data did not show any evidence that multiple users had generated the same visitid, there was also nothing to indicate that such a thing was not possible.

  As a result I proposed the use of a concatenated key derived from the visit_number (which counts each session per user) and the user id (generated as ‘fullvisitorid’). If maintained, this key would prove useful in aggregating a user’s activity over a whole session.

  The other main challenge encountered was the vast amount of missing data which would all for the analysis of purchases and the ability to match purchases to products in inventory. Because SKUs not present in the analytics, the sales denoted there are difficult to untangle. Without SKUs the only data regarding purchases would often be units_sold and unit_price. This would then make it difficult to figure out if what was present in the data was a duplicate order or several different orders completed at the same time for products with identical prices. 
  This also resulted in a great deal of difficulty joining the sales data from the all_sessions csv and the analytics csv, since one couldn't be sure that the data didn't already exist in one or the other table, and given the missing and duplicated information even appealing to the appearance of a positive integer in the revenue column couldn't be taken as a sign that there wasn't missing data, since often the revenue total would not be the same as the quantity multiplied by price.

  The only other thing of note which might be called a challenge was the inconsistent product categories, which often feature extraneous and duplicated information within a single string. I attempted to edit these strings with a regex function, but ended up just running a cascade of REPLACE and TRIM functions through a series of temporary tables in order to tidy them.

## Future Goals
  The only thing that comes to mind immediately is to further divide the data into those orders which can be definitively matched with SKUs, by matching visitid, fullvisitorid and price. An initial attempt at this only established a handful results, with some of those possibly being in error or duplicates.

  Also, future-proofing data collection, requesting the data missing regarding purchases, establishing a way of generating visitids that does not rely on the start-time of the visit.


