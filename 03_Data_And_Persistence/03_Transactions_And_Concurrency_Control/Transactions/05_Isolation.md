#Database #ACID #Transactions #Isolation #Concurrency

Isolation is a critical and often underrated property of ACID. It addresses the challenges of concurrency, where multiple transactions may be reading and writing the same pieces of data simultaneously.

The central question that isolation seeks to answer is: **Can an in-flight transaction see changes made by other transactions?** This includes changes from other transactions that are also in-flight (uncommitted) as well as changes from transactions that have already finished (committed). The answer is, "it depends," as there is no single right or wrong behavior for all use cases.

This leads to several "read phenomena," which are generally undesirable side effects of concurrency. Isolation levels were introduced to solve these phenomena.

---

## Read Phenomena

### 1. Dirty Reads
A dirty read occurs when a transaction reads data that another concurrent transaction has written, but **has not yet committed**. There is a chance that this change could be rolled back, the database could crash, or it could be committed. Reading data that is not yet finalized is considered "dirty."

**Example:**
Consider a `sales` table:

| product_id | quantity | price |
| :--- | :--- | :--- |
| 1 | 10 | 5 |
| 2 | 20 | 4 |

**Sequence of Events:**
1.  **Transaction 1** begins and runs a query to generate a report.
    ```sql
    -- Get total sales per product
    SELECT product_id, quantity * price AS total FROM sales;
    ```
    -   *Result:* Product 1: 50, Product 2: 80. The expected grand total is 130.

2.  **Transaction 2** begins concurrently and updates a row but does **not** commit.
    ```sql
    -- Update sales for product 1
    UPDATE sales SET quantity = 15 WHERE product_id = 1;
    ```
    -   The quantity for product 1 is now 15 in an uncommitted state.

3.  **Transaction 1**, still running, executes another query to get the grand total.
    ```sql
    -- Get grand total of all sales
    SELECT SUM(quantity * price) FROM sales;
    ```
    -   *Result with Dirty Read:* **155**. The query reads the uncommitted value (15 * 5 = 75) for product 1 and adds it to the value for product 2 (80), resulting in 155.
    -   This creates an inconsistent report: the details show a total of 130, but the summary shows 155.

4.  **Transaction 2** decides to roll back its change.
    ```sql
    ROLLBACK;
    ```
    -   The situation is now even worse. Transaction 1's report is based on a change that never actually happened.

### 2. Non-Repeatable Reads
A non-repeatable read occurs when a transaction reads the same row twice, but gets a different value each time. This happens because another transaction modified and **committed** that row in between the two reads.

It's not always two identical `SELECT` statements; it can be two different queries that depend on the same underlying value (e.g., a `SELECT` on a row and a `SELECT SUM()` over that row's column).

**Example:**
Using the same initial `sales` table.

**Sequence of Events:**
1.  **Transaction 1** begins and reads the sales data.
    ```sql
    SELECT product_id, quantity * price AS total FROM sales;
    ```
    -   *Result:* Product 1: 50, Product 2: 80.

2.  **Transaction 2** begins, updates a row, and **commits** the change.
    ```sql
    UPDATE sales SET quantity = 15 WHERE product_id = 1;
    COMMIT;
    ```
    -   The change is now permanent in the database.

3.  **Transaction 1**, still running, executes its second query.
    ```sql
    SELECT SUM(quantity * price) FROM sales;
    ```
    -   *Result:* **155**. Because Transaction 2 committed, Transaction 1's second query sees the new, updated value and calculates an inconsistent sum. The read of the initial data was "non-repeatable."

### 3. Phantom Reads
A phantom read occurs when a transaction runs a range query twice, and the second execution returns a set of rows that includes new rows that weren't there in the first execution. This happens because another transaction **inserted** new rows that satisfy the range query's conditions and committed.

It is distinct from a non-repeatable read because the original transaction never read the new rows in the first place, so they couldn't have "changed." They are "phantom" rows that appeared.

**Example:**
Using the same initial `sales` table.

**Sequence of Events:**
1.  **Transaction 1** begins and reads all sales data. Unbounded queries (without a `WHERE` clause) are a common source of this issue.
    ```sql
    SELECT product_id, quantity * price AS total FROM sales;
    ```
    -   *Result:* Product 1: 50, Product 2: 80. The expected total is 130.

2.  **Transaction 2** begins, **inserts** a completely new product, and commits.
    ```sql
    INSERT INTO sales (product_id, quantity, price) VALUES (3, 10, 1);
    COMMIT;
    ```
    -   A new row for product 3 now exists permanently.

3.  **Transaction 1**, still running, executes its second query.
    ```sql
    SELECT SUM(quantity * price) FROM sales;
    ```
    -   *Result:* **140**. The query now includes the newly inserted "phantom" row for product 3, leading to an inconsistent result within the same transaction.

### 4. Lost Updates
A lost update occurs when two transactions read the same data, both modify it based on the original value, and one transaction's update is overwritten (or "lost") by the other.

**Example:**
Using the same initial `sales` table where product 1 has a quantity of 10.

**Sequence of Events:**
1.  **Transaction 1** and **Transaction 2** begin concurrently. Both read the quantity of product 1, which is 10.

2.  **Transaction 1** calculates a new quantity.

    ```sql
    -- Add 10 units to product 1
    UPDATE sales SET quantity = quantity + 10 WHERE product_id = 1; -- (Calculates 10 + 10 = 20)
    ```

3.  **Transaction 2** also calculates a new quantity based on the original value it read.

    ```sql
    -- Add 5 units to product 1
    UPDATE sales SET quantity = quantity + 5 WHERE product_id = 1; -- (Calculates 10 + 5 = 15)
    ```

4.  **Transaction 2** commits first. The quantity for product 1 is now 15.

5.  **Transaction 1** then commits. It overwrites the value, setting the quantity to 20.
    -   *Result:* The final quantity is 20. The update from Transaction 2 (adding 5 units) has been **lost**. The correct final value should have been 25. This can be solved by locking the row during the update.

---

## Isolation Levels

Isolation levels were invented to prevent these read phenomena. You can set the isolation level for a transaction (e.g., `SET TRANSACTION ISOLATION LEVEL ...`).

-   **Read Uncommitted:** No isolation. A transaction can see uncommitted changes from other transactions. This allows **Dirty Reads**. It is theoretically fast but questionable. Supported by SQL Server, but few others.
-   **Read Committed:** One of the most popular levels. Each query in a transaction only sees changes that have been **committed** by other transactions. If a change is committed mid-transaction, a subsequent query in the same transaction *will* see it. This prevents dirty reads but allows non-repeatable reads and phantom reads.
-   **Repeatable Read:** This level ensures that if a query reads a row, that row will remain unchanged for the duration of the transaction. It prevents dirty reads and non-repeatable reads, but **phantom reads can still occur** in many databases.
-   **Snapshot:** A powerful level where each query in a transaction only sees changes that were committed *up to the start of the transaction*. It's like taking a snapshot of the entire database at that moment. This gets rid of all read phenomena, including phantom reads. In PostgreSQL, "Repeatable Read" is actually implemented as Snapshot Isolation.
-   **Serializable:** The highest level of isolation. It ensures that transactions execute as if they were run one after the other, serially. This eliminates all concurrency issues but is also the slowest.

### Comparison Table

| Isolation Level | Dirty Read | Non-repeatable Read | Phantom Read |
| :--- | :--- | :--- | :--- |
| **Read Uncommitted** | May Occur | May Occur | May Occur |
| **Read Committed** | Not Possible | May Occur | May Occur |
| **Repeatable Read** | Not Possible | Not Possible | May Occur |
| **Serializable** | Not Possible | Not Possible | Not Possible |

*Note: Snapshot isolation, not shown in this table, also prevents all of these phenomena.*

---

## Database Implementation of Isolation

-   **Pessimistic Concurrency Control:** This approach uses **locks** (table locks, page locks, row-level locks). When a transaction changes something, it locks it to prevent others from touching it until it's finished. This can cause other transactions to wait.
-   **Optimistic Concurrency Control:** This approach generally does not use locks. Transactions proceed, and if the database detects a conflict (two transactions stepping on each other), it will fail one of them with an error (e.g., a "serializable error"), and the application is expected to retry the failed transaction. This avoids the overhead of lock management and is preferred by many NoSQL databases.

---

## Summary

-   Isolation determines how concurrent transactions are separated from each other's effects.
-   Failures in isolation lead to read phenomena like Dirty Reads, Non-repeatable Reads, Phantom Reads, and Lost Updates.
-   Isolation Levels (`Read Committed`, `Repeatable Read`, `Serializable`, etc.) were invented to prevent these phenomena, each offering a different trade-off between consistency and performance.