#Database #StorageEngine #Pages #IO #CoreConcept

Databases often use **fixed-size pages** to store data on disk. All database objects—tables, collections, rows, columns, indexes, sequences, and documents—eventually end up as bytes in a page.

This abstraction allows the storage engine to be separated from the database frontend (which is responsible for the data format and API). Moreover, it makes it easier to read, write, or cache data when everything is treated as a page.

---

## The Buffer Pool and I/O

Databases read and write data in units of pages.
1.  When you request a row from a table, the database finds the page where the row lives and identifies the file and byte offset where that page is located on disk.
2.  The database asks the OS to read from that file at the specified offset for the length of the page.
3.  The OS checks its filesystem cache. If the data isn't there, the OS issues a physical read I/O and pulls the page from disk into memory for the database to consume.

The database allocates a large pool of memory, often called the **shared pool** or **buffer pool**. Pages read from disk are placed in this buffer pool.
-   **Read Efficiency:** Once a page is in the buffer pool, the database has access not only to the requested row but also to all other rows on that page. This makes reads very efficient, especially for index range scans. The smaller the rows, the more rows fit in a single page, and the more "bang for your buck" a single I/O gives you.
-   **Write Efficiency:** When a user updates a row, the database finds the page, pulls it into the buffer pool, and updates the row *in memory*. It then makes a journal entry of the change (often to a Write-Ahead Log or WAL) which is persisted to disk. The modified ("dirty") page can remain in memory to receive more writes before it is finally flushed back to disk, minimizing the number of I/Os.

## Page Content
What is stored in a page depends on the database's storage model. The goal is always to pack items into the page such that a single page read is as effective as possible and provides as much useful information as it can for the client's workload.

-   **Row-Store Databases:** Write entire rows (with all their attributes) one after the other, packed into the page. This is optimized for OLTP workloads, especially writes.
-   **Column-Store Databases:** Write the values for a single column contiguously. A single page read will be packed with values from one column, making aggregate functions like `SUM` much more effective for OLAP workloads.
-   **Document-Based Databases:** Compress documents and store them in pages, much like row stores.
-   **Graph-Based Databases:** Persist the connectivity (relationships) in pages such that a page read is efficient for traversing the graph.

> [!tip]
> If you find yourself reading many pages to do very little work, consider rethinking your data modeling.

## Small vs. Large Pages
-   **Small Pages:** Faster to read and write, especially if the page size is close to the disk's underlying media block size. However, the overhead cost of the page header metadata compared to the useful data can get high.
-   **Large Pages:** Minimize metadata overhead and can reduce page splits, but at the cost of higher "cold read" and write latency (the first time a page is read from disk).

**Default Page Sizes:**
-   **PostgreSQL:** 8 KB
-   **MySQL (InnoDB):** 16 KB
-   **MongoDB (WiredTiger):** 32 KB
-   **SQL Server:** 8 KB
-   **Oracle:** 8 KB

Database defaults work for most cases, but it is important to know these defaults and be prepared to configure them for your specific use case.

---

## How Pages are Stored on Disk

One common way to store and retrieve pages is to make a file per table, where the file is simply an array of fixed-size pages (Page 0, followed by Page 1, etc.).

To read something from disk, you need three pieces of information: the file name, the offset, and the length. This design provides all three.
-   **File Name:** Known from the table's metadata.
-   **Offset:** Calculated as `Page_Number * Page_Size`.
-   **Length:** The `Page_Size`.

**Example:** To read pages 2 through 9 from table `test` with an 8 KB page size:
-   File: The file for table `test`.
-   Offset: `2 * 8192 = 16384` bytes.
-   Length: `8 * 8192 = 65536` bytes.

---

## PostgreSQL Page Layout

PostgreSQL's default page size is 8 KB. A page consists of several sections:
-   **Page Header (24 bytes):** Fixed-size metadata describing the page, including the amount of free space available.
-   **ItemIds (4 bytes each):** An array of item pointers. This is **not** the data itself. Each `ItemId` is a 4-byte pointer containing the offset within the page where the item (tuple) begins and its length.
    -   This array of pointers enables the **HOT (Heap-Only Tuple)** optimization. When a row is updated, if the new version (tuple) fits on the same page, only the old `ItemId` pointer is changed to point to the new tuple. Indexes and other data structures can continue to point to the old, stable `ItemId`, avoiding the need to update them.
    -   *Criticism:* This can lead to high overhead. If a page stores 1000 small items, the `ItemId` array alone would consume 4 KB, or half the page.
-   **Items (variable length):** The area where the actual item data (the tuples) lives, stored one after the other.
-   **Special (variable length):** An area used only in B+Tree index pages to store pointers to the previous and next leaf pages in the index.

> [!note] Row vs. Tuple
> -   **Row:** The logical entity a user sees.
-   **Tuple:** A physical instance of a row on a page. The same logical row can have many tuples due to MVCC (Multi-Version Concurrency Control). For example, one active tuple, several older versions for other transactions to read, and some dead tuples that are no longer needed.
-   **Item:** The tuple itself.

## Summary
Data in databases, whether it's table rows, indexes, or sequences, ultimately ends up in pages. This abstraction makes it easier for the database to work with data regardless of its content. A page has a header and a data section and is stored on disk as part of a file. Each database has a different implementation of how a page looks and how it is physically stored, but the core concept is the same.