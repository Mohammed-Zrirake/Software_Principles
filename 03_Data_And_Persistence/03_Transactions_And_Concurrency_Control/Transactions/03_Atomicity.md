#Database #ACID #Transactions #Atomicity #CoreConcept

Atomicity is one of the four ACID properties that defines a relational database management system, but it is a critical concept for any database system, including NoSQL and graph databases.

The definition of atomicity is that **all queries in a transaction must succeed.** A transaction is treated as one unit of work, like an atom that cannot be split (disregarding nuclear fission for the analogy).

---

## The "All or Nothing" Rule

If a transaction contains 100 successful queries, but even one query fails, the entire transaction must be rolled back immediately. A query can fail for many reasons:
-   A failed constraint (e.g., a balance going negative).
-   A duplicate primary key entry.
-   An invalid SQL syntax.

This "all or nothing" principle is the core of atomicity.

### What if the Database Crashes?
A crash is another form of failure. If the database goes down mid-transaction, before a `COMMIT` has been issued, all the successful queries that were part of that uncommitted transaction must be rolled back.

The database itself is responsible for this cleanup. When the database restarts, it must detect that there was a transaction in progress that did not commit and was interrupted by the crash. It then needs to roll back any changes made by that transaction.

This raises questions about how a database implements transactions.
-   Does it write to disk with every query?
-   Or does it write only to memory and flush to disk upon `COMMIT`?

There is no single right or wrong answer; there is always a trade-off.
-   **Optimistic Writing:** Some databases write changes to disk during the transaction, assuming a `COMMIT` will happen. This makes the final `COMMIT` very fast (it might just need to flip a bit to mark the transaction as committed).
-   **Deferred Writing:** Other databases write all changes to memory. Queries are fast because they are not hitting the disk. The final `COMMIT` is slower because it has to flush everything from memory to disk. However, a `ROLLBACK` in this model can be extremely fast, as it may only involve discarding the in-memory changes.

Understanding how a database handles this is key to choosing the right one for your use case.

---

## Example: A Failure of Atomicity

Let's revisit the account transfer example.
-   **Table:** `account`
-   **Fields:** `account_id`, `balance`

| account_id | balance |
| :--- | :--- |
| 1 | 1000 |
| 2 | 500 |

We want to send $100 from account 1 to account 2. This requires two updates that must happen together in a transaction.

**The Transaction Steps:**
1.  `BEGIN TRANSACTION;`
2.  `SELECT balance FROM account WHERE account_id = 1;` (Result is 1000, which is > 100, so we proceed).
3.  `UPDATE account SET balance = balance - 100 WHERE account_id = 1;`
4.  ***DATABASE CRASHES***
5.  Database restarts.

Let's assume this is a badly implemented database system. When we restart and query the table, this is what we see:

| account_id | balance |
| :--- | :--- |
| 1 | 900 |
| 2 | 500 |

**The Result: An Inconsistent State**
-   $100 has been lost into thin air.
-   The first account was debited, but the other was never credited.
-   The data is now **inconsistent**.

Lack of atomicity leads to inconsistencies. We have lost data, and the information is corrupted.

### The Cleanup Responsibility
An atomic transaction is one that will roll back all queries if one or more fail. In this situation, the database itself failed, not a query. The database is responsible for cleaning up this "garbage."

Upon restart, a properly implemented database must:
1.  Recognize that a transaction was running and was interrupted by a crash.
2.  Initiate a rollback of any partial changes that were made. In our example, it must change the balance for account 1 from `900` back to `1000`.

This rollback process can be lengthy. For long-running transactions, a rollback upon restart can take over an hour, during which the database or specific tables might be unavailable. This is a primary reason why **long-running transactions are generally a bad idea.**

---

## Summary
-   Atomicity is the idea of a transaction being **one, indivisible unit of work.**
-   All queries in a transaction must succeed. If any single query fails, all previous successful queries in that transaction must be rolled back.
-   This "all or nothing" guarantee must also hold in the case of a system crash. If a transaction was in progress and not committed, the database must roll it back upon restart to ensure a consistent state.