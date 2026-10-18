#Database #ACID #Transactions #Isolation #Concurrency #Serializable #RepeatableRead

There is very little information available explaining the practical difference between `Serializable` and `Repeatable Read` isolation levels. This note aims to clarify the distinction and explain why one might be chosen over the other.

---

## The Scenario: The Write Skew Anomaly

Let's consider a scenario that `Repeatable Read` will allow but `Serializable` will prevent. This anomaly is often called a "write skew."

**Initial State:**
A table named `test` with a single text field `t`. It contains four rows with the values: 'a', 'a', 'b', 'b'.

**The Concurrent Transactions:**
-   **Transaction 1:** Intends to change all 'a' values to 'b'.
    ```sql
    UPDATE test SET t = 'b' WHERE t = 'a';
    ```
-   **Transaction 2:** Intends to change all 'b' values to 'a'.
    ```sql
    UPDATE test SET t = 'a' WHERE t = 'b';
    ```

### The `Repeatable Read` Behavior
If both transactions start at the same time under `Repeatable Read` isolation, they do not step on each other's toes directly.
-   Transaction 1 reads and locks the two 'a' rows.
-   Transaction 2 reads and locks the two 'b' rows.
The sets of rows they are modifying do not overlap.

**Sequence of Events in Practice (PostgreSQL):**
1.  **Session 1:** Begin a `Repeatable Read` transaction.
    ```sql
    BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
    ```
2.  **Session 2:** Begin a `Repeatable Read` transaction.
    ```sql
    BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
    ```
3.  **Session 1:** Execute the update.
    ```sql
    UPDATE test SET t = 'b' WHERE t = 'a'; -- Updates 2 rows
    ```
    -   *Result within Session 1:* If you `SELECT * FROM test;`, you will see all 'b's.

4.  **Session 2:** Execute the update.
    ```sql
    UPDATE test SET t = 'a' WHERE t = 'b'; -- Updates 2 rows
    ```
    -   *Result within Session 2:* If you `SELECT * FROM test;`, you will see all 'a's.

5.  **Session 1:** Commit. `COMMIT;`
6.  **Session 2:** Commit. `COMMIT;`
7.  **Final Result:** Querying the table from a new session (`SELECT * FROM test;`) will show the initial state of 'a', 'a', 'b', 'b' has returned. The updates effectively cancelled each other out because the rows they changed did not overlap. This might be the desired behavior, but often it is not. The final state is not one that could have been achieved if the transactions were run one after the other (serially).

### The `Serializable` Behavior
The `Serializable` isolation level is designed to produce a result that is equivalent to some serial (one after the other) execution of the transactions. It detects dependencies and prevents anomalies like write skew.

**Sequence of Events in Practice (PostgreSQL):**
1.  **Session 1:** Begin a `Serializable` transaction.
    ```sql
    BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
    ```
2.  **Session 2:** Begin a `Serializable` transaction.
    ```sql
    BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
    ```
3.  **Session 1:** Execute the update.
    ```sql
    UPDATE test SET t = 'b' WHERE t = 'a'; -- Updates 2 rows
    ```

4.  **Session 2:** Execute the update.
    ```sql
    UPDATE test SET t = 'a' WHERE t = 'b'; -- Updates 2 rows
    ```

5.  **Session 1:** Commit.
    ```sql
    COMMIT;
    ```
    -   The commit is successful. The state of the database is now all 'b's.

6.  **Session 2:** Attempt to commit.
    ```sql
    COMMIT;
    ```
    -   **Result: FAILURE.** PostgreSQL detects a conflict and fails the transaction.
        ```
        ERROR: could not serialize access due to read/write dependencies among transactions
        DETAIL: Reason code: Canceled on identification as a pivot, during commit attempt.
        HINT: The transaction might succeed if retried.
        ```

**Why did it fail?**
PostgreSQL did more work. It detected that Transaction 2 was reading rows that Transaction 1 was writing, and that Transaction 1 was reading rows that Transaction 2 was writing, creating a dependency cycle. To guarantee serializability, it could not allow both transactions to commit. It failed the second one.

The final state of the database is all 'b's, which is a state that could be achieved if Transaction 1 ran and then Transaction 2 ran against the new state. If Transaction 2 were retried, it would change all the 'b's to 'a's, resulting in a final state of all 'a's—another valid serial execution.

---

## Implications and Alternatives

-   **Retry Logic:** When using `Repeatable Read` or `Serializable` isolation levels, you must be prepared as an engineer to **retry transactions** in case of these serialization failures.
-   **Pessimistic Locking:** Another way to achieve this is to use pessimistic locking (e.g., `SELECT ... FOR UPDATE`). A transaction would lock the rows it intends to change, forcing other transactions to wait. This typically requires a lower isolation level like `Read Committed` to work as expected, which can introduce complexity.

---

## Conclusion

-   **`Repeatable Read`** prevents a transaction from seeing changes to rows it has already read, but it can allow anomalies like write skew where transactions modify non-overlapping sets of rows based on a shared, initial state.
-   **`Serializable`** is the strictest level. It detects and prevents these more complex anomalies by ensuring the final result is equivalent to some serial execution of the transactions, often by failing one of the conflicting transactions.
-   If you truly need to guarantee that your concurrent transactions produce a result that is equivalent to them running one after another, you must use the `Serializable` isolation level and build retry logic into your application.