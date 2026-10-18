#Database #ACID #Transactions #Consistency #CoreConcept

Consistency is one of the four ACID properties. It is a property that is often traded off in different database platforms (e.g., NoSQL vs. relational) in favor of speed, performance, and scalability.

Consistency can be understood in two distinct ways: consistency in the data itself and consistency in the reads.

---

## 1. Consistency in Data

This form of consistency relates to the state of the data that is currently persisted on disk within a given database instance. It answers the question: Is the data consistent with the data model and the rules you have defined?

This type of consistency is defined by the user (e.g., the DBA or developer who builds the data model) and is enforced through several mechanisms:

-   **Referential Integrity and Foreign Keys:** This is a primary way to enforce consistency. If you have a document or row that refers to another, there must be integrity in that reference.
-   **Atomicity:** As discussed in the atomicity section, a lack of atomicity leads to data inconsistency. If a transaction debits one account and then crashes before crediting another, the persisted data becomes corrupt.
-   **Isolation:** The level of isolation can also affect data consistency. If a transaction reads data, and then another transaction changes it, a subsequent read within the first transaction can see a different, inconsistent view of the data.

### Example of Data Inconsistency
Consider a simplified Instagram data model with two tables:

-   `pictures` (id, picture_blob, likes_count)
-   `likes` (user_id, picture_id)

The consistency rule, defined by the user, is that `pictures.likes_count` must be equal to the count of rows in the `likes` table for that `picture_id`.

**Inconsistencies in the Data Model:**
-   **Incorrect Count:** Picture 1 has `likes_count = 5`, but a query of the `likes` table shows only two likes for picture 1. The data is out of sync.
-   **Dangling Reference:** The `likes` table shows that a user liked picture 4, but picture 4 does not exist in the `pictures` table. This is an orphaned record, another form of referential integrity violation.

These are examples of inconsistency in the persisted data itself. This is the type of consistency being referred to in the context of ACID transactions.

---

## 2. Consistency in Reads

This form of consistency applies to the system as a whole, especially in distributed systems with multiple database instances, shards, or partitions. It answers the question: When a user reads data, is that read consistent across the system?

The core principle is that if a transaction commits a change, a new transaction that starts *after* the commit **must** see that change. If an immediate read after a committed write returns the old value, the system is providing an inconsistent read.

### Example of Read Inconsistency
This often occurs in systems with replication, such as a primary (master) database with multiple replica (worker) databases.
1.  A `WRITE` operation is sent to the primary database, updating a value `X`.
2.  The primary database commits the change.
3.  The primary then begins to asynchronously sync this change to its replicas.
4.  If a `READ` operation is sent to one of the replicas *before* the sync has completed, the replica will return the old, stale value of `X`. This is a read inconsistency.

Both relational and NoSQL databases can suffer from this kind of inconsistency. Even NoSQL systems suffer from the first type of inconsistency (in persisted data) if referential integrity between documents or records is not enforced at the application level.

### Eventual Consistency
The term **eventual consistency** is a marketing term used to describe a system that is not immediately consistent but will become consistent over time.
-   This term primarily applies to **consistency in reads**. If you continue issuing a read to a replicating system, you might get old values at first, but eventually, the replication will complete, and you will get the correct, final value.
-   Eventual consistency does **not** apply to **consistency in data** (corruption). If your data is corrupt (e.g., `likes_count` is wrong), there is no eventual consistency that will fix that, unless a separate repair job is run.

There are different types of eventual consistency (strong, weak) and different replication strategies (`synchronous` vs. `asynchronous`) that affect these guarantees. Synchronous replication is slower but provides strong consistency, while asynchronous replication is faster but provides eventual consistency.

---

## Summary
-   **Consistency in Data (ACID "C"):** This refers to the integrity of the persisted data itself, enforced by rules like referential integrity, constraints, atomicity, and isolation. If this is broken, the data is considered corrupt. There is no "eventual consistency" for corrupt data.
-   **Consistency in Reads:** This refers to whether a read operation across a distributed system reflects the most recently committed write. Lapses in this type of consistency are often addressed by the concept of **eventual consistency**, where the system will eventually converge on the correct value.