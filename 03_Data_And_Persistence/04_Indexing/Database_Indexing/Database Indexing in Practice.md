 #Database #Indexing #PostgreSQL #Performance #CoreConcept #BTree
 
Understanding database indexing is a critical fundamental for any backend engineer. An index is a data structure built on top of an existing table that summarizes it to create "shortcuts," allowing the database to find data efficiently without scanning the entire table.

---
## The Concept of an Index

The best way to think of an index is like the thumb tabs (A, B, C...) on a large phone book. If you're looking for a company named "Zebra," you don't start reading from page one. You immediately go to the "Z" tab and start your search there.

Indexes are typically implemented using data structures like **B-Trees** or **LSM Trees**, which are designed for effective searching.

---
## Practical Demonstration with PostgreSQL

To test indexing, we will first create a table with a large quantity of data.

### Step 1: Setup a PostgreSQL Instance with Docker
1.  **Spin up a container:** This command creates a PostgreSQL 13 container named `pg1`.
    ```sh
    docker run --name pg1 -e POSTGRES_PASSWORD=postgres postgres:13
    ```
2.  **Access the `psql` shell:** This command lets you interact with the database.
    ```sh
    docker exec -it pg1 psql -U postgres
    ```

### Step 2: Create and Populate a Large Table
1.  **Create a table:** Let's create a simple table to hold temperature readings from IoT devices.
    ```sql
    CREATE TABLE temp (t INT);
    ```
2.  **Insert a million rows:** We can use PostgreSQL's built-in `generate_series` function combined with `random()` to quickly populate the table with a million rows of random integer data between 0 and 100.
    ```sql
    INSERT INTO temp (t)
    SELECT random() * 100
    FROM generate_series(1, 1000000);
    ```
    -   `generate_series(1, 1000000)` creates a virtual table with a million rows.
    -   `random()` generates a fractional value between 0 and 1. We multiply by 100 to get our desired range.
    -   The `INSERT ... SELECT` statement executes this for each row generated.

3.  **Verify the data:**
    ```sql
    SELECT COUNT(*) FROM temp; -- Returns 1000000
    SELECT t FROM temp LIMIT 10; -- Shows a sample of the random data
    ```

> [!tip] Populating Multiple Fields
> If you have multiple fields, you can add more expressions to the `SELECT` statement. You can also use other functions to convert numbers into characters (e.g., using ASCII values) to generate random strings.

---

## Indexing Performance: A Case Study

For this test, a table named `employees` with ~11 million rows is used.
-   `id INT PRIMARY KEY`: This column is sequential, unique, and **has a B-Tree index by default** because it is a primary key.
-   `name TEXT`: This column is just random strings and **does not have an index**.

To measure performance in PostgreSQL, we use the `EXPLAIN ANALYZE` command, which executes the query and shows the query plan and execution time.

### Querying with an Index (`id` column)

**Query 1: Selecting only the indexed column**
```sql
EXPLAIN ANALYZE SELECT id FROM employees WHERE id = 2000;
````

**Result:**

- **Plan:** Index Only Scan on the primary key index.
    
- **Heap Fetches: 0**. The database did **not** have to go to the main table (the heap) to get the id, because the id value is already stored in the index itself. This is the "sweetest query" for a database engineer.
    
- **Execution Time:** Extremely fast (e.g., ~0.1 ms).
    

**Query 2: Selecting a non-indexed column**

codeSQL

```
EXPLAIN ANALYZE SELECT name FROM employees WHERE id = 5000;
```

**Result:**

- **Plan:** Index Scan. The database first uses the index to find the location of the row.
    
- **Heap Fetches: 1**. Because the name column is not in the index, the database had to perform an additional "hop" to the main table (the heap) to retrieve the name value for that row.
    
- **Execution Time:** Significantly slower than Query 1 (e.g., ~2.5 ms). The extra I/O operation to fetch from the heap adds cost.
    

> [!note] Caching  
> Executing the same query multiple times will result in much faster execution times because the data pages are cached in memory by the database, the OS, and even the SSD controller.

### Querying Without an Index (name column)

**Query 3: Searching a non-indexed column**

codeSQL

```
EXPLAIN ANALYZE SELECT id FROM employees WHERE name = 'some_random_string';
```

**Result:**

- **Plan:** Parallel Sequential Scan. This is a **full table scan**.
    
- **Explanation:** Because the name column has no index, the database has no choice but to read **every single one of the 11 million rows** from the heap and check if the name matches. PostgreSQL is smart and uses multiple worker threads to do this in parallel, but it is still fundamentally inefficient.
    
- **Execution Time:** Extremely slow (e.g., ~3 seconds).
    

**Query 4: The LIKE operator with a leading wildcard**

codeSQL

```
EXPLAIN ANALYZE SELECT id FROM employees WHERE name LIKE '%Zz%';
```

**Result:**

- **Plan:** Parallel Sequential Scan.
    
- **Execution Time:** Extremely slow (e.g., ~1.1 seconds).
    
- **Why?** You are asking the database to evaluate an expression, not to look up a literal value. A standard B-Tree index cannot be used to satisfy this query because the leading % means the value could be anywhere. The database is forced to do a full table scan.
    

---

## After Creating an Index

Let's create an index on the name column. This will take time, as the database needs to scan all 11 million rows to build the B-Tree structure.

codeSQL

```
CREATE INDEX employees_name_idx ON employees (name);
```

**Query 5: Rerunning the search on the now-indexed column**

codeSQL

```
EXPLAIN ANALYZE SELECT id FROM employees WHERE name = 'some_random_string';
```

**Result:**

- **Plan:** Bitmap Heap Scan using a Bitmap Index Scan. The database is now using our new index.
    
- **Execution Time:** Dramatically faster (e.g., ~47 ms instead of 3000+ ms).
    

**Query 6: Rerunning the LIKE query**

codeSQL

```
EXPLAIN ANALYZE SELECT id FROM employees WHERE name LIKE '%Zz%';
```

**Result:**

- **Plan:** Parallel Sequential Scan.
    
- **Execution Time:** Still extremely slow.
    
- **Why?** The index was **not used**. The planner quickly detects that the expression with the leading wildcard cannot be satisfied by the B-Tree index and falls back to a full table scan. This is a very common mistake.
    

---

## Conclusion

- **An index is a separate data structure** that creates shortcuts to data in a table's heap, avoiding slow full table scans.
    
- **EXPLAIN ANALYZE** is a critical tool for understanding how your database is executing a query and whether it's using an index.
    
- Querying for columns that are **part of the index itself** is the fastest (**Index Only Scan**).
    
- Querying for columns that are **not in the index** requires an extra I/O hop to the heap, which is slower.
    
- **Having an index does not guarantee it will be used.** The query planner makes the final decision. Using expressions or functions (like a LIKE with a leading wildcard) on an indexed column will often prevent the index from being used.
    
- It is up to you as the engineer to write queries that can take advantage of the indexes you create. Avoid SELECT * and only ask for the data you absolutely need.