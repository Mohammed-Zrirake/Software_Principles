 #Database #PostgreSQL #Performance #QueryPlanning #Explain #CoreConcept 


The `EXPLAIN` command in PostgreSQL is used to retrieve information about what **query plan** the database will use for a given SQL statement. It provides an *estimation* of the work to be done without actually executing the query.

For this demonstration, a `grades` table with over 200 million rows is used.
-   `id`: Has an index (primary key).
-   `g` (grade): Has an index.
-   `name`: Does **not** have an index.

---
## Deconstructing the `EXPLAIN` Output

The output of `EXPLAIN` contains several key pieces of information.

### 1. The Query Plan
This is the core of the output, describing the operations the database plans to perform. A common operation is a **Sequential Scan**, which is equivalent to a full table scan in other databases. It means the database will go directly to the table (the heap) and read every row.

### 2. The `cost`
The cost is represented by two numbers separated by `..`.
-   **Startup Cost (First Number):** An estimation of how long it will take (in arbitrary units, often milliseconds) to fetch the *first page* or row.
    -   A cost of `0.00` means the database can start returning results almost immediately.
    -   This number increases if the database needs to do work *before* fetching, such as sorting (`ORDER BY`) or aggregation.
-   **Total Cost (Second Number):** An estimation of the total time it will take to complete the entire query.

### 3. The `rows`
This is the database's *estimation* of the total number of rows that will be returned by the query.
> [!tip]
> This is a valuable tool for getting an approximate count of rows without running an expensive `SELECT COUNT(*)`, which physically counts every row.

### 4. The `width`
This is the estimated average width (in bytes) of a single row in the result set. This is useful for understanding the potential network overhead of your query.

---

## Practical Examples

### Query 1: Full Table Scan
```sql
EXPLAIN SELECT * FROM grades;
````

**Output Breakdown:**

- **Plan:** Seq Scan on grades. The database has no choice but to read the entire table.
    
- **cost=0.00..289000.00**: The startup cost is zero because it can start fetching immediately. The total cost is high.
    
- **rows=200000000**: It estimates 200 million rows.
    
- **width=31**: The average row width is 31 bytes.
    

### Query 2: ORDER BY an Indexed Column

codeSQL

```
EXPLAIN SELECT * FROM grades ORDER BY g;
```

**Output Breakdown:**

- The startup cost will be slightly higher than zero (e.g., 0.43). This indicates the database is doing some initial work.
    
- Because the g column has an index, which is already sorted, the sorting work is trivial. The database can perform an Index Scan to read the rows in the desired order.
    

### Query 3: ORDER BY a Non-Indexed Column

codeSQL

```
EXPLAIN SELECT * FROM grades ORDER BY name;
```

**Output Breakdown (Read from bottom to top):**

- **Inner Plan:** Parallel Seq Scan on grades. The database must first read the entire table from the heap.
    
- **Outer Plan:** Sort. The results of the scan are then sorted.
    
- **High Startup Cost:** The startup cost for the Sort operation will be extremely high (e.g., 1000000.00). This is because the database must read and sort **all 200 million rows** in memory or on disk before it can return the first row.
    

### Query 4: Varying the width

- EXPLAIN SELECT id FROM grades; -> width=4 (an integer is 4 bytes).
    
- EXPLAIN SELECT name FROM grades; -> width=19 (an estimated average for the text field).
    
- EXPLAIN SELECT g FROM grades; -> width=8 (a double or float is 8 bytes).  
    Only selecting the columns you need reduces the amount of data transferred and processed.
    

### Query 5: Index Scan vs. Index-Only Scan

- **Index Scan:**
    
    codeSQL
    
    ```
    EXPLAIN SELECT * FROM grades WHERE id = 10;
    ```
    
    - **Plan:** Index Scan. The database uses the index to find the row's location.
        
    - It then has to perform a "heap fetch" to go to the main table and retrieve the other columns (*).
        
- **Index-Only Scan:**
    
    codeSQL
    
    ```
    EXPLAIN SELECT id FROM grades WHERE id = 10;
    ```
    
    - **Plan:** Index Only Scan. This is a faster operation.
        
    - **Explanation:** The database found the id in the index and, since that was the only column requested, it **did not need to jump to the heap**. All the required information was available in the index itself.
        

---

## Conclusion

- EXPLAIN provides an estimation of the query plan and its costs without executing the query.
    
- **Startup Cost** is a critical metric. A high startup cost indicates significant work (like sorting a large, non-indexed dataset) is being done before any rows can be returned.
    
- The plan reveals what the database is doing:
    
    - Seq Scan (Full Table Scan) is generally bad and occurs when no suitable index is available.
        
    - Index Scan is good; it uses an index to find rows but still needs to visit the heap.
        
    - Index Only Scan is the best; all required data is found in the index, avoiding a trip to the heap entirely.