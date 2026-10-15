# Day 6 — Database Fundamentals

## Learning goals
- Compare relational SQL databases and NoSQL approaches.
- Understand keys, relationships, indexes, normalization, and transactions.
- Choose data storage based on requirements.

## 1. Relational databases
Relational databases organize data in tables with defined columns and relationships. They are often a good fit for structured data and operations that need transactions.

Example tables for an online store:

**users**
| id | name |
|---:|---|
| 1 | Asha |

**orders**
| id | user_id | total |
|---:|---:|---:|
| 101 | 1 | 25000 |

The `orders.user_id` value connects an order to a user.

- **Primary key:** uniquely identifies a row.
- **Foreign key:** references a row in another table.
- **Index:** helps the database find rows more efficiently, usually with extra storage and write cost.

## 2. SQL and NoSQL
SQL systems such as PostgreSQL and MySQL are commonly used for relational data and transactional operations. NoSQL is a broad category that includes document, key-value, wide-column, and graph databases.

Choose based on data shape, query patterns, consistency needs, operational maturity, and scaling requirements—not because one category is automatically better.

## 3. Normalization
Normalization organizes relational data to reduce duplication and update anomalies. For example, store a user's address in a user table rather than repeating it in every order row.

Sometimes systems intentionally duplicate data (denormalization) to speed up reads. That trade-off adds work to keep copies consistent.

## 4. Indexes
An index can make filters, joins, and sorting faster when it matches the query. Indexes also consume storage and can slow inserts, updates, and deletes because the index must be maintained.

Do not add indexes blindly. Inspect query plans and measure actual workloads.

## 5. Transactions and ACID
Transactions group operations into a unit that succeeds or fails together. ACID stands for:
- **Atomicity:** all operations in the transaction happen, or none do.
- **Consistency:** constraints remain satisfied.
- **Isolation:** concurrent transactions are controlled according to the isolation level.
- **Durability:** committed data survives failures within the database's guarantees.

Example: transferring money between accounts should debit one account and credit the other in a transaction, rather than allowing only one side to succeed.

## 6. Example SQL
```sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY,
    name VARCHAR(100) NOT NULL
);

CREATE TABLE orders (
    id BIGINT PRIMARY KEY,
    user_id BIGINT NOT NULL REFERENCES users(id),
    total_cents BIGINT NOT NULL CHECK (total_cents >= 0)
);

CREATE INDEX idx_orders_user_id ON orders(user_id);

SELECT id, total_cents
FROM orders
WHERE user_id = 1;
```

The example uses cents as an integer to avoid common floating-point money issues. Production schemas should also consider generated IDs, currency, timestamps, constraints, and business rules.

## Practical exercise
Design tables for a library:
1. Members can borrow many books.
2. A book may be borrowed many times over the years.
3. Each loan has a checkout date and optional return date.
4. Identify primary and foreign keys.
5. Which query might benefit from an index?

## Suggested resources
- [PostgreSQL Documentation — Tutorial](https://www.postgresql.org/docs/current/tutorial.html)
- [GeeksforGeeks — DBMS Tutorial](https://www.geeksforgeeks.org/dbms/dbms/)

## Key takeaway
Choose a database around access patterns, consistency requirements, and data relationships. Indexes and transactions are powerful, but both require thoughtful design.
