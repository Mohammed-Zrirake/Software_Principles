#Database #StorageEngine #RowStore #ColumnStore #OLTP #OLAP #CoreConcept

Row-oriented and column-oriented storage are two distinct styles by which databases store tables on disk. Each has pros and cons, and the choice between them depends on the specific use case.

This note will explore how each storage model works, how they perform against a set of common queries, and the advantages of each. We will analyze these concepts by scanning the table directly, without the added complexity of indexes.

---

## The Example Table and Queries

Consider an `employees` table with thousands of rows. For our analysis, we'll use a small sample. A `Row ID` is a system-maintained unique identifier for a row, distinct from a primary key like `ID`.

| Row ID | ID | FirstName | LastName | SSN | Salary | ... |
| :--- | :- | :--- | :--- | :--- | :--- | :--- |
| 1001 | 1 | John | Smith | 111 | 101k | ... |
| 1002 | 2 | Adam | Knight | 222 | 102k | ... |
| ... | ...| ... | ... | ... | ... | ... |

We will evaluate the performance of these three queries against both storage models:
1.  **Point Query (Few Columns):** `SELECT FirstName FROM employees WHERE SSN = '666';`
2.  **Point Query (All Columns):** `SELECT * FROM employees WHERE ID = 1;`
3.  **Aggregation Query (Single Column):** `SELECT SUM(Salary) FROM employees;`

---

## 1. Row-Oriented Storage (Row Store)

In a row-oriented database, tables are stored as **rows on disk**.

### How it Works
The data for a single row is stored contiguously. On disk, this looks like `ID, FirstName, LastName, SSN, ...` for the first row, immediately followed by the data for the second row, and so on.
-   Rows can have variable sizes due to variable-length data types (like `TEXT`).
-   The disk controller reads data in fixed-size **blocks** (e.g., 512 bytes, 4 KB).
-   A single I/O operation to read one block can fetch multiple rows.
-   When a block is read, you get **all columns** for all the rows in that block, whether you need them or not.

### On-Disk Representation
Imagine each gray box is a disk block that requires one I/O to read. For simplicity, each block holds two rows.
```
[BLOCK 0]
1001, 1, John, Smith, 111, 101k, ...
1002, 2, Adam, Knight, 222, 102k, ...
-------------------------------------
[BLOCK 1]
1003, 3, Melissa, Ford, 333, 103k, ...
1004, 4, Rick, Jones, 444, 104k, ...
-------------------------------------
... and so on
```

### Query Performance on a Row Store

#### Query 1: `SELECT FirstName FROM employees WHERE SSN = '666';`
To find the row with `SSN = '666'`, the database must scan the table.
1.  **I/O 1:** Read Block 0. Scan the SSN for rows 1001 and 1002. No match. Discard.
2.  **I/O 2:** Read Block 1. Scan the SSN for rows 1003 and 1004. No match. Discard.
3.  **I/O 3:** Read Block 2. Scan the SSN. Find row 1006 with `SSN = '666'`.
4.  **Result:** The `FirstName` is needed. Since the entire row was read into memory with the block, the `FirstName` is available immediately with **no extra I/O**.
-   *Observation:* Finding the row can be slow, but retrieving additional columns from that row is very cheap.

#### Query 2: `SELECT * FROM employees WHERE ID = 1;`
The database needs to find the row where `ID = 1`.
1.  **I/O 1:** Read Block 0. Scan the ID. Find row 1001 with `ID = 1`.
2.  **Result:** The user asked for `*` (all columns). Since the entire row was read into memory with the block, all columns are available immediately with **no extra I/O**.
-   *Observation:* This is a very efficient query for a row store. It's cheap because all the data for a single record is physically co-located.

#### Query 3: `SELECT SUM(Salary) FROM employees;`
The database needs to read the `Salary` from every row.
1.  **I/O 1:** Read Block 0. Extract the salaries from rows 1001 and 1002. Add them to a running total. Discard the rest of the data (ID, FirstName, etc.).
2.  **I/O 2:** Read Block 1. Extract the salaries. Add to total.
3.  ... repeat for all blocks in the table.
-   *Observation:* This is extremely inefficient. A huge amount of unnecessary data (all columns other than `Salary`) is read from disk into memory, only to be thrown away.

---

## 2. Column-Oriented Storage (Column Store / Columnar)

In a column-oriented database, tables are stored as **columns first on disk**.

### How it Works
All the values for a single column are stored contiguously, followed by all the values for the next column, and so on.
-   Each column's data is stored in its own set of blocks.
-   The `Row ID` must be stored with each column value to be able to reconstruct a full row later.
-   A single block I/O reads many values for a **single column**.
-   This makes it highly efficient to get many values for a given column.

### On-Disk Representation
Each column is stored in its own set of blocks. The `Row ID` is duplicated.
```
[ID BLOCKS]
1001: 1, 1002: 2, 1003: 3, 1004: 4
-------------------------------------
[FIRSTNAME BLOCKS]
1001: John, 1002: Adam, 1003: Melissa, 1004: Rick
-------------------------------------
[SSN BLOCKS]
1001: 111, 1002: 222, 1003: 333, 1004: 444
... and so on for each column
```

### Query Performance on a Column Store

#### Query 1: `SELECT FirstName FROM employees WHERE SSN = '666';`
1.  The database knows it only needs to scan the `SSN` column blocks.
2.  **I/O 1 (SSN):** Read first `SSN` block. No match.
3.  **I/O 2 (SSN):** Read second `SSN` block. Find `SSN = '666'` and see it corresponds to `Row ID = 1006`.
4.  **The Problem:** The `FirstName` is not here.
5.  **I/O 3 (FirstName):** The database now knows it needs the `FirstName` for `Row ID = 1006`. It performs a targeted read on the `FirstName` column blocks to find the value for that specific `Row ID`.
-   *Observation:* This query required jumping between different areas on the disk (thrashing), which can be expensive.

#### Query 2: `SELECT * FROM employees WHERE ID = 1;`
This is the worst-case scenario for a column store.
1.  **I/O 1 (ID):** Read the `ID` blocks to find `ID = 1`, which corresponds to `Row ID = 1001`.
2.  **The Problem:** The user asked for `*`.
3.  **I/O 2 (FirstName):** Jump to the `FirstName` blocks, find the value for `Row ID = 1001`.
4.  **I/O 3 (LastName):** Jump to the `LastName` blocks, find the value for `Row ID = 1001`.
5.  **I/O 4 (SSN):** Jump to the `SSN` blocks...
6.  ... repeat for every single column in the table.
-   *Observation:* This is the worst possible query for a column store. It causes a huge amount of disk thrashing. **Never do `SELECT *` on a column store.**

#### Query 3: `SELECT SUM(Salary) FROM employees;`
This is where column stores shine.
1.  The database knows it only needs the `Salary` column.
2.  **I/O 1 (Salary):** Read the first `Salary` block, getting many salary values at once.
3.  **I/O 2 (Salary):** Read the second `Salary` block...
4.  ... repeat for all `Salary` blocks.
-   *Observation:* This is extremely efficient. No unnecessary data is ever read from disk. The query only touches the exact data it needs. This is ideal for analytical queries.

---

## Pros and Cons

### Row-Oriented (Row Store)
-   **✔️ Pros:**
    -   Optimal for OLTP (Online Transactional Processing) workloads with frequent reads and writes of entire rows.
    -   Writes are fast and simple, as an entire row is written to one place.
    -   Efficient for queries that select many columns of a single row (e.g., `SELECT * WHERE ID = ?`).
-   **❌ Cons:**
    -   Inefficient for aggregation queries that only touch a few columns, as it reads many unnecessary columns from disk.
    -   Compression is less effective because adjacent data (different columns) is of different types and values.

### Column-Oriented (Column Store)
-   **✔️ Pros:**
    -   Perfect for OLAP (Online Analytical Processing) workloads (data warehouses, data lakes).
    -   Extremely efficient for aggregation queries that only read a few columns.
    -   Compresses greatly because adjacent data is of the same type and has high cardinality (many similar or duplicate values).
-   **❌ Cons:**
    -   Writes are slower because a single new row must be written to multiple column structures on disk.
    -   Very inefficient for queries that select many columns from a single row (`SELECT *`).
    -   More complex internal structure.

> [!tip] Hybrid Engines
> Most modern databases (PostgreSQL, MySQL, Oracle) default to a row-based storage engine. However, many offer the ability to choose the storage engine on a **per-table basis**. You can have a `row store` table for your transactional data and a `column store` table for your analytics data within the same database.