QA Process:
Describe your QA process and include the SQL queries used to execute it.

What are your risk areas? Identify and describe them.

### One of the risk areas with this data are making sure the logic is correct when answering questions about the data. For example let's say we wanted to get the number of distinct visitors on the website. We might think to do this:

```sql
SELECT COUNT(full_visitor_id) AS total_unique_visitors
FROM cleaned_all_sessions;
```

However doing this doesn't each full_visitor_is is actually unique meaning that we could double count visitors.
Instead we should do this:

```sql
SELECT COUNT(DISTINCT full_visitor_id) AS total_unique_visitors
FROM cleaned_all_sessions;
```
To summarize: This first risk area is duplicate data.

### A second risk area is incorrectly formatted data, for example. The product_variant columns data looked something like this, 

<pre>'          product_variant_name'</pre>
The data contained leading spaces.

We can the LTRIM() function in this query.

``` sql
SELECT LTRIM(product_variant) FROM all_sessions_backup WHERE product_variant != '(not set)'
```
### A third risk area is using incorrect data types.

``` sql
--This query calculate the percentage of visitors who made a purchase on the website.

WITH count_of_sales AS (
SELECT
  COUNT(DISTINCT CASE WHEN total_transaction_revenue > 0 THEN full_visitor_id END) AS count_with_revenue,
  COUNT(DISTINCT CASE WHEN total_transaction_revenue = 0 THEN full_visitor_id END) AS count_zero_revenue,
  COUNT(DISTINCT full_visitor_id) AS total_distinct_visitors,
  (COUNT(DISTINCT CASE WHEN total_transaction_revenue > 0 THEN full_visitor_id END) * 100.0 / COUNT(DISTINCT full_visitor_id)) AS percentage_with_revenue,
  (COUNT(DISTINCT CASE WHEN total_transaction_revenue = 0 THEN full_visitor_id END) * 100.0 / COUNT(DISTINCT full_visitor_id)) AS percentage_zero_revenue
FROM cleaned_all_sessions
)

SELECT (count_with_revenue/total_distinct_visitors), (count_zero_revenue/total_distinct_visitors)
FROM count_of_sales
```
This seems solid, however no matter the number of count_with_revenue and count_zero_revenue you always get 0 for both columns.
Why would this happen? Because when you division with two integer types of data you will get an integer as a result unless you specify a data type that can include decimal values. And since any number divided by a number larger than itself will result in a decimal value which that gets rounded by post gres down to zero, this query will not work the way you think it would.

In order to fix this we can specify the data type we want the result of our division to be in our SELECT statement.

``` sql
WITH count_of_sales AS (
SELECT
  COUNT(DISTINCT CASE WHEN total_transaction_revenue > 0 THEN full_visitor_id END) AS count_with_revenue,
  COUNT(DISTINCT CASE WHEN total_transaction_revenue = 0 THEN full_visitor_id END) AS count_zero_revenue,
  COUNT(DISTINCT full_visitor_id) AS total_distinct_visitors,
  (COUNT(DISTINCT CASE WHEN total_transaction_revenue > 0 THEN full_visitor_id END) * 100.0 / COUNT(DISTINCT full_visitor_id)) AS percentage_with_revenue,
  (COUNT(DISTINCT CASE WHEN total_transaction_revenue = 0 THEN full_visitor_id END) * 100.0 / COUNT(DISTINCT full_visitor_id)) AS percentage_zero_revenue
FROM cleaned_all_sessions
)

SELECT count_with_revenue/total_distinct_visitors::numeric, count_zero_revenue/total_distinct_visitors::numeric
FROM count_of_sales
```

Now that we have cast to the numeric data type we will get the expected result.
