#Database #StorageEngine #Indexing #CoreConcept #DataStructures

Understanding how tables and indexes are stored on disk is a fundamental concept for working with databases. These low-level details explain the cost associated with queries and are what makes a query slow or fast.

---

## Core Storage Concepts

### 1. The Table and the Row ID
A logical **table** is composed of columns (fields) and rows. While some databases work with documents, the underlying concept is the same.

Most databases do not work directly with the fields you provide them. Instead, they create their own system-maintained identifier for each row, often called a **Row ID** (or "Tuple ID" in PostgreSQL). This Row ID uniquely identifies a particular row within the database's internal storage system. In some databases, like MySQL, the primary key can serve as this pseudo-Row ID.

### 2. The Page
A table's rows are physically stored in logical units called **pages**.
-   A page is a fixed-size block of bytes in memory, which translates to a location on disk.
-   The default page size varies by database (e.g., 8 KB in PostgreSQL, 16 KB in MySQL) and is often configurable.
-   A single page can fit many rows, depending on the size of each row.

> [!important]
> The database does not read single rows from disk. Disks are not byte-addressable like RAM. A database reads data in units of pages.

**Example:**
-   If each page holds 3 rows, then a table with 1001 rows would be stored across approximately 334 pages on disk.
-   Understanding how many pages your query needs to pull is critical, as this is what makes a query slow or fast.

### 3. The I/O (Input/Output Operation)
An **I/O** is a request to read data from the disk.
-   **Minimizing I/O is the currency of a database.** The fewer I/Os you make, the faster your queries will be.
-   An I/O operation fetches one or more pages. It **cannot** read a single row.
-   When you perform an I/O to get a page, you get all the rows and all the columns within that page for free. You cannot ask for a page but only get certain rows or columns from it at the disk level. This is why `SELECT *` can be expensive; the database reads all the data for the rows from disk, and then the database engine has to process it in memory to filter out the columns you don't want.
-   An I/O does not always mean going to the physical disk. Some databases, like PostgreSQL, rely heavily on the **OS cache**. A read I/O might be served from this in-memory cache, which is much faster.

### 4. The Heap
A **heap** (or heap table) is the collection of pages that contains the actual data for a table.
-   It is the special data structure where all the row data is stored.
-   It is often unordered, which can make it expensive to query. To find a specific piece of data, the database might have to scan every single page in the heap.

### 5. Indexes
An **index** is another data structure, separate from the heap, that helps you pinpoint exactly where to go in the heap to fetch information efficiently.
-   It contains a subset of the data (the indexed column(s)) and a **pointer** back to the heap.
-   This pointer is typically the **Row ID** of the full data row.
-   An index allows you to find exactly which page in the heap to fetch, instead of scanning every page.
-   An index is also stored in pages on disk and costs I/O to read. If an index is small enough, it can fit entirely in memory, making searches very fast. If it's too large, searching the index itself can become expensive.
-   Popular data structures for indexes include **B-Trees**.

---

## How It All Fits Together: A Visual Example

Consider a heap table with 1000 rows, 3 rows per page.
-   **The Heap:** Pages on disk contain the full row data. Page 0 has Row IDs 1-3, Page 1 has Row IDs 4-6, and so on, up to Page 333.
-   **The Index:** A separate data structure (e.g., on the `employee_id` column). It contains entries like:
    -   `employee_id: 10`, `row_id: 1`, `page: 0`
    -   `employee_id: 20`, `row_id: 2`, `page: 0`
    -   `employee_id: 110`, `row_id: 11`, `page: 3`

### Query Example
Let's query for an employee with `employee_id = 10000`.

**Without an Index (Heap Scan):**
1.  The database has no choice but to scan the entire heap.
2.  **I/O 1:** Read Page 0. Check rows 1, 2, 3. No match. Discard.
3.  **I/O 2:** Read Page 1. Check rows 4, 5, 6. No match. Discard.
4.  ... continue this process for all 333 pages.
5.  **I/O 334:** Read Page 333. Find the row for employee 10000.
This is extremely expensive.

**With an Index (Index Scan):**
1.  The database first searches the index for `employee_id = 10000`. This is very fast because the index is an ordered data structure like a B-Tree. Let's say this takes one I/O.
2.  **I/O 1 (Index):** Read the relevant index page. Find the entry: `employee_id: 10000`, `row_id: 1000`, `page: 333`.
3.  **I/O 2 (Heap):** Now, the database knows exactly where to go. It performs a single, targeted I/O on the heap to fetch Page 333.
4.  It processes Page 333, finds Row ID 1000, and returns the full row data.
This is much more efficient, requiring only two I/Os instead of 334.

---

## Advanced Indexing Concepts

### Clustered Indexes
Sometimes, the heap table itself can be organized (ordered) around a single index. This is called a **clustered index** or an **Index-Organized Table (IOT)**.
-   The leaf nodes of the index B-Tree *are* the actual data pages. There is no separate heap.
-   The **primary key** of a table is usually the clustered index (unless otherwise specified).
-   MySQL (InnoDB) tables always have a clustered index. If you define a primary key, that becomes the clustered index.
-   PostgreSQL has **only secondary indexes**. All indexes in PostgreSQL point back to the Row ID (Tuple ID) in the heap.

> [!warning] A Note on Random Primary Keys
> Using a truly random value like a UUID as the primary key in a table with a clustered index can kill write performance. Because the table must be physically ordered by this random key, every `INSERT` can cause a "scattershot" of writes to random pages all over the disk, which is very expensive.

---

## Summary
-   **Table:** A logical concept of rows and columns.
-   **Row ID:** A system-maintained unique identifier for a row's physical location.
-   **Page:** A fixed-size block of data on disk. Databases read and write pages, not individual rows.
-   **I/O:** An operation to read a page from disk. The goal is to minimize I/O.
-   **Heap:** The collection of pages that contains the full, often unordered, data of a table.
-   **Index:** A separate, ordered data structure that contains a subset of data and pointers (Row IDs) to the heap, enabling fast lookups.