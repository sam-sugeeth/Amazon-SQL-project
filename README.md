---

# **Amazon USA Sales Analysis Project**

### **Difficulty Level: Advanced**

---

## **Project Overview**

I have worked on analyzing a dataset of over 20,000 sales records from an Amazon-like e-commerce platform. This project involves extensive querying of customer behavior, product performance, and sales trends using PostgreSQL. Through this project, I have tackled various SQL problems, including revenue analysis, customer segmentation, and inventory management.

The project also focuses on data cleaning, handling null values, and solving real-world business problems using structured queries.

An ERD diagram is included to visually represent the database schema and relationships between tables.

---

![ERD Scratch](https://github.com/najirh/amazon_usa_project5/blob/main/erd2.png)

## **Database Setup & Design**

### **Schema Structure**

```sql
CREATE TABLE category
(
  category_id	INT PRIMARY KEY,
  category_name VARCHAR(20)
);

-- customers TABLE
CREATE TABLE customers
(
  customer_id INT PRIMARY KEY,	
  first_name	VARCHAR(20),
  last_name	VARCHAR(20),
  state VARCHAR(20),
  address VARCHAR(5) DEFAULT ('xxxx')
);

-- sellers TABLE
CREATE TABLE sellers
(
  seller_id INT PRIMARY KEY,
  seller_name	VARCHAR(25),
  origin VARCHAR(15)
);

-- products table
  CREATE TABLE products
  (
  product_id INT PRIMARY KEY,	
  product_name VARCHAR(50),	
  price	FLOAT,
  cogs	FLOAT,
  category_id INT, -- FK 
  CONSTRAINT product_fk_category FOREIGN KEY(category_id) REFERENCES category(category_id)
);

-- orders
CREATE TABLE orders
(
  order_id INT PRIMARY KEY, 	
  order_date	DATE,
  customer_id	INT, -- FK
  seller_id INT, -- FK 
  order_status VARCHAR(15),
  CONSTRAINT orders_fk_customers FOREIGN KEY (customer_id) REFERENCES customers(customer_id),
  CONSTRAINT orders_fk_sellers FOREIGN KEY (seller_id) REFERENCES sellers(seller_id)
);

CREATE TABLE order_items
(
  order_item_id INT PRIMARY KEY,
  order_id INT,	-- FK 
  product_id INT, -- FK
  quantity INT,	
  price_per_unit FLOAT,
  CONSTRAINT order_items_fk_orders FOREIGN KEY (order_id) REFERENCES orders(order_id),
  CONSTRAINT order_items_fk_products FOREIGN KEY (product_id) REFERENCES products(product_id)
);

-- payment TABLE
CREATE TABLE payments
(
  payment_id	
  INT PRIMARY KEY,
  order_id INT, -- FK 	
  payment_date DATE,
  payment_status VARCHAR(20),
  CONSTRAINT payments_fk_orders FOREIGN KEY (order_id) REFERENCES orders(order_id)
);

CREATE TABLE shippings
(
  shipping_id	INT PRIMARY KEY,
  order_id	INT, -- FK
  shipping_date DATE,	
  return_date	 DATE,
  shipping_providers	VARCHAR(15),
  delivery_status VARCHAR(15),
  CONSTRAINT shippings_fk_orders FOREIGN KEY (order_id) REFERENCES orders(order_id)
);

CREATE TABLE inventory
(
  inventory_id INT PRIMARY KEY,
  product_id INT, -- FK
  stock INT,
  warehouse_id INT,
  last_stock_date DATE,
  CONSTRAINT inventory_fk_products FOREIGN KEY (product_id) REFERENCES products(product_id)
  );
```

---

## **Task: Data Cleaning**

I cleaned the dataset by:
- **Removing duplicates**: Duplicates in the customer and order tables were identified and removed.
- **Handling missing values**: Null values in critical fields (e.g., customer address, payment status) were either filled with default values or handled using appropriate methods.

---

## **Handling Null Values**

Null values were handled based on their context:
- **Customer addresses**: Missing addresses were assigned default placeholder values.
- **Payment statuses**: Orders with null payment statuses were categorized as “Pending.”
- **Shipping information**: Null return dates were left as is, as not all shipments are returned.

---

## **Objective**

The primary objective of this project is to showcase SQL proficiency through complex queries that address real-world e-commerce business challenges. The analysis covers various aspects of e-commerce operations, including:
- Customer behavior
- Sales trends
- Inventory management
- Payment and shipping analysis
- Forecasting and product performance
  

## **Identifying Business Problems**

Key business problems identified:
1. Low product availability due to inconsistent restocking.
2. High return rates for specific product categories.
3. Significant delays in shipments and inconsistencies in delivery times.
4. High customer acquisition costs with a low customer retention rate.

---

## **Solving Business Problems**

### Solutions Implemented:
1. Top Selling Products
Query the top 10 products by total sales value.
Challenge: Include product name, total quantity sold, and total sales value.

```sql
ALTER TABLE order_items
	ADD total_value INT;
    
UPDATE order_items
	set total_value = price_per_unit * quantity;
select * from order_items;
    
SELECT 
	oi.product_id,
	p.product_name,
	SUM(oi.total_value) as total_value,
	COUNT(o.order_id)  as total_orders
FROM order_items oi
JOIN orders o ON oi.order_id = o.order_id
JOIN products p ON oi.product_id = p.product_id
GROUP BY p.product_name, p.product_id
ORDER BY SUM(oi.total_value) desc
LIMIT 10;
```

2. Revenue by Category
Calculate total revenue generated by each product category.
Challenge: Include the percentage contribution of each category to total revenue.

```sql
SELECT 
	c.category_id,
	c.category_name,
	SUM(oi.total_value) as total_sale,
    SUM(oi.total_value)/(SELECT SUM(total_value) FROM order_items) * 100 as contributions
FROM order_items as oi
JOIN
products as p
ON p.product_id = oi.product_id
LEFT JOIN category as c
ON c.category_id = p.category_id
GROUP BY c.category_name, c.category_id;

```

3. Average Order Value (AOV)
Compute the average order value for each customer.
Challenge: Include only customers with more than 5 orders.

```sql
SELECT 
	c.customer_id,
	CONCAT(c.first_name, ' ',  c.last_name) as full_name,
	SUM(total_value)/COUNT(o.order_id) as AOV,
	COUNT(o.order_id) as total_orders
FROM orders as o
JOIN 
customers as c
ON c.customer_id = o.customer_id
JOIN 
order_items as oi
ON oi.order_id = o.order_id
GROUP BY c.customer_id, full_name
HAVING  COUNT(o.order_id) > 5;
```

4. Monthly Sales Trend
Query monthly total sales over the past year.
Challenge: Display the sales trend, grouping by month, return current_month sale, last month sale!

```sql
SELECT 
    year,
    month,
    total_sale AS current_month_sale,
    LAG(total_sale, 1) OVER (ORDER BY year, month) AS last_month_sale
FROM 
(
    SELECT 
        EXTRACT(MONTH FROM o.order_date) AS month,
        EXTRACT(YEAR FROM o.order_date) AS year,
        ROUND(SUM(CAST(oi.total_value AS DECIMAL(10,2))), 2) AS total_sale  -- Fixed conversion
    FROM orders AS o
    JOIN order_items AS oi
    ON oi.order_id = o.order_id
    WHERE o.order_date >= CURDATE() - INTERVAL 2 YEAR  -- MySQL uses CURDATE()
    GROUP BY year, month
    ORDER BY year, month
) AS t1;
```


5. Customers with No Purchases
Find customers who have registered but never placed an order.
Challenge: List customer details and the time since their registration.

```sql
select * from orders;
select * from customers c
left join orders o ON c.customer_id = o.customer_id
where order_id is NULL; 
```

6. Least-Selling Categories by State
Identify the least-selling product category for each state.
Challenge: Include the total sales for that category within each state.

```sql
SELECT 	c.category_name,
		cu.state,
        SUM(oi.total_value) as total_value_sum,
        RANK() OVER(PARTITION BY c.category_name ORDER BY SUM(oi.total_value) ASC) 
FROM order_items oi
JOIN products p ON p.product_id = oi.product_id
JOIN category c ON p.category_id = c.category_id
JOIN orders o ON oi.order_id = o.order_id
JOIN customers cu ON o.customer_id = cu.customer_id
GROUP BY cu.state,c.category_name
ORDER BY category_name, total_value_sum asc;
```


7. Customer Lifetime Value (CLTV)
Calculate the total value of orders placed by each customer over their lifetime.
Challenge: Rank customers based on their CLTV.

```sql
SELECT 
    c.customer_id, 
    CONCAT(c.first_name, ' ',  c.last_name) as full_name, 
    SUM(oi.total_value) as CLTV,
    RANK() OVER (ORDER BY SUM(oi.total_value) DESC) as 'rank'
FROM order_items oi
JOIN orders o ON oi.order_id = o.order_id
JOIN customers c ON o.customer_id = c.customer_id
GROUP BY c.customer_id, full_name
ORDER BY CLTV desc;

```


8. Inventory Stock Alerts
Query products with stock levels below a certain threshold (e.g., less than 10 units).
Challenge: Include last restock date and warehouse information.

```sql
SELECT 
    p.product_id, 
    p.product_name, 
    i.stock, 
    i.last_stock_date, 
    i.warehouse_id
FROM inventory AS i
JOIN products AS p ON p.product_id = i.product_id
WHERE i.stock < 10;
```

9. Shipping Delays
Identify orders where the shipping date is later than 3 days after the order date.
Challenge: Include customer, order details, and delivery provider.

```sql
SELECT 
    s.order_id, 
    o.order_date, 
    s.shipping_date, 
    s.shipping_providers, 
    c.customer_id  
FROM shipping AS s
JOIN orders AS o ON o.order_id = s.order_id
JOIN customers AS c ON c.customer_id = o.customer_id
WHERE DATEDIFF(s.shipping_date, o.order_date) > 3;
```

10. Payment Success Rate 
Calculate the percentage of successful payments across all orders.
Challenge: Include breakdowns by payment status (e.g., failed, pending).

```sql
SELECT 
    payment_status,
    COUNT(*) AS payment_count,
    ROUND((COUNT(*) / (SELECT COUNT(*) FROM payments)) * 100, 2) AS percentage
FROM payments
GROUP BY payment_status;
```

11. Top Performing Sellers
Find the top 5 sellers based on total sales value.
Challenge: Include both successful and failed orders, and display their percentage of successful orders.

```sql
WITH top_sellers AS (
    SELECT s.seller_id, s.seller_name, SUM(oi.total_value) AS total_sales
    FROM order_items oi
    JOIN orders o ON oi.order_id = o.order_id
    JOIN sellers s ON o.seller_id = s.seller_id
    GROUP BY s.seller_id, s.seller_name
    ORDER BY total_sales DESC
    LIMIT 5
),
seller_order_status AS (
    SELECT 
        o.seller_id, 
        o.order_status, 
        ts.seller_name, 
        COUNT(*) AS total_orders
    FROM orders o
    JOIN top_sellers ts ON o.seller_id = ts.seller_id
    WHERE o.order_status NOT IN ('Inprogress', 'Returned')
    GROUP BY o.seller_id, o.order_status, ts.seller_name
    ORDER BY o.seller_id
),
sellers_reports AS (
    SELECT * FROM seller_order_status
)

SELECT 
    seller_id,
    seller_name,
    SUM(CASE WHEN order_status = 'Completed' THEN total_orders ELSE 0 END) AS Completed_orders,
    SUM(CASE WHEN order_status = 'Cancelled' THEN total_orders ELSE 0 END) AS Cancelled_orders,
    SUM(total_orders) AS total_orders,
    CAST(SUM(CASE WHEN order_status = 'Completed' THEN total_orders ELSE 0 END) AS DECIMAL) /
    CAST(SUM(total_orders) AS DECIMAL) * 100 AS successful_orders_percentage
FROM sellers_reports
GROUP BY seller_id, seller_name;
```


12. Product Profit Margin
Calculate the profit margin for each product (difference between price and cost of goods sold).
Challenge: Rank products by their profit margin, showing highest to lowest.
*/


```sql
SELECT 
    product_id,
    product_name,
    profit_margin,
    DENSE_RANK() OVER(ORDER BY profit_margin DESC) as product_ranking
FROM (
    SELECT 
        p.product_id,
        p.product_name,
        SUM(total_value - (p.cogs * oi.quantity)) / SUM(total_value) * 100 AS profit_margin
    FROM order_items AS oi
    JOIN products AS p ON oi.product_id = p.product_id
    GROUP BY p.product_id, p.product_name
) AS t1;
```

13. Most Returned Products
Query the top 10 products by the number of returns.
Challenge: Display the return rate as a percentage of total units sold for each product.

```sql
SELECT 
    p.product_id,
    p.product_name,
    COUNT(*) AS total_unit_sold,
    SUM(CASE WHEN o.order_status = 'Returned' THEN 1 ELSE 0 END) AS total_returned,
    (SUM(CASE WHEN o.order_status = 'Returned' THEN 1 ELSE 0 END) * 100.0) / COUNT(*) AS return_percentage
FROM order_items AS oi
JOIN products AS p ON oi.product_id = p.product_id
JOIN orders AS o ON o.order_id = oi.order_id
GROUP BY p.product_id, p.product_name
ORDER BY return_percentage DESC;
```

14. Inactive Sellers
Identify sellers who haven’t made any sales in the last 6 months.
Challenge: Show the last sale date and total sales from those sellers.

```sql
WITH cte1 AS (
    SELECT * 
    FROM sellers
    WHERE seller_id NOT IN (
        SELECT seller_id 
        FROM orders 
        WHERE order_date >= CURRENT_DATE - INTERVAL 6 MONTH
    )
)

SELECT 
    o.seller_id,
    MAX(o.order_date) AS last_sale_date,
    MAX(oi.total_value) AS last_sale_amount
FROM orders o
JOIN cte1 ON cte1.seller_id = o.seller_id
JOIN order_items oi ON o.order_id = oi.order_id
GROUP BY o.seller_id;

```


15. IDENTITY customers into returning or new
if the customer has done more than 5 return categorize them as returning otherwise new
Challenge: List customers id, name, total orders, total returns

```sql
SELECT 
    c_full_name AS customers,
    total_orders,
    total_return,
    CASE
        WHEN total_return > 5 THEN 'Returning_customers'
        ELSE 'New'
    END AS cx_category
FROM (
    SELECT 
        CONCAT(c.first_name, ' ', c.last_name) AS c_full_name,
        COUNT(o.order_id) AS total_orders,
        SUM(CASE WHEN o.order_status = 'Returned' THEN 1 ELSE 0 END) AS total_return	
    FROM orders o
    JOIN customers c ON c.customer_id = o.customer_id
    JOIN order_items oi ON oi.order_id = o.order_id
    GROUP BY c_full_name
) AS sub;
```


16. Top 5 Customers by Orders in Each State
Identify the top 5 customers with the highest number of orders for each state.
Challenge: Include the number of orders and total sales for each customer.
```sql
SELECT *
FROM (
    SELECT 
        c.state,
        CONCAT(c.first_name, ' ', c.last_name) AS customers,
        COUNT(o.order_id) AS total_orders,
        SUM(oi.total_value) AS total_sale,
        DENSE_RANK() OVER (PARTITION BY c.state ORDER BY COUNT(o.order_id) DESC) AS `rank`
    FROM orders o
    JOIN order_items oi ON oi.order_id = o.order_id
    JOIN customers c ON c.customer_id = o.customer_id
    GROUP BY c.state, c.customer_id
) AS t1
WHERE `rank` <= 5;
```

17. Revenue by Shipping Provider
Calculate the total revenue handled by each shipping provider.
Challenge: Include the total number of orders handled and the average delivery time for each provider.

```sql
SELECT 
	s.shipping_providers,
	COUNT(o.order_id) as order_handled,
	SUM(oi.total_value) as total_sale,
	COALESCE(AVG(s.return_date - s.shipping_date), 0) as average_days
FROM orders as o
JOIN 
order_items as oi
ON oi.order_id = o.order_id
JOIN 
shipping as s
ON 
s.order_id = o.order_id
GROUP BY s.shipping_providers;
```

18. Top 10 product with highest decreasing revenue ratio compare to last year(2022) and current_year(2023)
Challenge: Return product_id, product_name, category_name, 2022 revenue and 2023 revenue decrease ratio at end Round the result
Note: Decrease ratio = cr-ls/ls* 100 (cs = current_year ls=last_year)

```sql
WITH last_year_sale AS
(
    SELECT 
        p.product_id,
        p.product_name,
        SUM(oi.total_value) AS revenue
    FROM orders AS o
    JOIN order_items AS oi ON oi.order_id = o.order_id
    JOIN products AS p ON p.product_id = oi.product_id
    WHERE YEAR(o.order_date) = 2022
    GROUP BY p.product_id, p.product_name
),

current_year_sale AS
(
    SELECT 
        p.product_id,
        p.product_name,
        SUM(oi.total_value) AS revenue
    FROM orders AS o
    JOIN order_items AS oi ON oi.order_id = o.order_id
    JOIN products AS p ON p.product_id = oi.product_id
    WHERE YEAR(o.order_date) = 2023
    GROUP BY p.product_id, p.product_name
)

SELECT
    cs.product_id,
    ls.revenue AS last_year_revenue,
    cs.revenue AS current_year_revenue,
    ls.revenue - cs.revenue AS rev_diff,
    ROUND((CAST(cs.revenue AS DECIMAL) - CAST(ls.revenue AS DECIMAL)) / CAST(ls.revenue AS DECIMAL) * 100, 2) AS revenue_dec_ratio
FROM last_year_sale AS ls
JOIN current_year_sale AS cs ON ls.product_id = cs.product_id
WHERE ls.revenue > cs.revenue
ORDER BY revenue_dec_ratio DESC
LIMIT 10;
```


19. Final Task: Stored Procedure
Create a stored procedure that, when a product is sold, performs the following actions:
Inserts a new sales record into the orders and order_items tables.
Updates the inventory table to reduce the stock based on the product and quantity purchased.
The procedure should ensure that the stock is adjusted immediately after recording the sale.

```SQL
DELIMITER $$

CREATE PROCEDURE amazon_project.add_sales(
    IN p_order_id INT,
    IN p_customer_id INT,
    IN p_seller_id INT,
    IN p_order_item_id INT,
    IN p_product_id INT,
    IN p_quantity INT
)
BEGIN
    DECLARE v_count INT;
    DECLARE v_price DECIMAL(10,2);
    DECLARE v_product VARCHAR(50);

    -- Fetching product price and name based on product ID entered
    SELECT price, product_name
    INTO v_price, v_product
    FROM products
    WHERE product_id = p_product_id;

    -- Checking stock and product availability in inventory
    SELECT COUNT(*) 
    INTO v_count
    FROM inventory
    WHERE product_id = p_product_id
    AND stock >= p_quantity;

    IF v_count > 0 THEN
        -- Add into orders and order_items table, update inventory
        INSERT INTO orders(order_id, order_date, customer_id, seller_id)
        VALUES (p_order_id, CURRENT_DATE, p_customer_id, p_seller_id);

        -- Adding to order items list
        INSERT INTO order_items(order_item_id, order_id, product_id, quantity, price_per_unit, total_value)
        VALUES (p_order_item_id, p_order_id, p_product_id, p_quantity, v_price, v_price * p_quantity);

        -- Updating inventory
        UPDATE inventory
        SET stock = stock - p_quantity
        WHERE product_id = p_product_id;

        -- RAISE NOTICE equivalent in MySQL
        SELECT CONCAT('Thank you, product: ', v_product, ' sale has been added. Inventory stock updated.') AS message;

    ELSE
        -- If the product is not available in sufficient quantity
        SELECT CONCAT('Sorry, the product: ', v_product, ' is not available in sufficient quantity.') AS message;

    END IF;

END $$

DELIMITER ;
```



**Testing Store Procedure**
call add_sales
(
25005, 2, 5, 25004, 1, 14
);

---

---

## **Learning Outcomes**

This project enabled me to:
- Design and implement a normalized database schema.
- Clean and preprocess real-world datasets for analysis.
- Use advanced SQL techniques, including window functions, subqueries, and joins.
- Conduct in-depth business analysis using SQL.
- Optimize query performance and handle large datasets efficiently.

---

## **Conclusion**

This advanced SQL project successfully demonstrates my ability to solve real-world e-commerce problems using structured queries. From improving customer retention to optimizing inventory and logistics, the project provides valuable insights into operational challenges and solutions.

By completing this project, I have gained a deeper understanding of how SQL can be used to tackle complex data problems and drive business decision-making.

---

### **Entity Relationship Diagram (ERD)**
![ERD](https://github.com/najirh/amazon_usa_project5/blob/main/erd.png)

---
