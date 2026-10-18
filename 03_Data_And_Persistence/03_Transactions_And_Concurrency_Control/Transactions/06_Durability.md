#Database #ACID #Transactions #Durability #CoreConcept

Durability, in a nutshell, is the process of persisting the writes that clients make to the database to a non-volatile storage system. This means that anything written is stored in such a way that it survives failures.
-   If a client crashes after committing a transaction, the change will be there upon reconnecting.
-   If the database crashes after a commit, the change will be there when the database is restored.
-   If the entire machine loses power and restarts, the change will still be there.

That is what durability is. It might seem obvious that all databases provide this, but many database systems play with this property, often trading it off for better performance.

### Durability and Performance
To make things durable, you have to persist them to disk (e.g., SSD or hard drive), and disk I/O is inherently slow.
-   Some databases, like Redis, offer options to make writes faster by writing primarily to memory and then occasionally snapshotting to disk in the background.
-   Relational databases, however, traditionally must have strong durability guarantees.

**The Rule:** Changes made by a **committed** transaction must be persisted in a durable, non-volatile storage. If it's not committed, all bets are off.

---

## Durability Techniques

### 1. Write-Ahead Log (WAL)
Writing all the changes to the primary data structures (tables, indexes, b-trees) can be very expensive and slow. The writes are amplified, and the data structures can be large.

To optimize this, databases use a more compact version of the changes, often called deltas. These are written to an append-only log known as the **Write-Ahead Log (WAL)**.
-   Any changes a transaction makes are first written to this log.
-   This log write is flushed immediately to disk, which guarantees persistence.
-   This provides a durable record of the changes. Even if the main data tables haven't been updated yet, in the event of a crash, the database can replay the WAL to rebuild its state.
-   Eventually, the actual data tables and structures on disk will be updated, but the WAL provides the immediate durability guarantee.

### 2. Snapshots and Append-Only Files
-   **Asynchronous Snapshots:** Another approach is to keep all changes in memory and then asynchronously, in the background, snapshot everything to disk at once.
-   **Append-Only File (AOF):** This is another option, very similar to a WAL, used by Redis. It keeps track of the changes and writes these operations to a file.

These techniques provide a lightweight way of durably storing data. They store only the changes, which is more efficient than writing the entire data structure, and allow for state reconstruction after a crash.

---

## The Challenge: The OS Cache

A critical problem that database systems must solve is the behavior of the operating system (OS) cache.

When an application (like a database) asks the OS to write to disk, the OS often performs a "shady" optimization: it writes not to the physical disk, but to its own **in-memory cache** (RAM). It does this to batch multiple small writes together into a single, larger write for performance reasons, as this reduces total I/O.

### The Durability Violation
1.  A transaction is committed.
2.  The database asks the OS to write the WAL segments to disk.
3.  The OS writes them to its memory cache and immediately reports back to the database, "Write successful."
4.  The database, believing the data is durable, reports to the client, "Commit successful."
5.  **A power loss or system crash occurs.**
6.  The OS restarts, and the in-memory cache is lost. The WAL segments never made it to the physical disk.

The result: **The database lied.** It told the client the transaction was committed, but the data was lost. The database was **not durable**.

### The Solution: `fsync`
To fight this, databases use a system call, often `fsync`, which **bypasses the OS cache** and forces the write to go directly to the physical disk.
-   This guarantees true durability.
-   However, it is **very slow** because it negates the performance benefits of the OS cache.

This creates a fundamental trade-off: true durability vs. performance. Databases that want to guarantee durability must bypass the OS cache, which can slow down commits.

---

## Trade-offs and Options

When you commit a transaction, the guarantee of durability is absolute. As a user, you should be able to shut off the power at the exact moment the database says "commit" and expect your data to be there when you come back.

However, some databases play with this property, especially for performance. Redis, for example, provides options:
-   **Strong Durability:** Every write is flushed to disk, which is slower.
-   **Eventual Durability (a self-coined term):** Writes are flushed to disk periodically (e.g., every few seconds). This makes writes extremely fast, but there is a small window of time where a crash could lead to data loss.

This trade-off can be acceptable for certain use cases, such as logging or IoT data, where losing a few seconds of non-critical data is an acceptable price for much higher write throughput. Crashes are rare, but according to Murphy's Law, "if it never happened, it will eventually happen."

---

## Summary
-   **Durability** is the guarantee that once a transaction commits, its changes are permanently persisted and will survive system crashes or power failures.
-   Databases use techniques like **Write-Ahead Logging (WAL)** to efficiently persist changes without the overhead of writing to large data structures on every commit.
-   A major challenge is the **OS cache**, which can lie to the database about a write being complete. Databases must use system calls like **`fsync`** to bypass this cache to guarantee true durability, which comes at a performance cost.
-   Some databases offer configurable durability levels, allowing users to trade off a small risk of data loss for significantly faster write performance.