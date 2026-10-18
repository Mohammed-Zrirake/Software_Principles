
 #Database #PostgreSQL #Indexing #QueryPlanning #CoreConcept #Performance 
 
This note explains three fundamental data retrieval strategies that PostgreSQL uses: Sequential Scan, Index Scan, and the clever Bitmap Scan. Understanding these is key to interpreting `EXPLAIN` plans and optimizing queries.

For this demonstration, a `grades` table with ~50 million rows is used.
-   `id`: Has a primary key index.
-   `g` (grade): Has a standard B-Tree index.
-   `name`: Does **not** have an index.

---

## 1. Sequential Scan (Full Table Scan)

A **Sequential Scan** is when PostgreSQL reads the entire table (the heap) from disk, page by page, from beginning to end, to find the rows that match the query.

### When it's Used
PostgreSQL will choose a sequential scan when:
-   There is no index on the columns in the `WHERE` clause.
-   The query is expected to retrieve a large percentage of the table's rows. In this case, the planner determines that it's cheaper to just read the whole table sequentially than to perform many random-access I/O operations by jumping back and forth between an index and the heap.

### Example: A High-Selectivity Query
This query asks for almost all rows in the table.
```sql
EXPLAIN SELECT name FROM grades WHERE id > 100;
````

**The Planner's Logic:**

1. The query is on id > 100.
    
2. The id column has an index.
    
3. However, the table has 50 million rows. This query will return almost all of them.
    
4. Using an index would mean: find a row in the index, jump to the heap, get the name; find the next row in the index, jump to the heap... performing millions of random-access "jumps."
    
5. **Decision:** It's far cheaper to just read the entire table sequentially from disk. The planner chooses Seq Scan.
    

---

## 2. Index Scan

An **Index Scan** is the "classic" index operation. The database traverses the index (a B-Tree) to find entries matching the query, and for each entry, it retrieves the corresponding row from the heap.

### When it's Used

PostgreSQL will choose an index scan when the query is highly selective—that is, when it is expected to return a **small number of rows**.

### Example: A Low-Selectivity Query

This query asks for a single row.

codeSQL

```
EXPLAIN SELECT name FROM grades WHERE id = 1000;
```

**The Planner's Logic & Execution:**

1. The query is id = 1000.
    
2. The planner sees there is an index on id.
    
3. It estimates that only one row will be returned.
    
4. **Decision:** An Index Scan is the most efficient plan.
    
5. **Execution:**
    
    - It traverses the B-Tree index to find the entry for id = 1000.
        
    - This index entry contains a pointer (the Tuple ID) to the row's physical location in the heap.
        
    - It performs a single **random-access** I/O to jump to that location in the heap and retrieve the full row to get the name column.
        

This process of "find in index -> jump to heap" is repeated for every row that matches. If a query like WHERE id < 100 returns 99 rows, it will perform 99 of these random-access jumps. This can become slow if many rows are returned.

---

## 3. Bitmap Scan

A **Bitmap Scan** is a clever, two-phase strategy that PostgreSQL uses as a compromise between a full Sequential Scan and a potentially expensive Index Scan with many random-access jumps. It is chosen when a query is expected to return a **medium number of rows**—too many for an efficient Index Scan, but not enough to justify a full Sequential Scan.

### How it Works

**Phase 1: Build the Bitmap (Index-Side)**

1. The database performs an **Bitmap Index Scan**. It scans the index (or indexes) to find all the rows that match the query conditions.
    
2. Crucially, instead of jumping to the heap for each row it finds, it creates a **bitmap** in memory. A bitmap is an array of bits where each bit corresponds to a heap **page**.
    
3. For every matching row found in the index, it identifies which heap page that row lives on and flips the corresponding bit in the bitmap to 1.
    

At the end of this phase, the database has a compact map of only the heap pages that contain at least one matching row.

**Phase 2: Fetch from the Heap (Heap-Side)**

1. The database performs a **Bitmap Heap Scan**. It looks at the generated bitmap.
    
2. It then visits **only the heap pages** that were marked with a 1 in the bitmap, reading them sequentially from disk.
    
3. **Recheck Condition:** Because a page may contain both matching and non-matching rows, PostgreSQL must **recheck** the WHERE clause condition for every row on the fetched pages to filter out the ones that don't actually match.
    

### Example: A Medium-Selectivity Query

codeSQL

```
EXPLAIN SELECT name FROM grades WHERE g > 95;
```

**The Planner's Logic:**

1. The query is g > 95. The planner estimates this will return a significant number of rows, but not the whole table.
    
2. Doing an Index Scan would require thousands of random jumps to the heap, which is inefficient.
    
3. Doing a Seq Scan would read the entire 50-million-row table, which is also inefficient.
    
4. **Decision:** A Bitmap Scan is the best compromise. It uses the index to identify the relevant pages and then reads those pages from the heap in a more sequential manner.
    

### The Power of Bitmaps: Combining Multiple Indexes

The true genius of the bitmap scan is its ability to combine multiple indexes.

codeSQL

```
EXPLAIN SELECT name FROM grades WHERE g > 95 AND id < 10000;
```

**Execution:**

1. A Bitmap Index Scan is performed on the g index, producing **Bitmap 1**.
    
2. A separate Bitmap Index Scan is performed on the id index, producing **Bitmap 2**.
    
3. The database then performs a bitwise **BitmapAnd** operation on these two bitmaps. The resulting bitmap will only have 1s for pages that contain rows satisfying **both** conditions.
    
4. The final, smaller bitmap is used for the Bitmap Heap Scan, dramatically reducing the number of heap pages that need to be visited.
    

---

## Summary

| Scan Type           | When It's Used                                                          | How It Works                                                                                                  | Key Characteristic                                                         |
| ------------------- | ----------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------- |
| **Sequential Scan** | Query is not selective (returns many rows) OR no suitable index exists. | Reads the entire table from disk, page by page.                                                               | Full table scan.                                                           |
| **Index Scan**      | Query is very selective (returns few rows).                             | Finds entry in index, then performs a random-access jump to the heap for each row.                            | Many random-access I/Os for many rows.                                     |
| **Bitmap Scan**     | Query is moderately selective.                                          | **Phase 1:** Scans index(es) to build a map of relevant heap pages. **Phase 2:** Scans only those heap pages. | The perfect compromise; turns many random I/Os into a more sequential one. |
