# SQL & Databases — From SELECT to System Design

> **Target**: Complete SQL beginner.
> **Goal**: Write any SQL query confidently, understand database internals, avoid production pitfalls.

---

# PART 1: SQL BASICS

## SELECT, WHERE, ORDER BY, LIMIT

### The Core Statement

```sql
-- Every query starts with SELECT
SELECT * FROM users;
-- Returns ALL columns, ALL rows from users table

SELECT name, email FROM users;
-- Returns ONLY name and email columns

SELECT * FROM users WHERE age > 18;
-- Filters: only users older than 18

SELECT * FROM users ORDER BY created_at DESC;
-- Sort by creation date, newest first

SELECT * FROM users LIMIT 10;
-- Only first 10 rows

-- Combine them:
SELECT name, email 
FROM users 
WHERE age > 18 AND country = 'India' 
ORDER BY name ASC 
LIMIT 20;
```

### WHERE Operators

```sql
-- Comparison
WHERE age = 25
WHERE age > 18
WHERE age BETWEEN 18 AND 30

-- Text matching
WHERE name LIKE 'A%'      -- starts with A
WHERE name LIKE '%son'    -- ends with "son"
WHERE email LIKE '%@gmail.com'

-- Set membership
WHERE country IN ('India', 'USA', 'UK')

-- Null checking
WHERE email IS NULL       -- no email
WHERE email IS NOT NULL   -- has email
```

### Analogy

Think of SQL as asking questions about a spreadsheet:
- `SELECT` = what columns to show
- `FROM` = which spreadsheet
- `WHERE` = filter rows
- `ORDER BY` = sort
- `LIMIT` = only show first N

### Why This Matters
Every backend developer writes SQL daily. ROAST uses SQLite. SYNAPSE uses PostgreSQL. SuperOwl uses Firestore. The concepts transfer.

---

## INNER JOIN

### What It Does

Returns rows where there's a MATCH in BOTH tables.

```sql
-- Users table
-- id | name
-- 1  | Alice
-- 2  | Bob
-- 3  | Charlie

-- Orders table
-- id | user_id | product
-- 1  | 1       | Laptop
-- 2  | 1       | Mouse
-- 3  | 3       | Keyboard

SELECT users.name, orders.product
FROM users
INNER JOIN orders ON users.id = orders.user_id;

-- Result:
-- Alice | Laptop
-- Alice | Mouse
-- Charlie | Keyboard
-- Bob doesn't appear (no orders)
```

### Venn Diagram

```
  Users          Orders
┌────────┐   ┌────────┐
│        │   │        │
│  Bob   │   │        │
│        │   │        │
│ Alice ─│──│─ 1, 2  │  ← INNER JOIN = intersection
│        │   │        │
│Charlie─│──│─ 3     │
│        │   │        │
└────────┘   └────────┘
```

### Why This Matters
Most business queries involve multiple tables. Orders + customers, posts + authors, calls + businesses. Joins are how you connect related data.

---

## LEFT JOIN

### What It Does

Returns ALL rows from the LEFT table, plus matching rows from the right table (NULLs where no match).

```sql
SELECT users.name, orders.product
FROM users
LEFT JOIN orders ON users.id = orders.user_id;

-- Result:
-- Alice   | Laptop
-- Alice   | Mouse
-- Bob     | NULL     ← Bob has no orders, but still appears!
-- Charlie | Keyboard
```

### LEFT JOIN vs INNER JOIN

```sql
-- INNER JOIN: only users WITH orders
SELECT COUNT(*) FROM users u
INNER JOIN orders o ON u.id = o.user_id;
-- Result: 2 users (Alice, Charlie)

-- LEFT JOIN: ALL users
SELECT COUNT(*) FROM users u
LEFT JOIN orders o ON u.id = o.user_id;
-- Result: 3 users (Alice, Bob, Charlie)

-- LEFT JOIN to find users with NO orders:
SELECT u.name FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE o.id IS NULL;
-- Result: Bob
```

### Why This Matters
LEFT JOIN is the most common join in production. You almost always want ALL users/orders/customers even if they don't have matching data. ROAST uses LEFT JOINs extensively.

---

## RIGHT JOIN

```sql
-- Opposite of LEFT JOIN: ALL rows from RIGHT table
SELECT users.name, orders.product
FROM users
RIGHT JOIN orders ON users.id = orders.user_id;
```

**Rarely used**. Just swap the table order and use LEFT JOIN instead.

---

## FULL JOIN (FULL OUTER JOIN)

### What It Does

ALL rows from BOTH tables. NULLs where no match.

```sql
-- MySQL doesn't support FULL JOIN natively
-- PostgreSQL does:
SELECT users.name, orders.product
FROM users
FULL JOIN orders ON users.id = orders.user_id;

-- Result:
-- Alice   | Laptop
-- Alice   | Mouse
-- Bob     | NULL
-- Charlie | Keyboard
-- NULL    | ??? order with non-existent user (orphan)
```

### Why This Matters
FULL JOIN catches orphaned data — records that reference non-existent parents. Good for data quality checks.

---

## CROSS JOIN

```sql
-- Every row from table A × every row from table B
SELECT users.name, products.name
FROM users
CROSS JOIN products;

-- If 3 users × 5 products = 15 rows
-- Every combination

-- Same as:
SELECT users.name, products.name
FROM users, products;  -- implicit cross join
```

### When to Use

- Generating combinations (all possible schedules)
- Test data generation
- Building lookup tables

---

## Self JOIN

### Joining a Table to Itself

```sql
-- Employees table:
-- id | name   | manager_id
-- 1  | Alice  | NULL       ← CEO (no manager)
-- 2  | Bob    | 1          ← reports to Alice
-- 3  | Charlie| 1          ← reports to Alice
-- 4  | Diana  | 2          ← reports to Bob

SELECT 
    e.name AS employee,
    m.name AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id;

-- Result:
-- Alice   | NULL       ← Alice is CEO
-- Bob     | Alice
-- Charlie | Alice
-- Diana   | Bob
```

### Why This Matters
Self joins handle hierarchical data: org charts, categories with subcategories, threaded comments, family trees.

---

# PART 2: AGGREGATION

## GROUP BY

### What It Does

Groups rows that share a value and runs aggregate functions on each group.

```sql
-- Orders table:
-- id | product   | amount | user_id
-- 1  | Laptop    | 1000   | 1
-- 2  | Mouse     | 50     | 1
-- 3  | Keyboard  | 100    | 3
-- 4  | Monitor   | 300    | 1

SELECT user_id, COUNT(*) as order_count, SUM(amount) as total_spent
FROM orders
GROUP BY user_id;

-- Result:
-- user_id | order_count | total_spent
-- 1       | 3           | 1350
-- 3       | 1           | 100
```

### GROUP BY + JOIN

```sql
SELECT 
    u.name,
    COUNT(o.id) as order_count,
    SUM(o.amount) as total_spent
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
GROUP BY u.id, u.name;  -- group by ALL non-aggregated columns
```

---

## HAVING

### WHERE vs HAVING

- **WHERE**: filters ROWS before GROUP BY
- **HAVING**: filters GROUPS after GROUP BY

```sql
-- WRONG: WHERE can't use aggregate functions
SELECT user_id, COUNT(*) as cnt
FROM orders
WHERE COUNT(*) > 1        -- ERROR! Can't use aggregate in WHERE
GROUP BY user_id;

-- CORRECT: Use HAVING
SELECT user_id, COUNT(*) as cnt
FROM orders
GROUP BY user_id
HAVING COUNT(*) > 1;     -- only users with multiple orders

-- WHERE + HAVING together:
SELECT user_id, COUNT(*) as cnt
FROM orders
WHERE amount > 100        -- only consider orders over $100
GROUP BY user_id
HAVING COUNT(*) >= 2;     -- users with 2+ such orders
```

---

## COUNT, SUM, AVG, MAX, MIN

```sql
-- Sales table
-- id | product | amount | quantity
-- 1  | Laptop  | 1000   | 2
-- 2  | Mouse   | 50     | 5
-- 3  | Cable   | 20     | 10

SELECT 
    COUNT(*) as total_orders,       -- 3
    COUNT(DISTINCT product) as products, -- 3 different products
    SUM(amount) as revenue,          -- 1070
    AVG(amount) as avg_order_value,  -- 356.67
    MAX(amount) as highest_order,    -- 1000
    MIN(amount) as lowest_order,     -- 20
    SUM(amount * quantity) as total  -- 2450
FROM sales;
```

---

# PART 3: ADVANCED

## Subqueries

### IN Subquery

```sql
-- Find users who have placed orders
SELECT name FROM users
WHERE id IN (
    SELECT DISTINCT user_id FROM orders
);

-- Same as INNER JOIN, but sometimes clearer
```

### EXISTS Subquery (More Efficient)

```sql
-- EXISTS stops scanning as soon as it finds a match
SELECT name FROM users u
WHERE EXISTS (
    SELECT 1 FROM orders o 
    WHERE o.user_id = u.id
);

-- More efficient than IN for large datasets
```

### Correlated Subquery

```sql
-- Find employees who earn more than the average in their department
SELECT name, salary, department_id
FROM employees e
WHERE salary > (
    SELECT AVG(salary)
    FROM employees
    WHERE department_id = e.department_id  -- references OUTER query
);
```

### Performance Warning

Subqueries in WHERE can be slow. Use JOINs or EXISTS where possible.

---

## Normalization — 1NF, 2NF, 3NF

### The Problem (Unnormalized Table)

```sql
-- One table storing everything:
-- customer_id | customer_name | orders
-- 1           | Alice         | "Laptop,1000;Mouse,50"
-- 2           | Bob           | "Keyboard,100"

-- Problems:
-- 1. "orders" column has multiple values (violates atomicity)
-- 2. Can't easily query "how many laptops sold"
-- 3. Updating an order requires parsing a text field
```

### First Normal Form (1NF)

**Rule**: Each column must contain ATOMIC values (one value per cell).

```sql
-- Fix: separate orders into their own table
-- Customers:
-- id | name
-- 1  | Alice

-- Orders:
-- id | customer_id | product | amount
-- 1  | 1           | Laptop  | 1000
-- 2  | 1           | Mouse   | 50
```

### Second Normal Form (2NF)

**Rule**: Must be in 1NF + no partial dependencies (every non-key column depends on ALL of the primary key, not just part of it).

```sql
-- Violation: composite primary key (order_id, product_id)
-- order_id | product_id | product_name | quantity
-- 1        | 1          | Laptop       | 2
-- 1        | 2          | Mouse        | 5

-- Problem: product_name depends on product_id, NOT on order_id
-- If we change "Laptop" to "Laptop Pro", we must update MULTIPLE rows

-- Fix: separate products into their own table
-- Products:
-- product_id | product_name
-- 1          | Laptop
-- 2          | Mouse

-- Order_Items:
-- order_id | product_id | quantity
-- 1        | 1          | 2
-- 1        | 2          | 5
```

### Third Normal Form (3NF)

**Rule**: Must be in 2NF + no transitive dependencies (non-key columns shouldn't depend on OTHER non-key columns).

```sql
-- Violation:
-- order_id | customer_id | customer_phone
-- 1        | 1           | 555-0100

-- Problem: customer_phone depends on customer_id, not order_id
-- If Alice changes phone, must update ALL her orders

-- Fix:
-- Orders: order_id, customer_id (customer_phone removed)
-- Customers: customer_id, name, phone
```

### Why This Matters
Real databases normalize to 3NF by default. Denormalization (intentionally breaking rules) is done for PERFORMANCE reasons when you know what you're doing.

---

## Views

### A View is a Saved Query

```sql
-- Create a virtual table
CREATE VIEW active_users AS
SELECT u.id, u.name, COUNT(o.id) as order_count
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE u.status = 'active'
GROUP BY u.id, u.name;

-- Use the view like a regular table
SELECT * FROM active_users WHERE order_count > 5;

-- Much simpler than writing the full join every time
```

### Benefits

- **Security**: show only specific columns (hide sensitive data)
- **Convenience**: complex queries simplified
- **Consistency**: same definition everywhere
- **Abstraction**: underlying tables can change without breaking views

---

## Stored Procedures

### SQL Code Stored in the Database

```sql
CREATE PROCEDURE transfer_funds(
    from_account INT,
    to_account INT,
    amount DECIMAL
)
BEGIN
    START TRANSACTION;
    
    UPDATE accounts SET balance = balance - amount WHERE id = from_account;
    UPDATE accounts SET balance = balance + amount WHERE id = to_account;
    
    INSERT INTO transfers (from_id, to_id, amount) 
    VALUES (from_account, to_account, amount);
    
    COMMIT;
END;
```

### Pros vs Cons

| Pros | Cons |
|------|------|
| Fast (pre-compiled on DB) | Hard to version control |
| Reduced network traffic (one call) | Tied to specific DB (MySQL vs PG) |
| Security (no direct table access) | Hard to test |
| Encapsulated logic | Logic split between app + DB |

### Why This Matters
ROAST uses SQLite functions (not stored procs). SYNAPSE uses application-level logic. Modern backends minimize stored procedures in favor of application code — easier to test, version, and deploy.

---

## Indexes

### What They Do

Indexes make reads FASTER and writes SLOWER.

**Without index**: scan EVERY row to find what you need (sequential scan)
**With index**: jump directly to the right rows (like a book's index)

### When to Index

```sql
-- THESE columns should be indexed:
-- 1. Primary keys (automatic)
-- 2. Foreign keys (for JOINs)
-- 3. WHERE clause columns
-- 4. ORDER BY columns
-- 5. UNIQUE columns

CREATE INDEX idx_orders_user_id ON orders(user_id);   -- for JOIN
CREATE INDEX idx_users_email ON users(email);          -- for WHERE
CREATE INDEX idx_orders_date ON orders(created_at);    -- for ORDER BY

-- Composite index (multiple columns)
CREATE INDEX idx_orders_user_date ON orders(user_id, created_at);
-- This helps: WHERE user_id = 123 ORDER BY created_at
-- Does NOT help: WHERE created_at > '2024-01-01' (user_id must be first)
```

### The Tradeoff

```sql
-- Each index:
-- ✓ Speeds up SELECT (reads)
-- ✓ Speeds up UPDATE/WHERE (if indexed column is in WHERE)
-- ✗ Slows down INSERT (must update every index)
-- ✗ Slows down UPDATE/column change (if indexed column changes)
-- ✗ Takes disk space

-- Rule of thumb:
-- Tables with heavy reads → more indexes
-- Tables with heavy writes → fewer indexes
-- Typical table: 3-5 indexes
```

### EXPLAIN ANALYZE

```sql
-- See how PostgreSQL executes your query
EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 123;

-- With index:
-- Index Scan using idx_orders_user_id (...)
-- This is FAST

-- Without index:
-- Seq Scan on orders (...)
-- This is SLOW (reads every row)
```

### Why This Matters
ROAST uses SQLite indexes on market_intel.db. Missing indexes are the #1 cause of slow queries in production. SoundHound expects engineers to know when to index.

---

## ACID Properties

### The Four Properties

| Property | Meaning | Analogy |
|----------|---------|---------|
| **Atomicity** | All or nothing | Transfer money: debit + credit. If one fails, both revert |
| **Consistency** | Data stays valid | Balance can't go negative. Constraints always hold |
| **Isolation** | Concurrent transactions don't interfere | Two people can transfer simultaneously without corruption |
| **Durability** | Committed data survives crashes | After COMMIT, data is safe even if power dies |

### Why Atomicity Matters

```sql
BEGIN TRANSACTION;

-- Deduct from sender
UPDATE accounts SET balance = balance - 100 WHERE id = 1;

-- POWER FAILURE HERE! (before crediting receiver)

-- Add to receiver (NEVER RUNS)
UPDATE accounts SET balance = balance + 100 WHERE id = 2;

COMMIT;
-- WITHOUT atomicity: money DISAPPEARS. $100 debited, never credited.
-- WITH atomicity: the entire transaction REVERTS. No money lost.
```

---

## Transactions

### Python Example

```python
import sqlite3

def transfer_funds(from_id, to_id, amount):
    conn = sqlite3.connect('bank.db')
    try:
        conn.execute("BEGIN TRANSACTION")
        
        # Check balance
        balance = conn.execute(
            "SELECT balance FROM accounts WHERE id = ?", 
            (from_id,)
        ).fetchone()[0]
        
        if balance < amount:
            raise ValueError("Insufficient funds")
        
        conn.execute(
            "UPDATE accounts SET balance = balance - ? WHERE id = ?",
            (amount, from_id)
        )
        conn.execute(
            "UPDATE accounts SET balance = balance + ? WHERE id = ?",
            (amount, to_id)
        )
        
        conn.commit()  # Both updates succeed together
        print("Transfer successful")
        
    except Exception as e:
        conn.rollback()  # Both updates revert
        print(f"Transfer failed: {e}")
    finally:
        conn.close()
```

### Why This Matters
SuperOwl's billing system uses transactions (deduct credits + create call log = atomic). Without transactions, you'd lose credits or double-charge.

---

## N+1 Query Problem

### The Problem

```python
# N+1 query — SLOW!

# 1 query to get all posts
posts = db.execute("SELECT * FROM posts").fetchall()

# N queries to get author for each post
for post in posts:
    author = db.execute(  # ← runs ONCE per post!
        "SELECT * FROM users WHERE id = ?", 
        (post["author_id"],)
    ).fetchone()
    print(f"{post['title']} by {author['name']}")

# If 100 posts: 1 + 100 = 101 queries total
```

### The Fix: JOIN (Eager Loading)

```python
# 1 query, JOIN does it all
rows = db.execute("""
    SELECT posts.title, users.name as author_name
    FROM posts
    JOIN users ON posts.author_id = users.id
""").fetchall()

# 1 query instead of 101!
```

### ORMs Make This Worse

```python
# Django ORM — N+1 trap!
posts = Post.objects.all()      # 1 query: SELECT * FROM posts

for post in posts:
    print(post.author.name)      # 1 query per post! N+1!
    
# Fix with select_related:
posts = Post.objects.select_related('author').all()  # 1 query with JOIN
```

### Why This Matters
This is THE most common performance bug in production. I've seen APIs that took 10 seconds because of N+1 queries. ROAST avoids it by loading all needed data upfront.

---

## Quick Reference

```sql
-- Your SQL Toolkit

-- Read
SELECT cols FROM table WHERE condition ORDER BY col LIMIT n;

-- Join
SELECT a.col, b.col 
FROM a 
LEFT JOIN b ON a.id = b.a_id;

-- Aggregate
SELECT group_col, COUNT(*), SUM(col), AVG(col)
FROM table
GROUP BY group_col
HAVING COUNT(*) > threshold;

-- Insert
INSERT INTO table (col1, col2) VALUES (val1, val2);

-- Update
UPDATE table SET col = val WHERE condition;

-- Delete
DELETE FROM table WHERE condition;

-- Transaction
BEGIN;
  -- queries
COMMIT;  -- or ROLLBACK;

-- Index
CREATE INDEX idx_name ON table(column);

-- View
CREATE VIEW name AS SELECT ... ;
```
