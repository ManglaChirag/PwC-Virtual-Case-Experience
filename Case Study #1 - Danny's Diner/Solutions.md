# Solutions: Case Study \#1


### 1. What is the total amount each customer spent at the restaurant?

```SQL
SELECT S.CUSTOMER_ID,
	SUM(PRICE) AS TOTAL_SPEND
FROM SALES S
JOIN MENU M 
ON S.PRODUCT_ID = M.PRODUCT_ID
GROUP BY CUSTOMER_ID
ORDER BY S.CUSTOMER_ID; 
```
### Output:
| **customer_id** | **total_spend** |
|-----------------|-----------------|
| A               | 76              |
| B               | 74              |
| C               | 36              |
***
### 2. How many days has each customer visited the restaurant?

```sql
SELECT CUSTOMER_ID,
	COUNT(DISTINCT(ORDER_DATE)) AS VISITS
FROM SALES
GROUP BY CUSTOMER_ID;
```
### Output:
| **customer_id** | **visits** |
|-----------------|------------|
| A               | 4          |
| B               | 6          |
| C               | 2          |
***
### 3. What was the first item from the menu purchased by each customer?

```sql
SELECT CUSTOMER_ID,PRODUCT_NAME
FROM
	(SELECT *,
			ROW_NUMBER() OVER(PARTITION BY CUSTOMER_ID
				       	       ORDER BY ORDER_DATE) AS RANK
		FROM SALES) AS R
JOIN MENU M 
ON R.PRODUCT_ID = M.PRODUCT_ID
WHERE RANK = 1;
```
### Output:
| **customer_id** | **product_name** |
|-----------------|------------------|
| A               | sushi            |
| B               | curry            |
| C               | ramen            |
***
### 4. What is the most purchased item on the menu and how many times was it purchased by all customers?

```sql
SELECT PRODUCT_NAME,
	ROW_NUMBER() OVER(PARTITION BY PRODUCT_NAME) AS ORDERS
FROM SALES S
JOIN MENU M ON S.PRODUCT_ID = M.PRODUCT_ID
ORDER BY ORDERS DESC
LIMIT 1;
```
### Output:
| **product_name** | **orders** |
|------------------|------------|
| ramen            | 8          |
***
### 5. Which item was the most popular for each customer?

```sql
WITH FAV_ITEM AS
	(SELECT S.CUSTOMER_ID,
			M.PRODUCT_NAME,
			COUNT(M.PRODUCT_ID) AS ORDER_COUNT,
			RANK() OVER(PARTITION BY S.CUSTOMER_ID
															ORDER BY COUNT(S.CUSTOMER_ID) DESC) AS RANK
		FROM MENU AS M
		JOIN SALES AS S ON M.PRODUCT_ID = S.PRODUCT_ID
		GROUP BY S.CUSTOMER_ID,
			M.PRODUCT_NAME)
SELECT CUSTOMER_ID,
	PRODUCT_NAME
FROM FAV_ITEM
WHERE RANK = 1;
```
### Output:
| **customer_id** | **product_name** |
|-----------------|------------------|
| A               | ramen            |
| B               | sushi            |
| B               | curry            |
| B               | ramen            |
| C               | ramen            |
***
### Which item was purchased first by the customer after they became a member?

```sql
WITH ORDER_RANK AS
	(SELECT M.CUSTOMER_ID,
			S.PRODUCT_ID,
			RANK() OVER(PARTITION BY M.CUSTOMER_ID
															ORDER BY ORDER_DATE) AS RANK
		FROM MEMBERS M
		LEFT JOIN SALES S ON M.CUSTOMER_ID = S.CUSTOMER_ID
		WHERE ORDER_DATE >= JOIN_DATE )
SELECT CUSTOMER_ID,
	PRODUCT_NAME AS DISH
FROM ORDER_RANK O
LEFT JOIN MENU ME ON O.PRODUCT_ID = ME.PRODUCT_ID
WHERE RANK = 1
ORDER BY CUSTOMER_ID;
```
### Output:
| **customer_id** | **dish** |
|-----------------|----------|
| A               | curry    |
| B               | sushi    |
***
### 7. Which item was purchased just before the customer became a member?

```sql
WITH ORDER_RANK AS
	(SELECT M.CUSTOMER_ID,
			S.PRODUCT_ID,
			RANK() OVER(PARTITION BY M.CUSTOMER_ID
															ORDER BY ORDER_DATE DESC) AS RANK
		FROM MEMBERS M
		LEFT JOIN SALES S ON M.CUSTOMER_ID = S.CUSTOMER_ID
		WHERE ORDER_DATE < JOIN_DATE )
SELECT CUSTOMER_ID,
	PRODUCT_NAME AS DISH
FROM ORDER_RANK O
LEFT JOIN MENU ME ON O.PRODUCT_ID = ME.PRODUCT_ID
WHERE RANK = 1
ORDER BY CUSTOMER_ID;
```
### Output:
| **customer_id** | **dish** |
|-----------------|----------|
| A               | sushi    |
| A               | curry    |
| B               | sushi    |
***
### 8. What is the total items and amount spent for each member before they became a member?

```sql
WITH ITEMS AS
	(SELECT S.CUSTOMER_ID,
			PRODUCT_ID
		FROM MEMBERS ME
		LEFT JOIN SALES S ON ME.CUSTOMER_ID = S.CUSTOMER_ID
		WHERE ORDER_DATE < JOIN_DATE)
SELECT DISTINCT(I.CUSTOMER_ID),
	COUNT(I.PRODUCT_ID) OVER(PARTITION BY I.CUSTOMER_ID),
	SUM(M.PRICE) OVER(PARTITION BY I.CUSTOMER_ID) TOTAL_SPEND
FROM ITEMS I
JOIN MENU M ON I.PRODUCT_ID = M.PRODUCT_ID
```
### Output:
| **customer_id** | **count** | **total_spend** |
|-----------------|-----------|-----------------|
| B               | 3         | 40              |
| A               | 2         | 25              |
***
### 9.  If each $1 spent equates to 10 points and sushi has a 2x points multiplier - how many points would each customer have?
#### Assuming we are considering records after customers become members

```sql
WITH Member_orders AS
	(SELECT M.CUSTOMER_ID,
			S.PRODUCT_ID
		FROM MEMBERS M
		LEFT JOIN SALES S ON M.CUSTOMER_ID = S.CUSTOMER_ID
		WHERE ORDER_DATE >= JOIN_DATE )
SELECT CUSTOMER_ID,
	sum(case
		when mo.product_id=1 then price*20
		else price*10
	end) as points	
FROM member_orders mo
LEFT JOIN menu me ON mo.PRODUCT_ID = ME.PRODUCT_ID
group by customer_id
```
### Output:
| **customer_id** | **points** |
|-----------------|------------|
| B               | 440        |
| A               | 510        |
***
### 10. In the first week after a customer joins the program (including their join date) they earn 2x points on all items, not just sushi - how many points do customer A and B have at the end of January?

```sql
WITH ELIGIBLE_ORDERS AS
	(SELECT S.CUSTOMER_ID,
			S.PRODUCT_ID,
			S.ORDER_DATE,
			ME.JOIN_DATE
		FROM MEMBERS ME
		LEFT JOIN SALES S ON ME.CUSTOMER_ID = S.CUSTOMER_ID
		WHERE ORDER_DATE >= JOIN_DATE
			AND EXTRACT(MONTH FROM ORDER_DATE) = 1)
SELECT CUSTOMER_ID,
	SUM(CASE
			WHEN (EO.ORDER_DATE - EO.JOIN_DATE) < 7 THEN PRICE * 20
			WHEN EO.PRODUCT_ID = 1 THEN PRICE * 20
			ELSE PRICE * 10
					END) AS POINTS
FROM ELIGIBLE_ORDERS EO
LEFT JOIN MENU M ON EO.PRODUCT_ID = M.PRODUCT_ID
GROUP BY CUSTOMER_ID
```
### Output:
| **customer_id** | **points** |
|-----------------|------------|
| B               | 320        |
| A               | 1020       |
