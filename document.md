# Database Schema and Queries

## Tables

### User

| Column Name | Data Type | Constraints |
|-------------|------------|-------------|
| `user_id` | `uuid` | `PRIMARY KEY` |
| `name` | `varchar(255)` | `NOT NULL` |
| `email` | `varchar(255)` | `NOT NULL, UNIQUE` |
| `address` | `text` |  |
| `created_at` | `datetime` | `DEFAULT CURRENT_TIMESTAMP` |

### Product

| Column Name | Data Type | Constraints |
|-------------|------------|-------------|
| `product_id` | `uuid` | `PRIMARY KEY` |
| `name` | `varchar(255)` | `NOT NULL` |
| `description` | `text` |  |
| `price` | `decimal(10,2)` | `NOT NULL` |
| `stock_quantity` | `int` | `NOT NULL DEFAULT 0` |
| `created_at` | `datetime` | `DEFAULT CURRENT_TIMESTAMP` |

### Order

| Column Name | Data Type | Constraints |
|-------------|------------|-------------|
| `order_id` | `uuid` | `PRIMARY KEY` |
| `order_number` | `int` |  |
| `user_id` | `uuid` | `FOREIGN KEY` |
| `order_date` | `datetime` | `DEFAULT CURRENT_TIMESTAMP` |
| `total_amount` | `decimal(10,2)` | `NOT NULL` |
| `status` | `varchar(50)` |  |
| `canceled_reason` | `text` |  |

### OrderItems

| Column Name | Data Type | Constraints |
|-------------|------------|-------------|
| `order_item_id` | `uuid` | `PRIMARY KEY` |
| `order_id` | `uuid` | `FOREIGN KEY` |
| `product_id` | `uuid` | `FOREIGN KEY` |
| `quantity` | `int` | `NOT NULL` |
| `price_at_order_time` | `decimal(10,2)` | `NOT NULL` |
| `created_at` | `datetime` | `DEFAULT CURRENT_TIMESTAMP` |

### Payment

| Column Name | Data Type | Constraints |
|-------------|------------|-------------|
| `payment_id` | `uuid` | `PRIMARY KEY` |
| `order_id` | `uuid` | `FOREIGN KEY` |
| `payment_date` | `datetime` | `DEFAULT CURRENT_TIMESTAMP` |
| `payment_method` | `varchar(50)` | `NOT NULL` |
| `amount` | `decimal(10,2)` | `NOT NULL` |
| `created_at` | `datetime` | `DEFAULT CURRENT_TIMESTAMP` |

## Relationships

- **Users to Orders**: One-to-Many relation. A single user has multiple orders, but each order is for only one user.
- **Orders to OrderItems**: One-to-Many relation. A single order can contain multiple order items, but each item in the order is linked to only one order.
- **Products to OrderItems**: One-to-Many relation. A product can appear in multiple order items, but each order item is related to only one product.
- **Orders to Payments**: One-to-One relation. Each order is linked with one payment, and each payment is linked with one order.

## Indexes

### User Table

- `PRIMARY KEY`: `user_id` for faster lookup and user-related queries.
- `INDEX`: `email` column for faster lookup and user-related queries.

### Product Table

- `PRIMARY KEY`: `product_id` for faster lookup and product-related queries.
- `INDEX`: `name` column for faster lookup and search queries.
- `INDEX`: `price` column for faster lookup and price-related search queries.

### Order Table

- `PRIMARY KEY`: `order_id` for faster lookup and order-related queries.
- `INDEX`: `status` for quick filtering and sorting.
- `INDEX`: `order_date` for quick filtering and sorting.

### OrderItem Table

- `INDEX`: `order_id` and `product_id` for optimized joins and faster filtering.

### Payment Table

- `INDEX`: `order_id` and `payment_date` for faster filtering of payments.


# Handling Product Price Changes Over Time

When dealing with product price changes, it’s important to ensure that orders reflect the price at the time of purchase, rather than the current price of the product. This is handled as follows:

## Price at Order Time

- The `OrderItems` table includes a column `price_at_order` which stores the price of the product at the time the order was placed.
- When an order is placed, the current price of the product is copied into this `price_at_order` field.
- This approach ensures that even if the product price changes later, the order reflects the price that the customer agreed to pay.

# Handling Order Cancellations

Order cancellations must be handled carefully to ensure data consistency and accurate financial reporting:

## Order Status Field

- Add a `status` column to the `Orders` table that can have values such as `Pending`, `Completed`, `Cancelled`, etc.
- When an order is cancelled, update this field to `Cancelled`.

## Reverting Stock Levels

- When an order is cancelled, the system should automatically increase the stock level of the products included in that order by the quantity that was originally ordered.
- This ensures that inventory remains accurate.

## Payment Reversal

- If a payment was made before the cancellation, a record of the payment reversal or refund should be stored in the `Payments` table.
- This can be done by creating a new payment entry with a negative amount, indicating a refund.

## Audit Trail

- For better tracking and auditing, consider adding an `order_history` or `order_log` table that records all changes to the status of an order, including cancellations, along with timestamps and the user who made the change.
- This can be important for resolving disputes or performing financial audits.


## Table Creation SQL Queries

### User Table

```sql
CREATE TABLE user (
    user_id UUID PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) NOT NULL UNIQUE,
    address TEXT,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

### Product Table

```sql
CREATE TABLE product (
    product_id UUID PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    description TEXT,
    price DECIMAL(10,2) NOT NULL,
    stock_quantity INT NOT NULL DEFAULT 0,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```


### Order Table

```sql
CREATE TABLE order (
    order_id UUID PRIMARY KEY,
    order_number INT,
    user_id UUID,
    order_date DATETIME DEFAULT CURRENT_TIMESTAMP,
    total_amount DECIMAL(10,2) NOT NULL,
    status VARCHAR(50) DEFAULT 'pending',
    canceled_reason TEXT,
    FOREIGN KEY (user_id) REFERENCES user(user_id)
);
```

### OrderItem Table

```sql
CREATE TABLE orderitem (
    order_item_id UUID PRIMARY KEY,
    order_id UUID,
    product_id UUID,
    quantity INT NOT NULL,
    price_at_order_time DECIMAL(10,2) NOT NULL,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (order_id) REFERENCES order(order_id),
    FOREIGN KEY (product_id) REFERENCES product(product_id)
);

```


### Payment Table

```sql
CREATE TABLE payment (
    payment_id UUID PRIMARY KEY,
    order_id UUID,
    payment_date DATETIME DEFAULT CURRENT_TIMESTAMP,
    payment_method VARCHAR(50) NOT NULL,
    amount DECIMAL(10,2) NOT NULL,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    FOREIGN KEY (order_id) REFERENCES order(order_id)
);
```

## Queries

### 1. Retrieve the Total Number of Orders Placed by Each User

```sql
SELECT u.user_id, u.name, COUNT(o.order_id) AS total_orders
FROM user AS u
LEFT JOIN order AS o ON u.user_id = o.user_id
GROUP BY u.user_id;
```

#### Explanation: This query counts the number of orders placed by each user. The LEFT JOIN ensures that users without orders are also listed.


### 2. Retrieve the Top 5 Products that Generated the Most Revenue

```sql
SELECT p.product_id, p.name, SUM(oi.quantity * oi.price_at_order_time) AS total_revenue
FROM orderitem AS oi
JOIN product p ON oi.product_id = p.product_id
GROUP BY p.product_id
ORDER BY total_revenue DESC
LIMIT 5;
```

#### Explanation: This query calculates the total revenue generated by each product by summing the product of quantity and price_at_order_time, ordering by total revenue in descending order, and returning the top 5 results.


### 3. Retrieve the List of Users Who Have Placed Orders in the Last 30 Days

```sql
SELECT DISTINCT u.user_id, u.name, u.email
FROM user AS u
JOIN order AS o ON u.user_id = o.user_id
WHERE o.order_date >= CURDATE() - INTERVAL 30 DAY;
```

#### Explanation: This query lists distinct users who have placed orders within the last 30 days by filtering on order_date. The INNER JOIN ensures that only users with orders in the last 30 days are listed.


### 4. Retrieve the Total Revenue Generated in the Last Month

```sql
SELECT DISTINCT u.user_id, u.name, u.email
FROM user AS u
JOIN order AS o ON u.user_id = o.user_id
WHERE o.order_date >= CURDATE() - INTERVAL 30 DAY;
```

#### Explanation: This query sums the total amount of orders placed in the last month.

### 5. Retrieve the List of Orders that Were Canceled, Along with the Reason (if Available)

```sql
SELECT o.order_id, o.user_id, o.order_date, o.total_amount, o.status, o.canceled_reason
FROM order AS o
WHERE o.status = 'cancelled';
```

#### Explanation: This query lists all canceled orders along with the reason for cancellation.



# Optimization:
    

