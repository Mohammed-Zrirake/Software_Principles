#Database #ACID #Transactions #Atomicity #Isolation #Consistency #Durability #PostgreSQL #Docker

This is a practical demonstration of the four ACID properties using a PostgreSQL database running in a Docker container.

---

## Setup: Spinning up a PostgreSQL Container

To try out different databases easily, Docker is an effective tool. The following commands will spin up a PostgreSQL container and provide a shell to interact with it.

**1. Start the Docker Container:**
-   `--name pg-acid`: Gives the container a memorable name.
-   `-d`: Runs the container in detached mode.
-   `-e POSTGRES_PASSWORD=postgres`: Sets the password for the default `postgres` user.
-   `postgres:13`: Specifies the image to use.
```sh
docker run --name pg-acid -e POSTGRES_PASSWORD=postgres -d postgres:13
```
After running, confirm the container is up and running.
```sh
docker ps
```

**2. Access the PostgreSQL Shell (`psql`):**
This command executes the `psql` command-line tool inside the running container, logging in as the `postgres` user.
```sh
docker exec -it pg-acid psql -U postgres
```
You are now in the `psql` shell, connected to the default `postgres` database.

---

## Table Creation

We will create two tables to simulate a simple sales system.

**1. `products` Table:**
```sql
CREATE TABLE products (
    pid SERIAL PRIMARY KEY,
    name TEXT,
    price FLOAT,
    inventory INT
);
```
> [!note] Data Types
> The `double` type is not a standard PostgreSQL type for floating-point numbers; `float` is used instead. Different databases have different names for similar types.

**2. `sales` Table:**
```sql
CREATE TABLE sales (
    sid SERIAL PRIMARY KEY,
    pid INT,
    price FLOAT,
    quantity INT
);
```
A foreign key could be created on `sales.pid`, but it is omitted here for simplicity.

---

## A. Atomicity in Practice

First, let's populate the `products` table with an initial item.
```sql
INSERT INTO products (name, price, inventory) VALUES ('phone', 999.99, 100);
```

**The Scenario:** A sale of 10 phones. This is a single logical task ("domain-driven design") that must be broken into multiple queries in the database.
1.  Deduct 10 units from the `products` inventory.
2.  Insert a new row into the `sales` table to record the sale.

These two operations must be performed in an **atomic transaction**.

### Simulating a Crash (Failure of Atomicity)
1.  **Begin a transaction.**
    ```sql
    BEGIN;
    ```
2.  **Update the inventory.**
    ```sql
    UPDATE products SET inventory = inventory - 10 WHERE pid = 1;
    ```
    -   If you query the products table now *within this same transaction*, you will see the inventory is 90. `SELECT * FROM products;` -> `inventory: 90`.

3.  **Simulate a crash.** If the database crashes at this exact moment, before the `sales` record is inserted and before the transaction is committed, you have a problem. An uncommitted change has been made.

Let's simulate this by exiting `psql` (`\q`) and stopping the container. If you were to restart a badly implemented database, you might see that the inventory is permanently 90, but no sale was recorded. Ten units would be lost.

### The Correct Behavior (With Atomicity)
Let's restart the container and reconnect.
```sh
docker start pg-acid
docker exec -it pg-acid psql -U postgres
```
Now, query the `products` table.
```sql
SELECT * FROM products;
```
-   *Result:* The inventory is back to **100**.

**Why?** Because of **Atomicity**. We started a transaction and made an update, but then crashed before committing. Upon restart, PostgreSQL detected the uncommitted, in-progress transaction and automatically rolled it back, restoring the database to its last consistent state.

### The Correct, Complete Transaction
1.  `BEGIN;`
2.  `UPDATE products SET inventory = inventory - 10 WHERE pid = 1;`
3.  `INSERT INTO sales (pid, price, quantity) VALUES (1, 999.99, 10);`
4.  `COMMIT;`

Now, both changes are made durable. A query of `products` shows inventory is 90, and a query of `sales` shows the new sale record. The sum of inventory and sold quantity is consistent. This demonstrates how atomicity is a prerequisite for **consistency**.

---

## I. Isolation in Practice

Let's test isolation with two concurrent sessions. Open a second terminal and connect to the same `pg-acid` container.

### The Problem: Lack of Isolation (Read Committed)
The default isolation level in PostgreSQL is **Read Committed**. This means a transaction can see changes made by other transactions as soon as they are committed.

**Session 1 (Report Generation):**
1.  Let's start by inserting some more sales data for our test.
2.  Begin a transaction to generate a report.
    ```sql
    BEGIN;
    ```
3.  First, get a summary of sales per product.
    ```sql
    SELECT pid, COUNT(pid) FROM sales GROUP BY pid;
    ```
    -   *Result:* Let's say it shows 4 sales for `pid` 1 and 3 sales for `pid` 2.

**Session 2 (Concurrent Sale):**
While Session 1 is running, another user makes a new sale.
```sql
BEGIN;
INSERT INTO sales (pid, price, quantity) VALUES (1, 999.99, 10);
UPDATE products SET inventory = inventory - 10 WHERE pid = 1;
COMMIT; -- The change is now permanent.
```

**Session 1 (Report Continues):**
Now, Session 1 continues its report by getting the detailed list of sales.
```sql
SELECT pid, price, quantity FROM sales;
```
-   *Result:* This query will now show **five** sales for `pid` 1. The report is now **inconsistent**. The summary says 4 sales, but the details list 5. This is an example of a **non-repeatable read**.

### The Solution: Higher Isolation Level (Repeatable Read)

We can fix this by using a higher isolation level.
**Session 1:**
1.  Roll back the previous transaction. `ROLLBACK;`
2.  Start a new transaction with a higher isolation level.
    ```sql
    BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
    ```
3.  Run the first summary query.
    ```sql
    SELECT pid, COUNT(pid) FROM sales GROUP BY pid;
    ```
    -   *Result:* 4 sales for `pid` 1, 3 for `pid` 2.

**Session 2:**
Another user makes a new sale and commits it, just like before.
```sql
BEGIN;
-- ... INSERT and UPDATE statements ...
COMMIT;
```
-   If you query from Session 2, you will see 5 sales for `pid` 1. `SELECT COUNT(*) FROM sales WHERE pid = 1;` -> `5`.

**Session 1:**
Now, run the detail query again inside the repeatable read transaction.
```sql
SELECT pid, price, quantity FROM sales;
```
-   *Result:* The query still shows only **four** sales for `pid` 1. The newly committed row is invisible to this transaction. This is because **Repeatable Read** (which is implemented as **Snapshot Isolation** in PostgreSQL) gives you a consistent snapshot of the database from the moment the transaction began. No matter how many times you execute a query, you get a repeatable read.

Once you `COMMIT` Session 1 and run the queries again, you will see the new count of 5.

---

## D. Durability in Practice

Durability is the guarantee that once a `COMMIT` is successful, the data is permanently saved.

**The Test:**
We will start a transaction, insert a new product, and issue a `COMMIT`. At the exact same moment, we will kill the container to simulate a power loss or system crash.

**Session 1:**
Prepare the `INSERT` and `COMMIT` commands.
```sql
BEGIN;
INSERT INTO products (name, price, inventory) VALUES ('TV', 3000.00, 10);
COMMIT;
```

**Another Terminal:**
Prepare the command to kill the container.
```sh
docker stop pg-acid
```

**Execution:**
Execute the `COMMIT` in Session 1 and immediately execute the `docker stop` command.

**Verification:**
1.  Restart the container. `docker start pg-acid`
2.  Reconnect to the database. `docker exec -it pg-acid psql -U postgres`
3.  Query the products table.
    ```sql
    SELECT * FROM products;
    ```
-   *Result:* The new **TV** product is present in the table.

This is **Durability**. Because the `psql` shell confirmed that the `COMMIT` was successful before the connection was lost, PostgreSQL guarantees that the data was written to a durable, non-volatile location (disk) and survived the crash. If the crash had happened before the `COMMIT` returned, atomicity would have ensured the transaction was rolled back upon restart.