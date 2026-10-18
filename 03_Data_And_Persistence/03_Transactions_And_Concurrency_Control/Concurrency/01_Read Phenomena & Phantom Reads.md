#Database #ACID #Transactions #Isolation #Concurrency #ReadPhenomena #PhantomReads

Phantom Reads are a read phenomenon in databases that can lead to disastrous results if misused or overlooked. They are one of three main phenomena that occur in concurrent transactions, along with dirty reads and non-repeatable reads.

A Phantom Read occurs when a transaction executes a query, and then a second, concurrent transaction **inserts a new row** that would satisfy the criteria of the first transaction's query. If the first transaction then re-executes its query or runs a different query that covers that new row, it will suddenly see this "phantom" row appear, leading to inconsistent results within its own isolated unit of work.

---

## Example of a Phantom Read

Let's use a `sales` table to demonstrate this.

**Initial State:**
Running `SELECT * FROM sales;` shows five records.

**The Scenario:** A transaction is started to produce a report. The default isolation level (in PostgreSQL, `Read Committed`) is in effect.

**Transaction 1 (Report Generation):**
1.  Begin the transaction.
    ```sql
    BEGIN;
    ```
2.  Run a summary query.
    ```sql
    SELECT pid, SUM(price) FROM sales GROUP BY pid;
    ```
    -   *Result:* Let's say this shows that product 1 has total sales of $40. The report is being generated based on this initial state.

**Transaction 2 (Concurrent Sale):**
While Transaction 1 is still in progress, another transaction starts, inserts a new sale, and commits immediately.
```sql
-- This runs in its own implicit transaction and commits
INSERT INTO sales (pid, price, date) VALUES (1, 15.00, '2021-02-07');
```

**Transaction 1 (Report Continues):**
The first transaction now continues generating its report and runs another query.
```sql
SELECT pid, SUM(price) FROM sales GROUP BY pid;
```
-   *Result:* The sum for product 1 has now changed. The report is now based on inconsistent data; a summary at the top of the report would not match the details or a final sum at the bottom.
-   If we run `SELECT * FROM sales;` within Transaction 1, we will see the new, sixth row that was inserted by Transaction 2. This new row is the **phantom read**. This is almost always an undesirable result.

---

## How to Fix Phantom Reads

The solution is to use a higher transaction isolation level.

### 1. `Serializable` Isolation Level
The `Serializable` isolation level is designed to prevent all read phenomena, including phantom reads. It ensures that transactions execute as if they were run one after the other.

**Demonstration:**
1.  Roll back the previous transaction: `ROLLBACK;`
2.  Start a new transaction with `Serializable` isolation.
    ```sql
    BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
    ```
3.  Run a query.
    ```sql
    SELECT * FROM sales;
    ```
    -   *Result:* Shows the six records that now exist.

4.  In a **separate session**, insert another new row and let it commit.
    ```sql
    INSERT INTO sales (pid, price, date) VALUES (1, 15.00, '2021-02-07');
    ```

5.  Go back to the **first session** (the `Serializable` transaction) and run the query again.
    ```sql
    SELECT * FROM sales;
    ```
    -   *Result:* You still see only **six** records. The newly inserted row is invisible. The database has detected the dependency and has isolated your transaction from that change. You can now safely complete your report with a consistent view of the data.

### 2. A Special Case: PostgreSQL and `Repeatable Read`
In most databases (Oracle, MySQL, SQL Server), the `Repeatable Read` isolation level prevents non-repeatable reads but does **not** prevent phantom reads. The only way to get rid of phantom reads in those systems is to use `Serializable` isolation or pessimistic locking.

PostgreSQL is special. Its implementation of `Repeatable Read` is actually **Snapshot Isolation**. This is a powerful feature.
-   Because of its Multi-Version Concurrency Control (MVCC) architecture, when you start a `Repeatable Read` transaction in PostgreSQL, you get a "version" or snapshot of the data.
-   Any inserts committed by other transactions after your snapshot was taken are invisible to you, even if they fall within your query's range.

**Demonstration in PostgreSQL:**
1.  `BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;`
2.  `SELECT * FROM sales;` (Shows 7 rows).
3.  In a separate session, insert a new row and commit.
4.  Back in the `Repeatable Read` transaction, run `SELECT * FROM sales;` again.
5.  *Result:* You still see only **7** rows. The phantom read is prevented.

> [!important]
> This behavior is specific to PostgreSQL. In other major SQL databases, `Repeatable Read` is susceptible to phantom reads.