

#Database #SystemDesign #SQL #NoSQL #NewSQL #DBMS #Architecture #DotNet #InterviewPreparation

> [!abstract] Executive Summary
> A database is not simply a repository for bytes; it is a specialized storage and retrieval engine engineered around deliberate trade-offs between **data integrity, query flexibility, write throughput, latency, and horizontal scale**.
> 
> Choosing the wrong database management system (DBMS) creates structural architectural friction that no amount of clean application code can fix. This guide provides a production-tested breakdown of the **data storage spectrum**, the **CAP/PACELC trade-offs**, an exhaustive catalog of **modern DBMS engines and the exact problems they solve**, official **.NET integration packages**, and the enterprise reality of **Polyglot Persistence with Change Data Capture (CDC)**.

---

## 1. The Modern Data Storage Spectrum

In production architecture, databases do not fall into a simplistic "SQL vs. NoSQL" binary. Instead, modern systems span a continuous spectrum:

```
┌────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                       THE MODERN DATA SPECTRUM                                         │
└────────────────────────────────────────────────────────────────────────────────────────────────────────┘

 [ RELATIONAL (RDBMS) ]         [ DISTRIBUTED SQL (NewSQL) ]               [ NON-RELATIONAL (NoSQL) ]
 PostgreSQL, SQL Server, MySQL   CockroachDB, Google Spanner, YugabyteDB    MongoDB, Redis, Cassandra, Neo4j
          │                                  │                                            │
          ├─► Model: Relational Tables       ├─► Model: Relational Tables                 ├─► Model: Document, Key-Value,
          │   (Rows & Strict Schemas)        │   with Distributed Consensus               │   Wide-Column, Graph, Vector
          ├─► Integrity: Immediate ACID      ├─► Integrity: Distributed ACID              ├─► Integrity: Tunable Consistency
          │   per single instance            │   across geographic nodes                  │   (Strong to Eventual)
          └─► Scaling: Scale-Up primary,     └─► Scaling: Linear horizontal               └─► Scaling: Native horizontal
              Scale-Out read replicas            partitioning (Scale-Out)                     partitioning (Scale-Out)
```

---

### Debunking the Myths: ACID vs. BASE

Historically, textbooks separated systems into a strict binary:
- **SQL = Strict ACID** (Atomicity, Consistency, Isolation, Durability)
- **NoSQL = BASE** (Basically Available, Soft-state, Eventual consistency)

In modern engineering, **this binary is outdated**:
1. **Modern NoSQL engines support ACID:** MongoDB supports multi-document ACID transactions across replica sets and shards; AWS DynamoDB provides native multi-item transactions.
2. **Relational engines exhibit eventual consistency:** When an application writes to a Primary PostgreSQL/SQL Server instance and reads from an asynchronous **Read Replica**, the read path is **eventually consistent** and subject to replication lag.
3. Systems sit on a spectrum of **tunable consistency and operational latency**.

---

### Distributed Systems Trade-Offs: CAP & PACELC

When evaluating distributed databases, two governing theorems dictate what is mathematically possible:

#### 1. The CAP Theorem
In any asynchronous distributed network, you can guarantee at most **two out of three** properties:
- **Consistency ($C$):** Every read receives the most recent write or an error.
- **Availability ($A$):** Every non-failing node returns a successful response (without guaranteeing it is the latest write).
- **Partition Tolerance ($P$):** The system continues functioning despite dropped or delayed network packets between nodes.

> [!IMPORTANT] The CAP Reality
> In real networks, **network partitions ($P$) are inevitable** (cables get cut, switches fail, cloud regions experience hiccups). Therefore, the actual choice is strictly:
> - **CP (Consistency over Availability):** If a partition occurs, refuse writes/reads on minority partitions to prevent data divergence (e.g., PostgreSQL primary, MongoDB majority write, CockroachDB).
> - **AP (Availability over Consistency):** If a partition occurs, all nodes accept reads and writes, resolving divergent conflicts later (e.g., Apache Cassandra, DynamoDB by default, CouchDB).

#### 2. The PACELC Theorem
CAP only describes behavior *during a network failure*. PACELC extends this to normal operations:
$$\text{\textbf{If P} (Partition)} \rightarrow \text{Choose between \textbf{A} (Availability) and \textbf{C} (Consistency);}$$
$$\text{\textbf{Else} (Normal Run)} \rightarrow \text{Choose between \textbf{L} (Latency) and \textbf{C} (Consistency).}$$

*Example:* Even under healthy network conditions, an engine that guarantees immediate consistency across distributed replicas must wait for cross-network round-trips (**higher Latency**), whereas an eventually consistent engine writes to local memory immediately (**lower Latency**).

---

### Consistency Models in Application Design

| Consistency Model | What the Application Observes | Example Engine |
| :--- | :--- | :--- |
| **Strong Consistency (Linearizability)** | A read is guaranteed to observe the latest committed write globally. Zero stale reads. | PostgreSQL (Single node), CockroachDB, Google Spanner |
| **Eventual Consistency** | Given no new updates, all replicas eventually converge. Temporary stale reads are allowed. | Amazon DynamoDB (default reads), Cassandra (low CL), Read Replicas |
| **Read-Your-Writes Consistency** | A user will always observe their own writes immediately, even if other users observe stale data temporarily. | DynamoDB (with consistent reads), Primary-routed writes in .NET |
| **Causal Consistency** | Operations that are causally related (e.g., a Question and its Reply) are observed in the identical order by all nodes. | MongoDB (causal sessions), Cassandra |

---

## 2. The Core DBMS Taxonomy & Sweet Spots

Below is the production-grade catalog of database engines you will encounter in enterprise .NET backend development:

```
┌────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                   CORE DBMS ENGINE TAXONOMY & SWEET SPOTS                              │
├────────────────────────────────┬───────────────────────────────────────────────────────────────────────┤
│ Engine / Family                │ Sweet Spot & Problem Solved                                           │
├────────────────────────────────┼───────────────────────────────────────────────────────────────────────┤
│ 1. PostgreSQL                  │ The Swiss-Army Knife: ACID relational, JSONB documents, and PostGIS   │
│ 2. Microsoft SQL Server        │ Enterprise .NET integration, deep security, Query Store, T-SQL        │
│ 3. MySQL                       │ High read-to-write web applications, classic primary-replica workloads│
│ 4. CockroachDB / Spanner       │ Distributed SQL (NewSQL): Global horizontal scale with strict ACID    │
│ 5. MongoDB                     │ Polymorphic schemas, hierarchical documents, rapid schema evolution   │
│ 6. Redis                       │ Sub-millisecond in-memory caching, distributed locks, session state   │
│ 7. Apache Cassandra / DynamoDB │ Unbounded write velocity, IoT telemetry, high availability at scale   │
│ 8. Elasticsearch / OpenSearch  │ Inverted index full-text search, fuzzy matching, centralized logging  │
│ 9. Vector Stores (pgvector)    │ High-dimensional AI embeddings, RAG pipelines, similarity search      │
│ 10. Time-Series (TimescaleDB)  │ High-frequency append-only telemetry, automated time rollups          │
│ 11. Neo4j                      │ Multi-hop graph traversals, fraud detection, recommendation engines   │
└────────────────────────────────┴───────────────────────────────────────────────────────────────────────┘
```

---

### Category A: Relational Database Engines (RDBMS)

#### 1. PostgreSQL (The Extensible Swiss-Army Knife)
* **What it is:** The world's most advanced, open-source object-relational database.
* **The Problem It Solves:** You require strict relational integrity and complex joins, but also need modern features like indexing semi-structured JSON, GIS geographic coordinates, and AI vector embeddings without spinning up three separate databases.
* **Key Superpowers:**
  * **`JSONB` Data Type:** Binary-encoded JSON with GIN indexing, allowing hybrid relational + document modeling.
  * **Rich Extensions Ecosystem:** `pgvector` for vector similarity, `PostGIS` for geospatial queries, `timescaledb` for time-series.
  * **MVCC Maturity:** Readers never block writers and writers never block readers.
* **Scaling Strategy:** Primarily scale-up (vertical) for primary transactional writes; scales horizontally for read queries via streaming read replicas; supports horizontal sharding via declarative table partitioning and extensions (e.g., Citus).
* **Official .NET Integration:** `Npgsql.EntityFrameworkCore.PostgreSQL`, `Dapper`.
* **When NOT to Choose:** Extremely high, globally distributed multi-region write workloads exceeding 150,000 writes/sec where masterless multi-master sharding is required.

---

#### 2. Microsoft SQL Server (The Enterprise .NET Titan)
* **What it is:** Microsoft’s flagship enterprise relational engine, powered by T-SQL.
* **The Problem It Solves:** Mission-critical enterprise applications in the Microsoft ecosystem requiring native Active Directory authentication, deep .NET performance optimizations, automated performance diagnostics, and strict regulatory compliance.
* **Key Superpowers:**
  * **Query Store:** Automatically captures historical execution plans, detecting query performance regressions without external monitoring tools.
  * **Temporal Tables:** Automatic, system-versioned historical tracking of every row change (`FOR SYSTEM_TIME AS OF`).
  * **Always On Availability Groups:** Zero-data-loss failover clusters with readable secondary replicas.
* **Scaling Strategy:** Primarily vertical scale-up for primary instance; read scale-out via readable Availability Group secondaries; enterprise horizontal scale via Azure SQL Hyperscale and Elastic Database Pools.
* **Official .NET Integration:** `Microsoft.EntityFrameworkCore.SqlServer`, `Microsoft.Data.SqlClient`, `Dapper`.
* **When NOT to Choose:** Open-source architectures with strict zero-cost licensing budgets, or lightweight containerized edge microservices where engine memory footprint must be tiny.

---

#### 3. MySQL (The High-Read Web Workhorse)
* **What it is:** The classic open-source relational database that powers WordPress, Drupal, and high-read web properties.
* **The Problem It Solves:** Fast primary-key reads and simple relational schemas with low administrative complexity.
* **Key Superpowers:** Simple setup, mature Group Replication, and MySQL InnoDB Cluster for automated high availability.
* **Scaling Strategy:** Vertical scale-up for primary; horizontal read scaling via multi-replica replication.
* **Official .NET Integration:** `Pomelo.EntityFrameworkCore.MySql`.
* **When NOT to Choose:** Complex analytical queries requiring advanced SQL windowing, rich JSON query operators, or custom transactional extensions.

---

### Category B: Distributed Relational Engines (NewSQL)

#### 4. CockroachDB / Google Cloud Spanner / YugabyteDB
* **What it is:** Cloud-native, distributed relational databases that combine the horizontal scalability of NoSQL with the familiar SQL syntax and ACID guarantees of RDBMS.
* **The Problem It Solves:** Your relational database has outgrown a single vertical server, but you cannot abandon SQL, multi-table joins, foreign keys, or ACID consistency.
* **Key Superpowers:**
  * **Automated Range Sharding:** Data is automatically split into contiguous chunks (ranges) and distributed evenly across cluster nodes.
  * **Consensus-Driven Replication:** Uses Raft or Paxos consensus across distributed nodes. Nodes can fail without taking down the cluster or corrupting transactions.
  * **Wire Compatibility:** CockroachDB is wire-compatible with PostgreSQL clients.
* **Official .NET Integration:** `Npgsql` (CockroachDB), `Google.Cloud.Spanner.Data`.
* **When NOT to Choose:** Small-to-medium single-region workloads. The network latency overhead of distributed consensus makes single-row writes slower than single-node PostgreSQL.

---

### Category C: Document & In-Memory Key-Value Stores

#### 5. MongoDB (The Flexible Document Store)
* **What it is:** A distributed database storing data as JSON-like BSON (Binary JSON) documents.
* **The Problem It Solves:** 
  1. **Hierarchical / Nested Data:** Storing an Order with Line Items, Shipping Addresses, and Payment Metadata as a single, cohesive document without multi-table relational joins.
  2. **Polymorphic Schemas:** Catalogs where different entities have vastly different attributes (e.g., Shoes have sizes; Laptops have RAM and GPU).
* **Key Superpowers & Modern Reality:**
  * **JSON Schema Validation:** Since version 3.6, MongoDB natively enforces strict, declarative schema validation rules at the collection level if configured.
  * **Multi-Document ACID Transactions:** Supported since version 4.0 across replica sets and sharded clusters (though costlier than single-node SQL due to two-phase distributed commit coordination).
* **Scaling Strategy:** Native horizontal partitioning (sharding) by partition key.
* **Official .NET Integration:** `MongoDB.Driver`, `MongoDB.EntityFrameworkCore`.
* **When NOT to Choose:** Financial ledgers or systems with complex, heavily normalized many-to-many relationships where foreign keys and cross-table joins are the primary access pattern.

---

#### 6. Redis (The In-Memory Speed Demon)
* **What it is:** An ultra-fast, in-memory Key-Value and data structure store.
* **The Problem It Solves:** Relational disk queries require 2ms–50ms. Under heavy traffic, hitting the database for every read exhausts connection pools and causes CPU saturation. Redis holds data directly in **RAM**, returning responses in **sub-millisecond (<1ms)** latency.
* **Key Superpowers:**
  * **In-Memory Data Structures:** Strings, Hashes, Lists, Sets, Sorted Sets (`ZSET` for leaderboards), Bitmaps, and HyperLogLogs.
  * **Atomic Operations:** Single-threaded execution loop guarantees atomic increments (`INCR`), list pushes, and lock acquisition.
  * **Redis Pub/Sub & Streams:** Lightweight pub/sub messaging and message stream processing.
* **Official .NET Integration:** `StackExchange.Redis`, `Microsoft.Extensions.Caching.StackExchangeRedis`.
* **When NOT to Choose:** Primary persistent storage for massive datasets where RAM costs would be prohibitive, or queries requiring ad-hoc sorting and complex multi-field filtering.

---

### Category D: Distributed Wide-Column & Managed NoSQL

#### 7. Apache Cassandra vs. Amazon DynamoDB
* **Apache Cassandra (Masterless Peer-to-Peer):**
  * **Architecture:** Every node in the cluster is identical (no primary/replica distinction). Requests route to any node via consistent hashing on a token ring.
  * **Fault Tolerance & Quorum:** Does not stay up blindly; availability is strictly governed by **Replication Factor ($RF$)** and **Consistency Level ($CL$)**. For example, with $RF=3$ and $CL=\text{QUORUM}$ ($\lfloor 3/2 \rfloor + 1 = 2$), the cluster survives the loss of exactly **1 node**.
* **Amazon DynamoDB (Managed Partitioned Cloud Store):**
  * **Architecture:** Fully managed AWS multi-AZ service. Internally partitions data across storage nodes with synchronous replication across 3 Availability Zones.
  * **Consistency:** Eventually consistent reads by default; strongly consistent reads supported via a query flag.
* **The Problem They Solve:** Ingestion of massive, unstoppable write streams (hundreds of thousands of writes/second) like IoT telemetry, clickstream logs, and global user activities.
* **Official .NET Integration:** `CassandraCSharpDriver`, `AWSSDK.DynamoDBv2`.
* **When NOT to Choose:** Workloads requiring flexible, ad-hoc queries, joins, or dynamic `WHERE` filtering. You must design tables specifically around predetermined query access patterns.

---

### Category E: Specialized Search, Vector & Time-Series

#### 8. Elasticsearch / OpenSearch (Full-Text Search & Analytics)
* **What it is:** A distributed search engine built on Apache Lucene.
* **The Problem It Solves:** Relational databases execute full-table scans for wildcard text queries (`LIKE '%term%'`) and cannot handle typos, synonyms, or linguistic relevance scoring. Elasticsearch provides typo-tolerant, ranked full-text search.
* **Key Superpowers:** The **Inverted Index** (mapping words directly to document IDs), tokenizers, stemmers, and n-gram analyzers.
* **Official .NET Integration:** `Elastic.Clients.Elasticsearch`.
* **When NOT to Choose:** Primary system-of-record transactional storage. It does not provide ACID transactional guarantees and is susceptible to data loss under severe partition conditions.

---

#### 9. Dedicated Vector Databases & pgvector (AI & Similarity Search)
* **What it is:** Specialized stores for high-dimensional numerical vectors (embeddings) generated by machine learning models (e.g., OpenAI embeddings).
* **The Problem It Solves:** Calculating semantic similarity across millions of unstructured texts, images, or audio files using mathematical algorithms (Cosine Similarity, Euclidean Distance, HNSW indexes).
* **Engines:** `pgvector` (PostgreSQL extension), `Qdrant`, `Pinecone`, `Milvus`.
* **Official .NET Integration:** `pgvector-dotnet` with Npgsql, `Microsoft.SemanticKernel`, official client SDKs.
* **When NOT to Choose:** Standard transactional business records without AI embeddings or semantic search requirements.

---

#### 10. Time-Series Databases (TimescaleDB / InfluxDB)
* **What it is:** Storage engines optimized specifically for immutable, timestamped data points.
* **The Problem It Solves:** Ingesting high-velocity metric streams (server CPU monitoring, financial tick data, factory sensors) with automated time-bucket rollups and data retention policies.
* **Engines:** `TimescaleDB` (packaged as a PostgreSQL extension), `InfluxDB`.
* **Official .NET Integration:** `Npgsql` (for TimescaleDB), `InfluxDB.Client`.
* **When NOT to Choose:** Systems requiring frequent random row updates or deletes of historical records.

---

### Category F: Graph Database Engines

#### 11. Neo4j (The Graph Specialist)
* **What it is:** A native graph database that models data as **Nodes** (entities), **Edges** (relationships), and **Properties**.
* **The Problem It Solves:** Querying deep, multi-hop relationship traversals. Finding "friends of friends of friends" in SQL requires nested, recursive `JOIN`s that destroy database performance. Neo4j executes pointer-hopping traversals in constant time relative to the visited sub-graph.
* **Key Superpowers:** Cypher query language, native graph index-free adjacency.
* **Official .NET Integration:** `Neo4j.Driver`.
* **When NOT to Choose:** Traditional tabular business records, simple CRUD apps, or workloads requiring bulk aggregate analytics across flat tables.

---

## 3. Master DBMS Engine Comparison Matrix

| DBMS Engine | Data Model | Scaling Architecture | CAP Classification | Official .NET Package | Primary Sweet Spot | Avoid When... |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **PostgreSQL** | Relational + `JSONB` | Scale-Up write, Scale-Out read replicas | CP (Primary) | `Npgsql.EntityFrameworkCore.PostgreSQL` | Standard backend default, hybrid relational + JSON | Distributed multi-region writes >150k/sec |
| **SQL Server** | Relational (T-SQL) | Scale-Up write, Scale-Out AG replicas | CP (Primary) | `Microsoft.EntityFrameworkCore.SqlServer` | Enterprise .NET apps, Query Store, compliance | Zero-cost open-source budgets |
| **MySQL** | Relational | Scale-Up write, Scale-Out read replicas | CP (Primary) | `Pomelo.EntityFrameworkCore.MySql` | Read-heavy web platforms, CMS | Complex analytical window queries |
| **CockroachDB**| Distributed SQL | Native Horizontal (Raft consensus) | CP | `Npgsql` | Global multi-region scale with strict ACID | Small single-region apps (consensus latency) |
| **MongoDB** | Document (BSON) | Native Horizontal (Sharding) | CP (by default) | `MongoDB.Driver` | Hierarchical data, polymorphic schemas | Highly normalized many-to-many tabular data |
| **Redis** | Key-Value in RAM | Horizontal (Redis Cluster) | CP / AP tunable | `StackExchange.Redis` | Sub-millisecond cache, sessions, locks | Primary persistence for datasets larger than RAM |
| **Cassandra** | Wide-Column | Masterless Peer-to-Peer Ring | AP (Tunable) | `CassandraCSharpDriver` | Massive write velocity, IoT telemetry | Ad-hoc queries, joins, unknown read access patterns |
| **DynamoDB** | Key-Value / Doc | Managed Multi-AZ Horizontal | AP / CP tunable | `AWSSDK.DynamoDBv2` | Serverless AWS apps, predictable single-digit ms | Multi-cloud, ad-hoc joins, complex filtering |
| **Elasticsearch**| Inverted Index | Horizontal Cluster | AP (Analytics) | `Elastic.Clients.Elasticsearch` | Fuzzy full-text search, ELK log analytics | Primary transactional source-of-truth |
| **pgvector / Qdrant**| Vector Embeddings| Horizontal / Node | CP / AP | `pgvector-dotnet` / `Qdrant.Client` | AI semantic similarity, RAG search | Standard relational CRUD operations |
| **TimescaleDB** | Time-Series | Scale-Up + Distributed Hypertables| CP | `Npgsql` | Append-only metrics, automated time rollups | Random historical row updates |
| **Neo4j** | Property Graph | Scale-Up write, Clustering | CP | `Neo4j.Driver` | Fraud rings, social networks, knowledge graphs| Tabular relational business entities |

---

## 4. The Practical 4-Step Decision Framework

When designing a backend service or answering a system design interview prompt, apply this decision sequence:

```
Step 1: Does the domain require strict relational integrity, complex multi-table joins, and ACID compliance?
 ├── YES ──► Does the transactional write volume exceed what a single scaled-up instance can handle?
 │            ├── NO  ──► Pick PostgreSQL (or SQL Server for enterprise .NET ecosystems).
 │            └── YES ──► Pick Distributed SQL (CockroachDB / Google Spanner).
 └── NO  ──► Proceed to Step 2.

Step 2: Is the data naturally structured as self-contained, nested documents with evolving schemas?
 ├── YES ──► Pick MongoDB (Document Store).
 └── NO  ──► Proceed to Step 3.

Step 3: What is the primary operational access bottleneck?
 ├── Sub-millisecond latency on repeated read paths? ──► Pick Redis (In-Memory Key-Value).
 ├── Unstoppable, massive write throughput across distributed nodes? ──► Pick Cassandra or DynamoDB.
 ├── Human-language text search, typos, and faceted filtering? ──► Pick Elasticsearch / OpenSearch.
 ├── High-dimensional AI embeddings & semantic search? ──► Pick pgvector / Qdrant.
 ├── Continuous, immutable timestamped metric streams? ──► Pick TimescaleDB / InfluxDB.
 └── Deep, multi-hop relationship traversals (fraud, social)? ──► Pick Neo4j (Graph).
```

---

## 5. Enterprise Reality: Polyglot Persistence & CDC

In production, mature applications **almost never rely on a single database engine**. 

Instead, systems implement **Polyglot Persistence**: using distinct specialized storage engines for distinct responsibilities, linked together via **Change Data Capture (CDC)** to eliminate the dangerous **Dual-Write Problem**:

```
┌────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                        POLYGLOT PERSISTENCE WITH CHANGE DATA CAPTURE (CDC)                             │
│                                                                                                        │
│   ┌────────────────────────────────────────────────────────────────────────────────────────────────┐   │
│   │                                      ASP.NET Core Web API                                      │   │
│   └───────────────────────────────────────────────┬────────────────────────────────────────────────┘   │
│                                                   │                                                    │
│                                                   ▼ 1. ACID Write (Single Source of Truth)             │
│                                      ┌─────────────────────────┐                                       │
│                                      │  PostgreSQL/SQL Server  │                                       │
│                                      └────────────┬────────────┘                                       │
│                                                   │ 2. WAL / CDC Stream (Debezium / Outbox)            │
│                                                   ▼                                                    │
│                                      ┌─────────────────────────┐                                       │
│                                      │   Apache Kafka / Bus    │                                       │
│                                      └────────────┬────────────┘                                       │
│                                                   │                                                    │
│                         ┌─────────────────────────┼─────────────────────────┐                          │
│                         ▼ 3a. Update Cache        ▼ 3b. Index Search Doc    ▼ 3c. Generate Embedding  │
│                  ┌─────────────┐           ┌─────────────┐           ┌─────────────┐                   │
│                  │    Redis    │           │Elasticsearch│           │   Qdrant    │                   │
│                  │ (Sub-ms RAM)│           │ (Full-Text) │           │  (Vectors)  │                   │
│                  └─────────────┘           └─────────────┘           └─────────────┘                   │
└────────────────────────────────────────────────────────────────────────────────────────────────────────┘
```

### The Dual-Write Danger & How CDC Solves It
- **The Anti-Pattern (Application Dual-Write):** The Web API writes an order to PostgreSQL, and then immediately tries to write the search document to Elasticsearch over HTTP. If the network drops or Elasticsearch fails during the second call, your search engine is permanently out-of-sync with your database.
- **The Solution (Change Data Capture & Outbox Pattern):**
  1. The Web API writes **strictly to the primary relational database** (PostgreSQL / SQL Server) within a transaction.
  2. A CDC tool (e.g., **Debezium**, SQL Server CDC, or the **Transactional Outbox Pattern**) reads the database's internal transaction log (WAL).
  3. The events are published to a message broker (e.g., Kafka / RabbitMQ) and projected asynchronously into Redis, Elasticsearch, and Vector stores.
  4. Data consistency is guaranteed eventually without tying transactional performance to secondary storage availability.

---

## 6. Related Notes in this Vault

- [[1_Indexing_and_Query_Tuning|01. Indexing, B+Trees, and Query Optimization]]
- [[Transactions_ACID_and_Concurrency_Control|01. Transactions, ACID, Isolation Levels, and MVCC]]
- [[04_Modern_DotNet_Data_Access_EF_Core_and_Performance|04. Modern .NET Data Access: EF Core vs. Dapper]]
- [[05_Practical_Caching_Strategies_with_Redis|05. Practical Caching Strategies with Redis]]
- [[02_Modern Web Communication Paradigms_ Beyond Request-Response|02. Modern Web Communication Paradigms]]