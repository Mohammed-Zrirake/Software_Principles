#Database #PostgreSQL #Indexing #Performance #BestPractice #CoreConcept

 The optimal indexing strategy depends entirely on your query patterns. **Separate indexes** are flexible but less efficient for `AND` queries. A **composite index** is extremely fast for queries filtering on its leading columns but can be inflexible. A **hybrid approach** offers the most coverage at the cost of storage and write overhead.

---

This note explores three common indexing strategies against four common query types to understand their performance characteristics.

### The Setup
-   **Table:** `test` (`a INT`, `b INT`, `c INT`) with ~12 million rows.
-   **Goal:** Analyze how different indexes on columns `a` and `b` affect query performance.

### The Four Test Queries
1.  **Filter on `A`:** `SELECT c FROM test WHERE a = 70;`
2.  **Filter on `B`:** `SELECT c FROM test WHERE b = 100;`
3.  **Filter on `A AND B`:** `SELECT c FROM test WHERE a = 100 AND b = 100;`
4.  **Filter on `A OR B`:** `SELECT c FROM test WHERE a = 70 OR b = 100;`

> [!info] A Note on `Bitmap Heap Scan`
> When a query is expected to return a large number of rows, PostgreSQL may choose a `Bitmap Heap Scan`. It first builds an in-memory bitmap of all matching page locations from the index, and then visits the table heap in an orderly fashion to fetch the rows. If only a few rows are needed (e.g., with a `LIMIT` clause), it may opt for a more direct `Index Scan`.

---

## 🏛️ Scenario 1: Two Separate Indexes

We create one index on column `a` and a separate index on column `b`.

```sql
CREATE INDEX ON test (a);
CREATE INDEX ON test (b);
```

### Query 1: `WHERE a = 70`
-   **Plan:** `Bitmap Heap Scan` using the index on `a`.
-   **Performance:** ✅ **Good**. The database correctly uses the index on `a` to find the matching rows.

### Query 2: `WHERE b = 100`
-   **Plan:** `Bitmap Heap Scan` using the index on `b`.
-   **Performance:** ✅ **Good**. The database correctly uses the index on `b`.

### Query 3: `WHERE a = 100 AND b = 100`
-   **Plan:** `BitmapAnd`. PostgreSQL scans *both* indexes in parallel, builds a bitmap for each, and then performs a very fast `AND` operation on the bitmaps to find the rows that match both conditions.
-   **Performance:** ✅ **Very Good**. This is an efficient way to handle an `AND` condition with separate indexes.

### Query 4: `WHERE a = 70 OR b = 100`
-   **Plan:** `BitmapOr`. Similar to the `AND` case, it scans both indexes and combines the results with a `BitmapOr` operation.
-   **Performance:** ✅ **Good**. This is an effective strategy for `OR` queries.

---

## 🏛️ Scenario 2: Single Composite Index

We drop the separate indexes and create a single composite (or multi-column) index on `(a, b)`.

```sql
CREATE INDEX ON test (a, b);
```

### Query 1: `WHERE a = 70`
-   **Plan:** `Bitmap Heap Scan` using the composite index `(a, b)`.
-   **Performance:** ✅ **Good**. Because `a` is the **leading (left-most) column** in the index, PostgreSQL can efficiently use the index to satisfy this query.

### Query 2: `WHERE b = 100`
-   **Plan:** `Parallel Sequential Scan`. **The index is NOT used.**
-   **Performance:** ❌ **Terrible**. This is the critical weakness of composite indexes. You **cannot** use the index to filter on a secondary column (`b`) without also providing a filter for the leading column (`a`).

> [!danger] The Composite Index Pitfall
> A composite index on `(a, b)` can be used for queries on `a` or on `a AND b`. It **cannot** be used for queries that only filter on `b`. The order matters!

### Query 3: `WHERE a = 100 AND b = 100`
-   **Plan:** `Index Scan`.
-   **Performance:** 🚀 **Excellent**. This is the absolute best-case scenario and the primary reason to use a composite index. The database can use the single index to pinpoint the exact rows with extreme efficiency. The performance is "absolutely insane."

### Query 4: `WHERE a = 70 OR b = 100`
-   **Plan:** `Parallel Sequential Scan`. **The index is NOT used.**
-   **Performance:** ❌ **Terrible**. The composite index is not useful for `OR` conditions, as the database cannot efficiently seek to all possible combinations. It decides a full table scan is faster.

---

## 🏛️ Scenario 3: The Hybrid Approach

We keep the composite index on `(a, b)` and add back a separate index just for `b`.

```sql
-- We already have the composite index on (a, b)
CREATE INDEX ON test (b);
```

### Query 1: `WHERE a = 70`
-   **Performance:** ✅ **Good**. No change. Uses the composite index.

### Query 2: `WHERE b = 100`
-   **Plan:** `Bitmap Heap Scan` using the new, dedicated index on `b`.
-   **Performance:** ✅ **Good**. We have solved the major weakness of the composite-only approach. Queries on `b` are now fast again.

### Query 3: `WHERE a = 100 AND b = 100`
-   **Performance:** 🚀 **Excellent**. No change. Continues to use the highly efficient composite index.

### Query 4: `WHERE a = 70 OR b = 100`
-   **Plan:** `BitmapOr`.
-   **Performance:** ✅ **Good**. This is a major improvement over the composite-only scenario. Now that PostgreSQL has a viable index for each side of the `OR` condition (the composite index for `a`, and the separate index for `b`), it can revert to the efficient `BitmapOr` strategy.

---

## 📊 Summary & Best Practices

| Query Type | Separate Indexes (A), (B) | Composite Index (A, B) | Hybrid (A, B) + (B) |
|---|---|---|---|
| `WHERE a = ...` | ✅ Good | ✅ Good | ✅ Good |
| `WHERE b = ...` | ✅ Good | ❌ **Terrible** (Seq Scan) | ✅ **Good** (Uses B index) |
| `WHERE a AND b` | ✅ Very Good (BitmapAnd) | 🚀 **Excellent** (Ideal case) | 🚀 **Excellent** |
| `WHERE a OR b` | ✅ Good (BitmapOr) | ❌ **Terrible** (Seq Scan) | ✅ **Good** (BitmapOr) |

> [!tip] As a backend engineer, your choice of index is a trade-off.
> - If your queries are almost always `WHERE a AND b`, a **composite index** is the clear winner for performance.
> - If you have a mix of queries on `a` alone, `b` alone, and `a AND b`, the **hybrid approach** offers the most flexibility and consistently good performance.
> - **Separate indexes** are a good, flexible default but may be slightly less performant for `AND` queries compared to a perfectly matched composite index.
>
> Remember that every index you add incurs a cost in storage space and slower write performance (`INSERT`, `UPDATE`). Analyze your application's query patterns and index accordingly. This is a critical skill for building scalable database applications.