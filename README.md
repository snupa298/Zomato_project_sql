# SQL Project: Data Analysis for Zomato - A Food Delivery Company

## Overview

This project demonstrates my SQL problem-solving skills through the analysis of data for Zomato, a popular food delivery company in India. The project involves setting up the database, importing data, handling null values, and solving a variety of business problems using complex SQL queries.

use zomato;

select * from customers;
select * from restaurants;
select * from riders;
select * from deliveries;

## Data Import

## Data Cleaning and Handling Null Values.
select count(*) from customers
where customer_name is null
or
reg_date is null;

select count(*) from restaurants
where restaurant_name is null
or
city is null
or opening_hours is null;

select * from orders
where order_item is null
or
order_date is null
or
order_time is null
or
order_status is null
or
total_amount is null;

-- --------------------------------------------------
-- Analysis and reports
-- --------------------------------------------------
-- 1.Write the query to find the top 5 most frequently ordered dishes by customer "Arjun Mehta" in last 1 year.
SELECT 
	customer_name,
	dishes,
	total_orders
FROM -- table name
	(SELECT 
		c.customer_id,
		c.customer_name,
		o.order_item as dishes,
		COUNT(*) as total_orders,
		DENSE_RANK() OVER(ORDER BY COUNT(*) DESC) as rank1
	FROM orders as o
	JOIN
	customers as c
	ON c.customer_id = o.customer_id
	WHERE 
		o.order_date >= CURRENT_DATE - INTERVAL 2 YEAR
		AND 
		c.customer_name = 'Arjun Mehta'
	GROUP BY c.customer_id, c.customer_name, dishes
	ORDER BY c.customer_id,total_orders DESC) as t1
WHERE rank1 <= 5;

-- 2.Identify the time slots during which most orders are placed.based on 2 hour interval.

SELECT
    CASE
        WHEN EXTRACT(HOUR FROM order_time) BETWEEN 0 AND 1 THEN '00:00 - 02:00'
        WHEN EXTRACT(HOUR FROM order_time) BETWEEN 2 AND 3 THEN '02:00 - 04:00'
        WHEN EXTRACT(HOUR FROM order_time) BETWEEN 4 AND 5 THEN '04:00 - 06:00'
        WHEN EXTRACT(HOUR FROM order_time) BETWEEN 6 AND 7 THEN '06:00 - 08:00'
        WHEN EXTRACT(HOUR FROM order_time) BETWEEN 8 AND 9 THEN '08:00 - 10:00'
        WHEN EXTRACT(HOUR FROM order_time) BETWEEN 10 AND 11 THEN '10:00 - 12:00'
        WHEN EXTRACT(HOUR FROM order_time) BETWEEN 12 AND 13 THEN '12:00 - 14:00'
        WHEN EXTRACT(HOUR FROM order_time) BETWEEN 14 AND 15 THEN '14:00 - 16:00'
        WHEN EXTRACT(HOUR FROM order_time) BETWEEN 16 AND 17 THEN '16:00 - 18:00'
        WHEN EXTRACT(HOUR FROM order_time) BETWEEN 18 AND 19 THEN '18:00 - 20:00'
        WHEN EXTRACT(HOUR FROM order_time) BETWEEN 20 AND 21 THEN '20:00 - 22:00'
        WHEN EXTRACT(HOUR FROM order_time) BETWEEN 22 AND 23 THEN '22:00 - 00:00'
    END AS time_slot,
    COUNT(order_id) AS order_count
FROM Orders
GROUP BY time_slot
ORDER BY order_count DESC;

-- 3.Find the average order value per customer who has placed more than 750 orders.

select 
c.customer_name,
COUNT(*) as total_orders,
AVG(o.total_amount) as aov
from orders as o
join customers as c
on o.customer_id = c.customer_id
group by customer_name
having  COUNT(order_id) > 750;

-- 4.List the customers who have spent more than 100K in total on food orders

SELECT 
    c.customer_id,
    c.customer_name,
    SUM(o.total_amount) as tot_amt
FROM customers AS c
JOIN orders AS o
ON o.customer_id = c.customer_id
GROUP BY c.customer_id, c.customer_name
HAVING tot_amt >= 100000;

-- 5.Write a query to find orders that were placed but not delivered. 

SELECT 
	r.restaurant_name,
	COUNT(o.order_id) as cnt_not_delivered_orders
FROM orders as o
LEFT JOIN 
restaurants as r
ON r.restaurant_id = o.restaurant_id
LEFT JOIN
deliveries as d
ON d.order_id = o.order_id
WHERE d.delivery_id IS NULL
GROUP BY 1
ORDER BY 2 DESC;

-- 6. Rank restaurants by their total revenue from the last year, including their name, 
-- total revenue, and rank within their city.

WITH ranking_table
AS
(SELECT 
  r.city,
  r.restaurant_name,
  sum(o.total_amount) as revenue,
  rank() over(partition by r.city ORDER BY SUM(o.total_amount) DESC) as r1
FROM orders as o
LEFT JOIN 
restaurants as r
ON r.restaurant_id = o.restaurant_id
where o.order_date >= CURRENT_DATE - INTERVAL 2 YEAR
group by r.city,r.restaurant_name
)
SELECT 
	*
FROM ranking_table
WHERE r1 = 1;

-- 7.Identify the most popular dish in each city based on the number of orders.

SELECT * FROM
(SELECT 
  r.city,
  o.order_item,
  count(o.order_item) as item,
  rank() over(partition by r.city order by count(o.order_item) desc) as most_ordered
FROM orders as o
JOIN 
restaurants as r
ON r.restaurant_id = o.restaurant_id
group by r.city,o.order_item)
AS t1
WHERE most_ordered = 1;

-- 8.Find customers who haven’t placed an order in 2024 but did in 2023.
-- find cx who has done orders in 2023
-- find cx who has not done orders in 2024
-- compare 1 and 2

Select distinct customer_id from orders
where
EXTRACT(year from order_date) = 2023
AND customer_id NOT IN
(Select distinct customer_id from orders
where
EXTRACT(year from order_date) = 2024);

-- 9. Calculate and compare the order cancellation rate for each restaurant between the 
-- current year and the previous year.

WITH cancel_ratio_23
AS
(select 
o.restaurant_id,
count(o.order_id) as total_orders,
count(case when d.delivery_id is null then 1 end)as not_delivered
from orders as o
left join deliveries as d
on o.order_id = d.order_id
where extract(year from order_date) = 2023
group by o.restaurant_id
order by o.restaurant_id),
cancel_ratio_24
AS
(select 
o.restaurant_id,
count(o.order_id) as total_orders,
count(case when d.delivery_id is null then 1 end)as not_delivered
from orders as o
left join deliveries as d
on o.order_id = d.order_id
where extract(year from order_date) = 2024
group by o.restaurant_id
order by o.restaurant_id),
last_year_data as
(SELECT 
restaurant_id,
total_orders,
not_delivered,
round(not_delivered/total_orders * 100 ,2)as cancel_ratio
FROM cancel_ratio_23),
current_year_data as
(SELECT 
restaurant_id,
total_orders,
not_delivered,
round(not_delivered/total_orders * 100 ,2)as cancel_ratio
FROM cancel_ratio_24)
select
 c.restaurant_id AS restaurant_id,
    c.cancel_ratio AS current_year_cancel_ratio,
    l.cancel_ratio AS last_year_cancel_ratio
from current_year_data as c
join last_year_data as l
on c.restaurant_id = l.restaurant_id;

-- 10. Determine each rider's average delivery time.

with time_table as
(select 
o.order_id,
d.rider_id,
o.order_time,
d.delivery_time,
 Round(( TIME_TO_SEC(d.delivery_time) - TIME_TO_SEC(o.order_time) + case when
 d.delivery_time < o.order_time then 86400 else 0 end)/60,2) AS time_difference_minutes
from orders as o
join deliveries as d
on o.order_id = d.order_id
where d.delivery_status = "Delivered")
select
rider_id,
round(avg(time_difference_minutes),2)
from time_table
group by rider_id
order by rider_id;

-- 11. Monthly Restaurant Growth Ratio: 
-- Calculate each restaurant's growth ratio based on the total number of delivered orders since its joining.

with monthly_orders AS(
select 
o.restaurant_id,
 	EXTRACT(YEAR FROM o.order_date) as yr,
	EXTRACT(MONTH FROM o.order_date) as mnt,
    count(o.order_id) as cr_mnt
from orders as o
join deliveries as d
on o.order_id = d.order_id
where delivery_status = "Delivered"
group by o.restaurant_id,yr,mnt
order by o.restaurant_id,yr,mnt
),
last_month_orders AS(
SELECT 
    restaurant_id,
    yr,
    mnt,
    cr_mnt,
    LAG(cr_mnt, 1) OVER (PARTITION BY restaurant_id ORDER BY yr, mnt) AS last_mnt
FROM monthly_orders
ORDER BY restaurant_id, yr, mnt
)
SELECT
  restaurant_id,
    yr,
    mnt,
   
    LAG(cr_mnt, 1) OVER (PARTITION BY restaurant_id ORDER BY yr, mnt) AS last_mnt,
     cr_mnt,
    ROUND((cr_mnt-last_mnt)/last_mnt * 100,2) as growth_ratio
    from last_month_orders;
    
-- 12.Customer Segmentation: Segment customers into 'Gold' or 'Silver' groups based on their total spending 
-- compared to the average order value (AOV). If a customer's total spending exceeds the AOV, 
-- label them as 'Gold'; otherwise, label them as 'Silver'. Write an SQL query to determine each segment's 
-- total number of orders and total revenue    

SELECT
cx_category,
SUM(total_spent) as tot_revenue,
SUM(tot_order) as tot_orders
FROM
(select
customer_id,
SUM(total_amount) as total_spent,
count(order_id) as tot_order,
CASE when SUM(total_amount) > (select avg(total_amount) from orders) THEN 'Gold' ELSE
'Silver' END as cx_category
from orders
group by customer_id) as t1
group by cx_category;

-- 13.Rider Monthly Earnings: 
-- Calculate each rider's total monthly earnings, assuming they earn 8% of the order amount.

select 
d.rider_id,
EXTRACT(year from order_date)as yr,
EXTRACT(month from order_date)as mnt,
(SUM(total_amount)*8)/100 as amtper_rider
from orders as o
join deliveries as d 
on o.order_id = d.order_id
group by d.rider_id,yr,mnt
order by d.rider_id,yr,mnt;

-- 14.Find the number of 5-star, 4-star, and 3-star ratings each rider has.
-- riders receive this rating based on delivery time.
-- If orders are delivered less than 15 minutes of order received time the rider get 5 star rating,
-- if they deliver 15 and 20 minute they get 4 star rating 
-- if they deliver after 20 minute they get 3 star rating.
SELECT
rider_id,
stars,
count(*) as total_stars
FROM
(SELECT
rider_id,
delivery_took_time,
	CASE 
			WHEN delivery_took_time < 15 THEN '5 star'
			WHEN delivery_took_time BETWEEN 15 AND 20 THEN '4 star'
			ELSE '3 star'
		END as stars
FROM
(select 
d.rider_id,
o.order_id,
o.order_time,
d.delivery_time,
Round(( TIME_TO_SEC(d.delivery_time) - TIME_TO_SEC(o.order_time) + case when
 d.delivery_time < o.order_time then 86400 else 0 end)/60,2) AS delivery_took_time
from orders as o
join deliveries as d 
on o.order_id = d.order_id
where delivery_status = 'Delivered'
) as t1
) as t2
group by rider_id,stars
order by rider_id,total_stars desc;

-- 15.Order Frequency by Day: 
-- Analyze order frequency per day of the week and identify the peak day for each restaurant.

SELECT
restaurant_name,
day_of_week,
total_orders
FROM
(SELECT
    r.restaurant_name,
    DAYNAME(o.order_date) AS day_of_week,
    COUNT(o.order_id) AS total_orders,
    RANK() over(partition by   r.restaurant_name order by COUNT(o.order_id) desc)as rk
FROM orders AS o
JOIN restaurants AS r
ON o.restaurant_id = r.restaurant_id
GROUP BY r.restaurant_name, day_of_week
ORDER BY r.restaurant_name, total_orders desc
) as t1
WHERE rk = 1;

-- 16. Customer Lifetime Value (CLV): 
-- Calculate the total revenue generated by each customer over all their orders.

SELECT 
	o.customer_id,
	c.customer_name,
	SUM(o.total_amount) as CLV
FROM orders as o
JOIN customers as c
ON o.customer_id = c.customer_id
GROUP BY 1, 2;

-- 17.Monthly Sales Trends: 
-- Identify sales trends by comparing each month's total sales to the previous month.

SELECT 
	EXTRACT(YEAR FROM order_date) as year,
	EXTRACT(MONTH FROM order_date) as month,
	SUM(total_amount) as total_sale,
	LAG(SUM(total_amount), 1) OVER(ORDER BY EXTRACT(YEAR FROM order_date), EXTRACT(MONTH FROM order_date)) as prev_month_sale
FROM orders
GROUP BY 1, 2;

-- 18.Rider Efficiency: 
-- Evaluate rider efficiency by determining average delivery times and identifying those with the lowest and highest averages.

WITH new_table as
(select 
d.rider_id,
Round(( TIME_TO_SEC(d.delivery_time) - TIME_TO_SEC(o.order_time) + case when
 d.delivery_time < o.order_time then 86400 else 0 end)/60,2) AS delivery_took_time
from orders as o
join deliveries as d
on o.order_id = d.delivery_id
WHERE d.delivery_status = 'Delivered'
),
riders_time as
(
select 
rider_id,
AVG(delivery_took_time) as avg_time
from new_table
group by rider_id
)
SELECT 
	MIN(avg_time),
	MAX(avg_time)
FROM riders_time;

-- 19. Order Item Popularity: 
-- Track the popularity of specific order items over time and identify seasonal demand spikes.

SELECT 
	order_item,
	seasons,
	COUNT(order_id) as total_orders
FROM 
(
SELECT 
		*,
		EXTRACT(MONTH FROM order_date) as month,
		CASE 
			WHEN EXTRACT(MONTH FROM order_date) BETWEEN 4 AND 6 THEN 'Spring'
			WHEN EXTRACT(MONTH FROM order_date) > 6 AND 
			EXTRACT(MONTH FROM order_date) < 9 THEN 'Summer'
			ELSE 'Winter'
		END as seasons
	FROM orders
) as t1
GROUP BY 1, 2
ORDER BY 1, 3 DESC;

-- 20.Rank each city based on the total revenue for last year 2023 .
    
    SELECT 
	r.city,
	SUM(total_amount) as total_revenue,
	RANK() OVER(ORDER BY SUM(total_amount) DESC) as city_rank
FROM orders as o
JOIN
restaurants as r
ON o.restaurant_id = r.restaurant_id
GROUP BY 1;
