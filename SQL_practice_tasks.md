### 1. Coding contest leaderboard
Write a query to print the respective hacker_id and name of hackers who achieved full scores for more than one challenge. Order your output in descending order by the total number of challenges in which the hacker earned a full score. If more than one hacker received full scores in same number of challenges, then sort them by ascending hacker_id. 

![hgkjhkh](https://github.com/oksana-kh/sql_practice/blob/main/help_files/task1_scoreboard.png)

~~~sql
SELECT h.hacker_id,
       h.name
FROM hackers AS h
JOIN submissions AS s
	ON h.hacker_id = s.hacker_id
JOIN challenges AS c
	ON s.challenge_id = c.challenge_id
JOIN difficulty AS d
	ON c.difficulty_level = d.difficulty_level
GROUP BY h.hacker_id,
         h.name
HAVING SUM(IF (s.score = d.score, 1, 0) ) > 1 
ORDER BY SUM(IF (s.score = d.score, 1, 0) ) DESC,
         h.hacker_id;
~~~

### 2. Search algorithm evaluation
We have a table with the user's search term, search result positions, and whether or not the user clicked on the search result.  
Write a query that assigns ratings to the searches in the following way:  
•	If the search was not clicked for any term, assign the search with rating=1  
•	If the search was clicked but the top position of clicked terms was outside the top 3 positions, assign the search a rating=2  
•	If the search was clicked and the top position of a clicked term was in the top 3 positions, assign the search a rating=3  
As a search ID can contain more than one search term, select the highest rating for that search ID. Output the search ID and its highest rating.
Example: The search_id 1 was clicked (clicked = 1) and its position is outside of the top 3 positions (search_results_position = 5), therefore its rating is 2. 

**fb_search_events**  
search_id: int  
search_term: varchar  
clicked: int  
search_results_position: int

~~~sql
SELECT search_id,
       MAX(rating) AS rating
FROM (
	SELECT *, CASE 
			WHEN clicked * search_results_position = 0
				THEN 1
			WHEN clicked * search_results_position > 3
				THEN 2
			WHEN clicked * search_results_position BETWEEN 1 AND 3
				THEN 3
			END AS rating
	FROM fb_search_events
	) AS tbl_ratings
GROUP BY search_id;
~~~

### 3. Month-over-month revenue change
Given a table of purchases by date, calculate the month-over-month percentage change in revenue. The output should include the year-month date (YYYY-MM) and percentage change, rounded to the 2nd decimal point, and sorted from the beginning of the year to the end of the year.  
The percentage change can be calculated as ((this month's revenue - last month's revenue) / last month's revenue)*100. 

**sf_transactions**  
id: int  
created_at: datetime  
value:int  
purchase_id:int

~~~sql
SELECT month,
       ROUND(100 * (revenue - rev_preced) / rev_preced, 2) AS percent_change
FROM (
      SELECT DATE_FORMAT(created_at, "%Y-%m") AS month,
             SUM(value) AS revenue,
             LAG(SUM(value)) OVER (
                                ORDER BY DATE_FORMAT(created_at, "%Y-%m") ASC
                                ) AS rev_preced
      FROM sf_transactions
      GROUP BY month
      ) AS tbl_revenues
ORDER BY month ASC;
~~~

### 4. Paying and non-paying downloads
Find the total number of downloads for paying and non-paying users by date. Include only records where non-paying customers have more downloads than paying customers. The output should be sorted by earliest date first and contain 3 columns date, non-paying downloads, paying downloads.

**ms_download_facts**  
date: datetime  
user_id: int  
downloads: int  

**ms_user_dimension**  
user_id: int  
acc_id: int  

**ms_acc_dimension**  
acc_id: int  
paying_customer: varchar  

~~~sql
SELECT date,
       SUM(IF(paying_customer = "yes", downloads, 0)) AS paying,
       SUM(IF(paying_customer = "no", downloads, 0)) AS non_paying
FROM ms_download_facts AS d
JOIN ms_user_dimension  AS u
	ON d.user_id = u.user_id
JOIN ms_acc_dimension AS a
	ON u.acc_id = a.acc_id
GROUP BY date
HAVING non_paying > paying
ORDER BY date;
~~~

### 5. Top 5 percentile
ABC Corp is a mid-sized insurer in the US and in the recent past their fraudulent claims have increased significantly for their personal auto insurance portfolio. They have developed a ML based predictive model to identify propensity of fraudulent claims. Now, they assign highly experienced claim adjusters for top 5 percentile of claims identified by the model.  
Your objective is to identify the top 5 percentile of claims from each state. Your output should be policy number, state, claim cost, and fraud score.

**fraud_score**  
policy_num: varchar  
state: varchar  
claim_cost: int  
fraud_score: float  

~~~sql
WITH percentiles
AS (
      SELECT policy_num,
             state,
             claim_cost,
             fraud_score,
             PERCENT_RANK() OVER (
                                  PARTITION BY STATE
                                  ORDER BY fraud_score ASC
                                  ) AS percentile
      FROM fraud_score
      )
SELECT policy_num,
       state,
       claim_cost,
       fraud_score
FROM percentiles
WHERE percentile >= 0.95;
~~~

### 6. Popularity percentage
Find the popularity percentage for each user on Meta/Facebook. The popularity percentage is defined as the total number of friends the user has divided by the total number of users on the platform, then converted into a percentage by multiplying by 100.  
Output each user along with their popularity percentage. Order records in ascending order by user id.  
The 'user1' and 'user2' column are pairs of friends.

**facebook_friends**  
user1: int  
user2: int  

~~~sql
WITH total_users
AS (
    SELECT COUNT(*)
    FROM (
          SELECT user1
          FROM facebook_friends AS tbl1
          
          UNION
          
          SELECT user2
          FROM facebook_friends AS tbl2
          ) AS tbl3
    )
SELECT user_1 AS user,
       100 * COUNT(*) / (SELECT * FROM  total_users) AS popularity
FROM (
      SELECT user1 AS user_1,
             user2 AS user_2
      FROM facebook_friends AS tbl1
      
      UNION
      
      SELECT user2 AS user_1,
             user1 AS user_2
      FROM facebook_friends AS tbl2
      ) AS tbl3
GROUP BY user
ORDER BY user;
~~~

### 7. Top 5 states with the most 5 star businesses
Find the top 5 states with the most 5 star businesses. Output the state name along with the number of 5-star businesses and order records by the number of 5-star businesses in descending order. In case there are ties in the number of businesses, return all the unique states. If two states have the same result, sort them in alphabetical order.

**yelp_business**  
business_id: varchar  
name: varchar  
neighborhood: varchar  
address: varchar  
city: varchar  
state: varchar  
postal_code: varchar  
latitude: float  
longitude: float  
stars: float  
review_count: int  
is_open: int  
categories: varchar  

~~~sql
SELECT state,
       num_5_stars
FROM (
      SELECT state,
             COUNT(*) AS num_5_stars,
             RANK() OVER (ORDER BY COUNT(*) DESC) AS num_5_stars_rank
      FROM yelp_business
      WHERE stars = 5
      GROUP BY state
      )
WHERE num_5_stars_rank <= 5
ORDER BY num_5_stars DESC,
         state;
~~~

### 8. Pulling dosage information
You work with a database to track intricate drug dosages administered during clinical trials, often referencing varied units, including secondary measures like body weight.  
Your task as to write SQL query that pulls relevant dosage information. The management wants to view the record_id, name of the drug, the drug_amount, and its associated dose_units for each record.  
The dose_units are sometimes a combination of two units. If a secondary unit exists (check_unit_id), the team wants to see it in the format primary_unit/secondary_unit (e.g., mg/kg). If there's no secondary unit, only the primary unit (drug_unit_id) should be displayed.

**drugs**  
drug_id (integer)  
drug_name (varchar)   

**units**  
unit_id (integer)  
unit_name (varchar)   

**dose_records**  
record_id (integer)  
drug_id (integer)  
drug_amount (float)  
drug_unit_id (integer)  
check_unit_id (integer, nullable)  

~~~sql
SELECT ds.record_id,
       dr.drug_name,
       ds.drug_amount,
        CASE 
            WHEN ds.check_unit_id IS NOT NULL THEN u.unit_name || '/' ||  u2.unit_name
            ELSE u.unit_name
        END AS dose_units
FROM dose_records AS ds
JOIN drugs AS dr
	ON ds.drug_id = dr.drug_id
JOIN units AS u
	ON ds.drug_unit_id = u.unit_id
LEFT JOIN units AS u2
	ON ds.check_unit_id  = u2.unit_id
ORDER BY dr.drug_name,
         ds.record_id
~~~

### 9. Rebate percentage
Write an SQL query that calculates the total amount of orders for each customer for September 2023 and determines the corresponding rebate percentage they are eligible for based on their total orders.  
Objectives:
1. Calculate the sum of order amounts for each customer for orders placed in September 2023 (Assuming that the orders table contains various orders with order_date spanning different months and years, and order_amount varying between orders).
2. Determine the appropriate rebate percentage for each customer based on the calculated total order amount.
3. Exclude customers who do not have orders in September 2023 or whose total orders in September 2023 are below the lowest min_purchase in the rebates table.  

A result set should contain the following columns:
1. customer_id (integer) - The ID of the customer.
2. total_orders (integer) - The total sum of the orders made by the customer in September 2023.
3. rebate_percentage (float) - The rebate percentage applicable to the customer based on their total orders in September 2023.  

and be sorted by customer_id in descending order. 

**orders**  
customer_id  
order_amount  

**rebates**  
id  
rebate_percentage  
min_purchase  

~~~sql
WITH totals
AS (
    SELECT customer_id,
           SUM(order_amount) AS total_orders
    FROM orders
    WHERE order_date BETWEEN '2023-09-01' AND '2023-09-30'
    GROUP BY customer_id
    )
SELECT customer_id,
       total_orders,
       MAX(rebate_percentage) AS rebate_percentage
FROM totals AS t
JOIN rebates AS r
	ON t.total_orders >= r.min_purchase
GROUP BY customer_id,
         total_orders
ORDER BY customer_id DESC;
~~~

### 10. Customers with undelivered orders
Write an SQL query that selects all customers for whom all orders are undelivered. Return the result sorted in descending order by customer_id.

**orders**  
id  
customer_id  
delivery_date  

~~~sql
SELECT customer_id
FROM orders
GROUP BY customer_id
HAVING EVERY(delivery_date IS NULL)
ORDER BY 1 DESC;
~~~

### 11. Product codes
Write an SQL query to identify products that:  
- Have at least one other product with the same product_code but from a different group_id.  
- For a given product_code and group_id combination, if multiple entries exist, the output should only return the entry with the highest product_id.  
- If a product has a product_code that isn't found in any other group_id, then it should not appear in the result set.  

**products**  
product_id (integer)  
group_id (integer)  
product_code (varchar)  

~~~sql
SELECT DISTINCT ON (group_id, product_code) p.* - - for each combination of group_id and product_code select the first row by the order assigned in ORDER BY
FROM products p
JOIN products e
USING(product_code)
WHERE e.group_id <> p.group_id  - - selects only rows with product codes that appear with more than one group_id
ORDER BY product_code, group_id DESC, product_id DESC;
~~~

### 12. Backups information
Product IT Company uses AWS clusters for its operations and employs SSD-backed volumes within these clusters for regular data backups. Each cluster has a unique ID, and handles a series of backup tasks. Occasionally, backups on a cluster are restarted due to various reasons, leading to multiple 'Start' events before an 'End' event is logged for a backup task. Your role is to analyze the backup logs, ensuring each start event pairs with an end, and derive insights from these events.

We have backup_events table:

id (integer) - primary key.
event_datetime (timestamp) - The date and time of the backup event.
aws_cluster_id (integer) - A unique identifier for each AWS cluster.
backup_status (text) - Either 'Start' or 'End', indicating the backup event's status.
From the given database table, write an SQL query to:

Pair each backup start time with its corresponding end time.
Calculate the total duration of the backup.
Determine the number of restarts within each backup event.
Order the results by start_time in ascending order, and in the case of a tie - by aws_cluster_id also in asc order.
Assumptions:

There are no overlaps in the time frames for a given aws_cluster_id, meaning that a backup will end before another begins.
An 'End' event will always follow a 'Start' event for the same aws_cluster_id, but there may be multiple 'Start' events before an 'End' event if there are restarts.

**backup_events**  
id (integer)  
event_datetime (timestamp)  
aws_cluster_id (integer)  
backup_status (text)  

~~~sql
WITH backups_end_start
AS (
      SELECT *,
            (CASE WHEN backup_status = 'Start' THEN event_datetime END) AS start,
            (CASE WHEN backup_status = 'End' THEN event_datetime END) AS end_,
            MIN(CASE WHEN backup_status = 'End' THEN event_datetime END) OVER (
                                                                              PARTITION BY aws_cluster_id 
                                                                              ORDER BY event_datetime ASC 
                                                                              ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING
                                                                              ) AS backup_end --for each timestamp returns the end time of the backup event that it refers to
      FROM backup_events
      ORDER BY aws_cluster_id,
               event_datetime
      )
SELECT CAST(MIN(start) AS TEXT) AS start_time,
       CAST(MAX(end_) AS TEXT) AS end_time,
       aws_cluster_id,
       MAX(end_) - MIN(start) AS total_backup_duration,
       COUNT(*) - 2 AS number_of_restarts
FROM backups_end_start
GROUP BY aws_cluster_id,
         backup_end
ORDER BY start_time,
         aws_cluster_id;
~~~
