## Case Study #1 - Danny's Diner

<img src="https://8weeksqlchallenge.com/images/case-study-designs/1.png" width=30% height=30%>

## Entity Relationship Diagram

<img src="https://github.com/alexis-anzaldo/8-week-SQL-challenge/blob/main/images/Week_1_entity_relationship_diagram.png?raw=true" width=60% height=60%>


## 1. What is the total amount each customer spent at the restaurant?

~~~~sql
SELECT
    sales.customer_id,
    SUM(menu.price) AS total
FROM dannys_diner.sales
LEFT JOIN dannys_diner.menu
    ON sales.product_id = menu.product_id
GROUP BY
    sales.customer_id
ORDER BY
    sales.customer_id ASC
LIMIT 5;
~~~~

|customer_id|total|
|------------|--------|
| A          |	76    |
| B          |	74    |
| C          |	36    |


## 2. How many days has each customer visited the restaurant?

~~~~sql
SELECT
  	customer_id,
    COUNT (DISTINCT order_date) as visits
FROM dannys_diner.sales
GROUP BY
	customer_id
ORDER BY
	customer_id ASC
LIMIT 5;
~~~~


|customer_id|visits|
|-|-|
|A|4|
|B|6|
|C|2|



## 3. What was the first item from the menu purchased by each customer?

~~~~sql
SELECT sales.customer_id, menu.product_name AS first_item_purchased
FROM dannys_diner.sales
JOIN dannys_diner.menu ON sales.product_id = menu.product_id
WHERE sales.order_date = (
    SELECT MIN(order_date)
    FROM dannys_diner.sales
    WHERE customer_id = sales.customer_id
    )
GROUP BY sales.customer_id, menu.product_name
ORDER BY
	sales.customer_id ASC;
~~~~

|customer_id|first_item_purchased|
|-|-|
|A|curry|
|A|sushi|
|B|curry|
|C|ramen|




## 4. What is the most purchased item on the menu and how many times was it purchased by all customers?

~~~sql
SELECT
     menu.product_name,
     COUNT(menu.product_name) AS purchases
FROM dannys_diner.menu
LEFT JOIN dannys_diner.sales ON sales.product_id = menu.product_id
GROUP BY menu.product_name
ORDER BY purchases DESC
LIMIT 1;
~~~

## 5. Which item was the most popular for each customer?

~~~sql
WITH max_purchases_cte AS (
    SELECT
        sales.customer_id,
        menu.product_name,
        COUNT(menu.product_name) AS purchases
    FROM dannys_diner.menu
    LEFT JOIN dannys_diner.sales ON sales.product_id = menu.product_id
    GROUP BY sales.customer_id, menu.product_name
)
SELECT customer_id, product_name, purchases
FROM max_purchases_cte
WHERE (customer_id, purchases) IN (
    SELECT customer_id, MAX(purchases)
    FROM max_purchases_cte
    GROUP BY customer_id
)
ORDER BY customer_id;
~~~

## 6. Which item was purchased first by the customer after they became a member?

~~~sql
WITH members_purchases_cte AS (
  SELECT sales.customer_id,
         sales.order_date,
  		 menu.product_name,
  		 ROW_NUMBER() OVER (PARTITION BY sales.customer_id ORDER BY sales.order_date) AS c_order
  FROM dannys_diner.sales
  JOIN dannys_diner.menu ON sales.product_id = menu.product_id
  JOIN dannys_diner.members ON sales.customer_id = members.customer_id
  WHERE sales.order_date > members.join_date
)
SELECT customer_id,
	   order_date,
       product_name
FROM members_purchases_cte
WHERE c_order = 1;
~~~

## 7. Which item was purchased just before the customer became a member?

~~~sql
WITH members_purchases_cte AS (
  SELECT sales.customer_id,
         sales.order_date,
  		 menu.product_name,
  		 ROW_NUMBER() OVER (PARTITION BY sales.customer_id ORDER BY sales.order_date DESC) AS c_order
  FROM dannys_diner.sales
  JOIN dannys_diner.menu ON sales.product_id = menu.product_id
  JOIN dannys_diner.members ON sales.customer_id = members.customer_id
  WHERE sales.order_date < members.join_date
)
SELECT customer_id,
	   order_date,
       product_name
FROM members_purchases_cte
WHERE c_order = 1;
~~~

## 8. What is the total items and amount spent for each member before they became a member?

~~~sql
WITH members_purchases_cte AS (
  SELECT sales.customer_id,
  		 menu.product_name,
  		 menu.price,
  		COUNT(product_name) AS product_purchase
  FROM dannys_diner.sales
  JOIN dannys_diner.menu ON sales.product_id = menu.product_id
  JOIN dannys_diner.members ON sales.customer_id = members.customer_id
  WHERE sales.order_date < members.join_date
  GROUP BY sales.customer_id, menu.product_name, menu.price
)
SELECT customer_id,
	   SUM(product_purchase) AS total_items,
       SUM(price*product_purchase) AS total
FROM members_purchases_cte
GROUP BY customer_id
ORDER BY customer_id;
~~~

## 9.  If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?

~~~sql
WITH members_purchases_cte AS (
  SELECT sales.customer_id,
  		 menu.product_name,
  		 menu.price,
  		COUNT(product_name) AS product_purchase,
  		CASE menu.product_name
  			WHEN 'sushi' THEN 2 ELSE 1 END AS bonus
  FROM dannys_diner.sales
  JOIN dannys_diner.menu ON sales.product_id = menu.product_id
  GROUP BY sales.customer_id, menu.product_name, menu.price
)
SELECT customer_id,
       SUM(price*product_purchase*bonus*10) AS points
FROM members_purchases_cte
GROUP BY customer_id
ORDER BY customer_id;
~~~

## 10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?

~~~sql
WITH dates_cte AS (
    SELECT
        m.customer_id,
        m.join_date,
        m.join_date + 6 AS valid_date,
        DATE_TRUNC('month', '2021-01-31'::DATE) + interval '1 month' - interval '1 day' AS last_date
    FROM dannys_diner.members AS m
), members_purchases_cte AS (
    SELECT s.customer_id,
  		s.order_date,
        menu.product_name,
        menu.price,
        CASE menu.product_name
            WHEN 'sushi' THEN 2 ELSE 1 END AS bonus
    FROM dannys_diner.sales AS s
    JOIN dannys_diner.menu ON s.product_id = menu.product_id
)

SELECT customer_id,
       SUM(CASE
           WHEN wk.order_date >= wk.join_date AND wk.order_date <= wk.valid_date THEN wk.price * 2 * 10
       	   WHEN wk.order_date > wk.valid_date AND wk.order_date <= wk.last_date THEN wk.price * wk.bonus * 10
           WHEN wk.order_date < wk.join_date THEN wk.price * wk.bonus * 10 ELSE 0
       END) AS total
FROM (
    SELECT mpc.*, dc.join_date, dc.valid_date, dc.last_date
    FROM members_purchases_cte AS mpc
    JOIN dates_cte AS dc ON mpc.customer_id = dc.customer_id
) AS wk
GROUP BY wk.customer_id;
~~~

## BONUS QUESTION

### Join All The Things
### The following questions are related creating basic data tables that Danny and his team can use to quickly derive insights without needing to join the underlying tables using SQL.

### Recreate the table with: customer_id, order_date, product_name, price, member (Y/N)

~~~sql
SELECT
    sales.customer_id,
    sales.order_date,
    menu.product_name,
    menu.price,
    CASE WHEN sales.order_date >= members.join_date THEN 'Y' ELSE 'N' END AS member
FROM dannys_diner.sales
JOIN dannys_diner.menu ON sales.product_id = menu.product_id
JOIN dannys_diner.members ON sales.customer_id = members.customer_id
ORDER BY customer_id, order_date, product_name;
~~~

