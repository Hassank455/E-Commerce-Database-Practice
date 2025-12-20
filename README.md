# E-Commerce Database Practice

This repository contains a simple e-commerce database schema used for practicing
SQL, ER modeling, and reporting queries.

The main entities are:

- **Category**: Stores product categories.
- **Product**: Stores products and their prices/stock.
- **Customer**: Stores customer accounts.
- **Order**: Stores customer orders (header).
- **Order_details**: Stores products inside each order (order lines).

---

## 1. Objectives

The goal of this mini-project is to:

1. Design the **database schema** using SQL (DDL).
2. Identify and implement the **relationships** between entities using primary keys and foreign keys.
3. Draw an **ERD diagram** for the schema.
4. Write several **reporting queries**:
   - Daily revenue report
   - Monthly top-selling products
   - High-value customers
5. Discuss a possible **denormalization** mechanism for Customer & Order entities.

---

## 2. Database Schema (DDL)

Below is the SQL script used to create the schema and define all entity relationships:

```sql
CREATE TABLE category(
    category_id INT AUTO_INCREMENT PRIMARY KEY,
    category_name VARCHAR(80) NOT NULL
);

---

CREATE TABLE customer (
    customer_id INT AUTO_INCREMENT PRIMARY KEY,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    email VARCHAR(100) NOT NULL UNIQUE,
    password VARCHAR(255) NOT NULL
);

---

CREATE TABLE `orders` (
    order_id INT AUTO_INCREMENT PRIMARY KEY,
    customer_id INT NOT NULL,
    order_date DATE NOT NULL,
    total_amount DECIMAL(10,2),
    CONSTRAINT FK_order_customer
        FOREIGN KEY (customer_id)
        REFERENCES customer(customer_id)
        ON DELETE CASCADE
        ON UPDATE CASCADE
);

---

CREATE TABLE product (
    product_id INT AUTO_INCREMENT PRIMARY KEY,
    category_id INT NOT NULL,
    name VARCHAR(50) NOT NULL,
    description VARCHAR(200) NOT NULL,
    price DECIMAL(10,2) NOT NULL CHECK (price > 0),
    stock_quantity INT NOT NULL CHECK (stock_quantity >= 0),
    CONSTRAINT FK_product_category
        FOREIGN KEY (category_id)
        REFERENCES category(category_id)
        ON DELETE CASCADE
        ON UPDATE CASCADE
);

---

CREATE TABLE order_details (
    order_detail_id INT AUTO_INCREMENT PRIMARY KEY,
    order_id INT NOT NULL,
    product_id INT NOT NULL,
    quantity INT NOT NULL CHECK (quantity > 0),
    unit_price DECIMAL(10,2) NOT NULL CHECK (unit_price >= 0),
    CONSTRAINT FK_order_details_order
        FOREIGN KEY (order_id)
        REFERENCES orders(order_id)
        ON DELETE CASCADE
        ON UPDATE CASCADE,
    CONSTRAINT FK_order_details_product
        FOREIGN KEY (product_id)
        REFERENCES product(product_id)
        ON DELETE CASCADE
        ON UPDATE CASCADE
);
```

---

## 3. Relationships Between Entities

Below are the main relationships in this schema:

1. **Customer → Order**

   - Relationship: **One-to-Many (1:N)**
   - Explanation:
     - One customer can place many orders.
     - Each order belongs to exactly one customer.
   - Implementation: `orders.customer_id` → `customer.customer_id`

2. **Order → Order_details**

   - Relationship: **One-to-Many (1:N)**
   - Explanation:
     - One order can contain many order lines (order details).
     - Each order_detail row belongs to exactly one order.
   - Implementation: `order_details.order_id` → `orders.order_id`

3. **Product → Order_details**

   - Relationship: **One-to-Many (1:N)**
   - Explanation:
     - One product can appear in many different order details (across different orders).
     - Each order_detail row refers to exactly one product.
   - Implementation: `order_details.product_id` → `product.product_id`

4. **Category → Product**

   - Relationship: **One-to-Many (1:N)**
   - Explanation:
     - One category can have many products.
     - Each product belongs to exactly one category.
   - Implementation: `product.category_id` → `category.category_id`

5. **Order ↔ Product (via Order_details)**
   - Logical Relationship: **Many-to-Many (M:N)**
   - Explanation:
     - One order can contain many products.
     - One product can appear in many orders.
     - This M:N relationship is implemented using the **Order_details** bridge table.

---

## 4. ERD Diagram

![E-Commerce ERD](docs/ecommerce_erd.jpg)

---

## 5. Reporting Queries

This section contains example SQL reporting queries based on the schema.

### 5.1. Daily Revenue Report

**Goal:** Get the total revenue for a specific date.

```sql
SELECT
    order_date,
    SUM(total_amount) AS daily_revenue
FROM orders
WHERE order_date = '2025-01-05';
```

### **5.2. Monthly Top-Selling Products**

There are **two types** of “top-selling” metrics used in e-commerce analytics:

### 🟡 **A. Top-Selling Products by Quantity (Units Sold)**

**Meaning:**  
Products ranked by the **number of items sold**, regardless of price.

**Example:**  
A **$10 T-shirt** sold **300 times**  
is ranked higher than a **$100 item** sold **20 times**.

```sql
SELECT
    p.name AS top_selling_product,
    SUM(od.quantity) AS total_quantity
FROM order_details od
JOIN product p
    ON p.product_id = od.product_id
JOIN orders o
    ON o.order_id = od.order_id
WHERE o.order_date >= '2025-01-01'
  AND o.order_date <  '2025-02-01'
GROUP BY p.product_id, p.name
ORDER BY total_quantity DESC
LIMIT 5;
```

### 🟡 **B. Top-Selling Products by Revenue (Money Earned)**

**Meaning:**  
Products ranked by **total revenue generated** (money earned).<br>
This is often the **true business definition** of “top-selling”.

**Example:**  
An **iPhone ($1000)** sold **5 units**<br>
earns more than a **$10 T-shirt** sold **100 units**.

```sql
SELECT
    p.name AS top_selling_product,
    SUM(od.quantity * od.unit_price) AS total_revenue
FROM order_details od
JOIN product p
    ON p.product_id = od.product_id
JOIN orders o
    ON o.order_id = od.order_id
WHERE o.order_date >= '2025-01-01'
  AND o.order_date <  '2025-02-01'
GROUP BY p.product_id, p.name
ORDER BY total_revenue DESC
LIMIT 5;
```

### 5.3. High-Value Customers (Complex Query)

**Goal:** Retrieve customers whose total order amount exceeds $500 within a given period, including their names and their total spending amount.

```sql
SELECT
    c.first_name,
    c.last_name,
    SUM(o.total_amount) AS total_spent
FROM orders o
JOIN customer c
    ON o.customer_id = c.customer_id
WHERE o.order_date >= '2025-01-01'
  AND o.order_date <  '2025-02-01'
GROUP BY c.customer_id, c.first_name, c.last_name
HAVING total_spent > 500;
```

### **5.4. Search for Products Containing a Keyword**

**Goal:** Find all products with the word `"camera"` in the name or description.

```sql
SELECT
    name,
    description
FROM product
WHERE name LIKE '%camera%'
   OR description LIKE '%camera%';
```

### **5.5. Recommend Popular Products in the Same Category and Same Author**

**Goal:**  
Suggest popular products related to an already purchased product based on:

- Same category
- Same author
- Excluding the purchased product
- Ranked by popularity (units sold)

```sql
SELECT
    p2.product_id,
    p2.name,
    p2.category_id,
    p2.author_id,
    COALESCE(SUM(od2.quantity), 0) AS total_sold
FROM product p1
JOIN product p2
  ON p2.category_id = p1.category_id
 AND p2.author_id   = p1.author_id
LEFT JOIN order_details od2
  ON od2.product_id = p2.product_id
WHERE p1.product_id = :purchased_product_id
  AND p2.product_id <> p1.product_id
GROUP BY
    p2.product_id,
    p2.name,
    p2.category_id,
    p2.author_id
ORDER BY total_sold DESC;
```

---

## 6. Denormalization Discussion

In some cases, reporting queries that require customer information may become expensive due to frequent JOIN operations between the `orders` and `customer` tables. To optimize read performance, a denormalization approach can be applied by storing redundant customer data—such as `first_name` and `last_name`—directly inside the `orders` table.

**Benefits:**

- Faster reporting and analytics queries.
- Reduced reliance on JOIN operations.
- Improved read performance in systems with heavy reporting workloads.

**Drawbacks:**

- Data redundancy: customer names are duplicated across many orders.
- Update anomalies: if the customer updates their name, all related orders must be updated.
- Slightly increased storage usage.

This trade-off is common in analytical systems where read performance is more important than write efficiency.

---

## 7. Sale History Trigger (Automatic Sales Logging)

To support analytics and reporting use cases, a **sale history mechanism** was implemented using a database trigger.  
This trigger automatically records sales data whenever a new order line is inserted.

### 7.1. Purpose

The goal of this trigger is to:

- Capture **sales transactions automatically**
- Store a **denormalized snapshot** of sales data
- Improve performance for reporting and analytics queries
- Avoid expensive joins between `orders`, `order_details`, and `customer`

Each row inserted into `order_details` represents a real sale line, making it the ideal trigger point.

---

### 7.2. Sale History Table

The `sale_history` table stores a snapshot of each sold product at the time of purchase.

```sql
CREATE TABLE sale_history (
    sale_id INT AUTO_INCREMENT PRIMARY KEY,
    order_id INT NOT NULL,
    order_date DATE NOT NULL,
    customer_id INT NOT NULL,
    product_id INT NOT NULL,
    quantity INT NOT NULL,
    unit_price DECIMAL(10,2) NOT NULL,
    line_total DECIMAL(10,2) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

---

### 7.3. Trigger: Create Sale History on Order Line Insert

The following trigger is executed **after inserting a new row into order_details.** <br>
It combines data from the newly inserted order line and its related order header.

```sql
-- Change delimiter to allow multiple SQL statements in trigger
DELIMITER $$
-- Trigger: Automatically create a sale history record
-- Fired after inserting a new order line (order_details)
CREATE TRIGGER trg_create_sale_history
AFTER INSERT ON order_details
-- FOR EACH ROW = trigger executes once per inserted row
FOR EACH ROW
BEGIN
    INSERT INTO sale_history (
        order_id,
        order_date,
        customer_id,
        product_id,
        quantity,
        unit_price,
        line_total
    )
    -- NEW = refers to the inserted order_details row
    -- Using SELECT allows mixing data from orders + order_details
    SELECT
        o.order_id,
        o.order_date,
        o.customer_id,
        NEW.product_id,
        NEW.quantity,
        NEW.unit_price,
        NEW.quantity * NEW.unit_price
    FROM orders o
    WHERE o.order_id = NEW.order_id;
END;

-- Restore default delimiter
DELIMITER ;
```

---

### 7.4. Trigger Execution Example

```sql
INSERT INTO order_details (order_id, product_id, quantity, unit_price)
VALUES (1, 5, 2, 150);
```

This automatically generates the following record in sale_history:

| sale_id | order_id | order_date | customer_id | product_id | quantity | unit_price | line_total | created_at          |
| ------: | -------: | ---------- | ----------: | ---------: | -------: | ---------: | ---------: | ------------------- |
|       1 |        1 | 2025-01-05 |           1 |          5 |        2 |        150 |        300 | 2025-12-20 10:18:29 |

---

### 7.5. Design Notes

- The trigger runs **AFTER INSERT** to ensure all NEW.\* values exist.
- `order_details` is used instead of orders because it contains actual sales data.
- `sale_history` is intentionally denormalized for fast reporting.
- One `order_details` row = one real sales transaction.
- This approach is commonly used in analytical and reporting systems.

---

### Lock Quantity Field for a Product

Although the requirement is to lock a single field (`stock_quantity`),  
MySQL applies locks at the **row level**, not the column level.

The following example demonstrates how this is achieved using transactions.

##

#### Transaction A (Lock Holder)

```sql
START TRANSACTION;

-- Locks the row that contains product_id = 211
-- Prevents other transactions from updating stock_quantity
SELECT stock_quantity
FROM product
WHERE product_id = 211
FOR UPDATE;
```

The transaction remains open and holds the lock on the row.

##

#### Transaction B (Blocked Transaction)

```sql
UPDATE product
SET stock_quantity = stock_quantity + 1
WHERE product_id = 211;
```

This update will be blocked until Transaction A commits or rolls back.

##

#### Release the Lock

```sql
COMMIT;
```

Once the transaction is committed, the lock is released and
Transaction B continues execution.

**Note:**
Even though the intention is to lock a single column,
InnoDB enforces locking at the row level, not the column level.

---

### Lock an Entire Product Row

In this task, the goal is to explicitly lock the entire row of a product
to prevent any updates or deletions by other transactions.

##

#### Transaction A (Row Lock Holder)

```sql
START TRANSACTION;

-- Locks the entire row for product_id = 211
SELECT *
FROM product
WHERE product_id = 211
FOR UPDATE;
```

The transaction holds an exclusive lock on the selected row.

##

#### Transaction B (Blocked Transaction)

```sql
UPDATE product
SET stock_quantity = 0
WHERE product_id = 211;
```

This statement will be blocked until the lock is released.

##

#### Release the Lock

```sql
COMMIT;
```

After committing, the row lock is released and other transactions may proceed.

---

### Full-Text Search on Products

This section demonstrates how **Full-Text Search** can be used to search text-based
columns more efficiently than traditional `LIKE` queries.

A FULLTEXT index is required on the searchable columns before running these queries.

##

#### Create Full-Text Index

```sql
CREATE FULLTEXT INDEX idx_product_text_search
ON product (name, description);
```

##

#### Query 1: Basic Full-Text Search

This query retrieves all products that contain the keyword `"camera"`
in either the product name or description using Full-Text Search.

```sql
SELECT *
FROM product
WHERE MATCH(name, description)
AGAINST ('camera');
```

**Explanation:**
- Uses the FULLTEXT index to search text efficiently
- Returns matching rows only (true / false match)
- Does not rank results by relevance
- Result order is not guaranteed

##

#### Query 2: Full-Text Search with Relevance Ranking

This query improves the basic full-text search by calculating a **relevance score**
and ordering the results from most relevant to least relevant.

```sql
SELECT
    product_id,
    name,
    description,

    -- Calculates how relevant each row is to the search keyword
    -- Higher score means a better match
    MATCH(name, description) AGAINST ('camera') AS relevance_score

FROM product

-- Filters rows that match the full-text search condition
WHERE MATCH(name, description)
AGAINST ('camera')

-- Orders results so the most relevant products appear first
ORDER BY relevance_score DESC;
```

**Explanation:**
- `MATCH(...) AGAINST(...)` computes a relevance score for each row
- The score is based on:
  - Term frequency
  - Term location (name vs description)
  - Overall text distribution
- Results are sorted by relevance, similar to a search engine
- Suitable for search bars and product discovery features