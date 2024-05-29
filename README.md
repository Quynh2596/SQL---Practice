# Case Study 1: Danny's Diner 


Please note that all the information regarding the case study has been sourced from the following link: [here](https://8weeksqlchallenge.com/case-study-1/). 

***

## Business Task
Danny wants to use the data to answer a few simple questions about his customers, especially about their visiting patterns, how much money they’ve spent and also which menu items are their favourite. 

***

## Entity Relationship Diagram

![image](https://user-images.githubusercontent.com/81607668/127271130-dca9aedd-4ca9-4ed8-b6ec-1e1920dca4a8.png)

***

**Schema (PostgreSQL)**
````sql
    CREATE SCHEMA dannys_diner;
    SET search_path = dannys_diner;
    
    CREATE TABLE sales (
      "customer_id" VARCHAR(1),
      "order_date" DATE,
      "product_id" INTEGER
    );
    
    INSERT INTO sales
      ("customer_id", "order_date", "product_id")
    VALUES
      ('A', '2021-01-01', '1'),
      ('A', '2021-01-01', '2'),
      ('A', '2021-01-07', '2'),
      ('A', '2021-01-10', '3'),
      ('A', '2021-01-11', '3'),
      ('A', '2021-01-11', '3'),
      ('B', '2021-01-01', '2'),
      ('B', '2021-01-02', '2'),
      ('B', '2021-01-04', '1'),
      ('B', '2021-01-11', '1'),
      ('B', '2021-01-16', '3'),
      ('B', '2021-02-01', '3'),
      ('C', '2021-01-01', '3'),
      ('C', '2021-01-01', '3'),
      ('C', '2021-01-07', '3');
     
    
    CREATE TABLE menu (
      "product_id" INTEGER,
      "product_name" VARCHAR(5),
      "price" INTEGER
    );
    
    INSERT INTO menu
      ("product_id", "product_name", "price")
    VALUES
      ('1', 'sushi', '10'),
      ('2', 'curry', '15'),
      ('3', 'ramen', '12');
      
    
    CREATE TABLE members (
      "customer_id" VARCHAR(1),
      "join_date" DATE
    );
    
    INSERT INTO members
      ("customer_id", "join_date")
    VALUES
      ('A', '2021-01-07'),
      ('B', '2021-01-09');
````
---

**Question 1: What is the total amount each customer spent at the restaurant?**
````sql
    SELECT sales.customer_id, SUM((menu.price))
FROm sales JOIN menu
ON sales.product_id = menu.product_id
GROUP BY sales.customer_id;
````
#### Answer:
| customer_id | total_spent |
| ----------- | ----------- |
| A           | 76          |
| B           | 74          |
| C           | 36          |

---

**Question 2: How many days has each customer visited the restaurant?**

````sql
SELECT customer_id, COUNT(DISTINCT order_date)
FROM sales
GROUP BY customer_id
ORDER BY customer_id;
````
#### Answer:
| customer_id | visit_count |
| ----------- | ----------- |
| A           | 4          |
| B           | 6          |
| C           | 2          |

---

**Question 3: What was the first item from the menu purchased by each customer?**

````sql
SELECT DISTINCT sales.customer_id, menu.product_name
FROM sales 
LEFT JOIN menu ON sales.product_id = menu.product_id
WHERE sales.order_date = (SELECT MIN(min_order_date) 
                          FROM (SELECT MIN(order_date) min_order_date
                                FROM sales
                                GROUP BY customer_id)
                         )
ORDER BY sales.customer_id;

````
#### Answer:
| customer_id | product_name | 
| ----------- | ----------- |
| A           | curry        | 
| A           | sushi        | 
| B           | curry        | 
| C           | ramen        |

---

**Question 4: What is the most purchased item on the menu and how many times was it purchased by all customers?**

````sql
SELECT menu.product_name, COUNT(sales.product_id)
FROM sales LEFT JOIN menu ON sales.product_id = menu.product_id
GROUP BY menu.product_name
HAVING count(sales.product_id) = (SELECT MAX(pp) 
                                  FROM (SELECT count(product_id) AS pp 
                                        FROM sales 
                                        GROUP BY product_id)
                                 );

````
#### Answer:
| product_name | count | 
| ----------- | ----------- |
| ramen       | 8 |

---

**Question 5: What is the most purchased item on the menu and how many times was it purchased by all customers?**
````sql
WITH rank_table AS
 (
   SELECT sales.customer_id, menu.product_name, COUNT(*),
   RANK () OVER (PARTITION BY sales.customer_id ORDER BY COUNT(*)DESC) ranking_
   FROM sales JOIN menu on sales.product_id = menu.product_id
   GROUP BY sales.customer_id, menu.product_name
   )
SELECT * FROM rank_table
WHERE ranking_ = 1;

````
#### Answer:
| customer_id | product_name  | count      |
| ----------- | ------------- | --------   |
| A           | ramen         |     3      |
| B           | ramen         |     2      |
| B           | curry         |     2      |
| B           | sushi         |     2      |
| C           | ramen         |     3      |

---
**Question 6: Which item was purchased first by the customer after they became a member?**
````sql
WITH order_abm AS (
SELECT sales.customer_id, sales.order_date, sales.product_id, members.join_date
FROM members RIGHT JOIN sales ON members.customer_id=sales.customer_id
WHERE sales.order_date>=members.join_date
)

SELECT order_abm.customer_id, menu.product_name
FROM order_abm
JOIN
(SELECT customer_id, MIN(order_date) earliest_date
FROM order_abm
GROUP BY customer_id) T
ON order_abm.customer_id = T.customer_id
AND order_abm.order_date = T.earliest_date
JOIN menu ON menu.product_id=order_abm.product_id;

````

#### Answer:
| customer_id | product_name |
| ----------- | ------------ |
| A           | curry        |
| B           | sushi        |

---
**Question 7: Which item was purchased just before the customer became a member?**
````sql
WITH order_bbm AS (
SELECT sales.customer_id, sales.order_date, sales.product_id, members.join_date
FROM members JOIN sales ON members.customer_id=sales.customer_id
WHERE sales.order_date<members.join_date
)

SELECT order_bbm.customer_id, menu.product_name
FROM order_bbm
JOIN
(SELECT customer_id, MAX(order_date) latest_date
FROM order_bbm
GROUP BY customer_id) T
ON order_bbm.customer_id = T.customer_id
AND order_bbm.order_date = T.latest_date
JOIN menu ON menu.product_id=order_bbm.product_id;
````

#### Answer:
| customer_id | product_name |
| ----------- | ------------ |
| A           | curry        |
| A           | sushi        |
| B           | sushi        |

---
**Question 8: What is the total items and amount spent for each member before they became a member?**
````sql
SELECT sales.customer_id, SUM(menu.price) total_spent, COUNT(sales.product_id) total_items
FROM sales
JOIN  menu ON menu.product_id=sales.product_id
JOIN members ON members.customer_id = sales.customer_id
WHERE members.join_date<sales.order_date
GROUP BY sales.customer_id;

````

#### Answer:
| customer_id | total_items | total_spent |
| ----------- | ---------- |----------  |
| A           | 3 |  36      |
| B           | 3 |  34      |

---
**Question 9: If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?**
````sql
SELECT sales.customer_id, SUM(
CASE
  WHEN menu.product_name = 'sushi' THEN 20 * menu.price
  ELSE 10 * menu.price
  END) total_point
FROM sales  LEFT JOIN menu
ON sales.product_id = menu.product_id
GROUP BY sales.customer_id;

````

#### Answer:
| customer_id | total_points | 
| ----------- | ---------- |
| A           | 860 |
| B           | 940 |
| C           | 360 |

---
**Question 10: In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?**
````sql
SELECT sales.customer_id, SUM(
CASE
  WHEN members.join_date <= sales.order_date
  AND members.join_date + 6 >= sales.order_date
  THEN 20 * menu.price
  WHEN menu.product_name = 'sushi'
  AND members.join_date + 6 < sales.order_date
  AND members.join_date > sales.order_date
  THEN 20 * menu.price
  ELSE 10 * menu.price
  END) total_point
FROM sales JOIN menu
ON sales.product_id = menu.product_id
JOIN members ON members.customer_id = sales.customer_id
WHERE sales.order_date <= '2021-01-31'
AND sales.order_date >= members.join_date
GROUP BY sales.customer_id;

````
#### Answer:

| customer_id | total_points |
| ----------- | ------------ |
| A           | 1020         |
| B           | 320          |

---



