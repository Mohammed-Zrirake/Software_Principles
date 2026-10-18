#Database #PostgreSQL #Cursors #ServerSideCursor #SQL #NodeJS #Python #Streaming #SystemDesign #Tutorial

> [!abstract] Executive Summary
> In PostgreSQL, a **Server-Side Cursor** is an encapsulated query execution context that allows applications to retrieve query results incrementally. Rather than materializing all matching rows into memory or transmitting them across the network in one massive burst, the database engine maintains an internal cursor position pointer.
> 
> A server-side cursor **must be declared and executed inside an active transaction block**. This guide breaks down the full SQL cursor grammar (`DECLARE`, `FETCH`, `MOVE`, `CLOSE`), explores **Scrollable vs. Holdable** cursor variants, provides complete implementations in **Python (`psycopg2`)** and **Node.js (`pg-cursor`)**, and demonstrates how to construct high-throughput **streaming pipelines** (HTTP Chunked Transfer, WebSockets, gRPC).

---

## 1. The PostgreSQL Cursor Lifecycle

A PostgreSQL server-side cursor progresses through four distinct lifecycle phases:

```mermaid
stateDiagram-v2
    [*] --> Inactive: Connection established
    Inactive --> Declared: BEGIN & DECLARE cursor_name CURSOR FOR query
    note right of Declared: Execution plan prepared.<br/>Pointer sits BEFORE the first row.<br/>ZERO data transmitted!
    
    Declared --> Fetching: FETCH [n] FROM cursor_name
    Fetching --> Fetching: FETCH NEXT / FETCH FORWARD [n]
    note right of Fetching: Chunks of tuples streamed over wire.<br/>Client processes and frees local RAM.
    
    Fetching --> Repositioning: MOVE FORWARD [n] FROM cursor_name
    Repositioning --> Fetching: Pointer shifted without network payload
    
    Fetching --> Closed: CLOSE cursor_name OR COMMIT / ROLLBACK
    note right of Closed: Server-side memory & locks released.
    Closed --> [*]
```

---

## 2. PostgreSQL SQL Syntax Deep Dive

The official PostgreSQL syntax for declaring cursors is:

```sql
DECLARE name [ BINARY ] [ INSENSITIVE ] [ [ NO ] SCROLL ] 
    CURSOR [ { WITH | WITHOUT } HOLD ] 
    FOR query;
```

### Key Modifiers Explained:

1. **`SCROLL` vs. `NO SCROLL`:**
   - **`NO SCROLL` (Default Forward-Only):** The cursor can only move forward (`FETCH NEXT`, `FETCH FORWARD n`). Highly efficient; requires minimal memory buffers on the server.
   - **`SCROLL` (Bidirectional):** The cursor can navigate backwards and jump (`FETCH PRIOR`, `FETCH BACKWARD n`, `FETCH FIRST`, `FETCH LAST`).
   - *Performance Warning:* As noted by database engineers, `FETCH LAST` on an unindexed query forces the database to evaluate the entire table to determine the final row before returning it!

2. **`WITH HOLD` vs. `WITHOUT HOLD`:**
   - **`WITHOUT HOLD` (Default):** The cursor is strictly bound to the active transaction. Executing `COMMIT` or `ROLLBACK` immediately destroys the cursor and frees server resources.
   - **`WITH HOLD`:** The cursor **survives transaction commit**! It remains open across subsequent transactions on the same connection until explicitly closed or the connection terminates.

---

## 3. The `FETCH` and `MOVE` Command Family

Once a cursor is declared, the client controls data retrieval using `FETCH` and `MOVE`:

```
┌─────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                 POSTGRESQL FETCH & MOVE COMMANDS                                │
├─────────────────────────┬───────────────────────────────────────────────────────────────────────┤
│ Command                 │ Action & Wire Transmission                                            │
├─────────────────────────┼───────────────────────────────────────────────────────────────────────┤
│ `FETCH 1 FROM c;`       │ Retrieves the next 1 row and advances pointer.                        │
├─────────────────────────┼───────────────────────────────────────────────────────────────────────┤
│ `FETCH 100 FROM c;`     │ Retrieves the next 100 rows as a batch.                               │
├─────────────────────────┼───────────────────────────────────────────────────────────────────────┤
│ `FETCH ALL FROM c;`     │ Retrieves all remaining rows from current position to end.            │
├─────────────────────────┼───────────────────────────────────────────────────────────────────────┤
│ `FETCH PRIOR FROM c;`   │ (Requires `SCROLL`) Retrieves the row immediately preceding pointer.  │
├─────────────────────────┼───────────────────────────────────────────────────────────────────────┤
│ `FETCH FIRST FROM c;`   │ (Requires `SCROLL`) Jumps to and retrieves the very first row.        │
├─────────────────────────┼───────────────────────────────────────────────────────────────────────┤
│ `FETCH LAST FROM c;`    │ (Requires `SCROLL`) Jumps to and retrieves the very last row.         │
├─────────────────────────┼───────────────────────────────────────────────────────────────────────┤
│ `MOVE 500 FROM c;`      │ **Advances pointer by 500 rows WITHOUT sending any data across wire!**│
│                         │ Ideal for fast-forwarding over unneeded rows with zero network I/O.   │
├─────────────────────────┼───────────────────────────────────────────────────────────────────────┤
│ `CLOSE c;`              │ Immediately frees the cursor from PostgreSQL server memory.          │
└─────────────────────────┴───────────────────────────────────────────────────────────────────────┘
```

---

## 4. Hands-On SQL Laboratory: The 12-Million Grades Walkthrough

Open `psql` and execute the following sequential commands:

### Step 1: Open Transaction & Declare Cursor
```sql
BEGIN;

-- Declare server-side cursor for all grades >= 90
DECLARE top_students_cur CURSOR FOR 
    SELECT student_id, course_id, grade 
    FROM grades 
    WHERE grade BETWEEN 90 AND 100 
    ORDER BY student_id ASC;
```
*Note:* The query returns `DECLARE CURSOR` instantly in **< 1ms**. PostgreSQL has formulated the execution plan, but has **not yet scanned or transferred the 1,000,000 tuples**.

### Step 2: Fetch Controlled Chunks
```sql
-- Retrieve first batch of 5 rows
FETCH 5 FROM top_students_cur;
```
**Output:**
```text
 student_id | course_id | grade 
------------+-----------+-------
       1002 | CS-101    |    94
       1008 | MATH-201  |    98
       1015 | PHYS-101  |    91
       1022 | CS-101    |    99
       1030 | CHEM-101  |    90
(5 rows)
```

### Step 3: Fast-Forward with `MOVE`
```sql
-- Skip the next 1,000 rows without network overhead:
MOVE 1000 FROM top_students_cur;
-- Output: MOVE 1000

-- Fetch the next row after the skip:
FETCH 1 FROM top_students_cur;
```

### Step 4: Clean Up and Release Server Resources
```sql
CLOSE top_students_cur;
COMMIT;
```

---

## 5. Application Implementations: Node.js & Python

### Pattern A: Node.js Streaming with `pg-cursor`
In Node.js, standard `node-postgres` (`pg`) buffers entire query results. To enable true server-side cursors, combine `pg` with the official **`pg-cursor`** package:

```javascript
const { Pool } = require('pg');
const Cursor = require('pg-cursor');

const pool = new Pool({ connectionString: 'postgres://postgres:postgres@localhost:5432/university' });

async function streamLargeExport(res) {
    const client = await pool.connect();
    
    try {
        await client.query('BEGIN');
        
        // 1. Declare cursor using pg-cursor
        const queryText = 'SELECT student_id, grade FROM grades WHERE grade >= 90';
        const cursor = client.query(new Cursor(queryText));

        // 2. Fetch rows in batches of 2,000
        const batchSize = 2000;
        
        const fetchNextBatch = () => {
            cursor.read(batchSize, (err, rows) => {
                if (err) throw err;
                
                // If rows is empty, cursor reached the end:
                if (rows.length === 0) {
                    cursor.close(() => {
                        client.query('COMMIT', () => client.release());
                        res.end(); // Complete HTTP stream
                    });
                    return;
                }

                // 3. Stream batch directly to HTTP response:
                for (const row of rows) {
                    res.write(`${row.student_id},${row.grade}\n`);
                }

                // Recursively fetch next batch
                fetchNextBatch();
            });
        };

        fetchNextBatch();
    } catch (err) {
        await client.query('ROLLBACK');
        client.release();
        res.status(500).send(err.message);
    }
}
```

---

### Pattern B: Python Generator Streaming with `psycopg2`
Python's generator syntax (`yield`) paired with named cursors provides an elegant streaming pipeline:

```python
import psycopg2

def stream_grade_records(batch_size=2000):
    conn = psycopg2.connect("dbname=university user=postgres password=postgres")
    
    try:
        # Named cursor forces PostgreSQL server-side cursor!
        with conn.cursor(name='student_grades_cur') as cur:
            cur.itersize = batch_size
            cur.execute("SELECT student_id, grade FROM grades WHERE grade >= 90")
            
            # Yield records one by one without buffering the entire table in RAM:
            for record in cur:
                yield record
        
        conn.commit()
    finally:
        conn.close()

# Consuming the streaming generator:
for student_id, grade in stream_grade_records():
    send_to_downstream_pipeline(student_id, grade)
```

---

## 6. Real-Time Streaming Architecture (WebSockets / gRPC / HTTP Chunked)

Server-side cursors are the engine behind high-throughput real-time data streaming architectures:

```mermaid
flowchart LR
    subgraph Storage["PostgreSQL Engine"]
        Cur[("Server-Side Cursor<br/>1,000,000 Tuples")]
    end

    subgraph Service["Node.js / Python Backend"]
        Worker["Worker Process<br/>(Lean 20 MB RAM)"]
    end

    subgraph Consumer["Downstream Clients"]
        ClientA["Browser (HTTP Chunked Export)"]
        ClientB["Microservice (gRPC Stream)"]
        ClientC["Dashboard (WebSocket Feed)"]
    end

    Cur -->|FETCH 2000 Chunks| Worker
    Worker -->|res.write Chunk| ClientA
    Worker -->|gRPC Message Stream| ClientB
    Worker -->|WebSocket Frames| ClientC
```

### The Beauty of Early Cancellation:
If a user downloading a 10-million-row CSV export closes their browser tab after 50,000 rows:
1. The backend detects the socket disconnection (`req.on('close')`).
2. The backend immediately issues `CLOSE cursor; ROLLBACK;`.
3. **PostgreSQL instantly aborts the remaining 9,950,000 row evaluations!** Zero CPU or I/O is wasted computing records nobody will ever read.

---

## 7. Senior Engineer Production Checklist

- [ ] **Always close cursors explicitly:** While committing or rolling back closes `WITHOUT HOLD` cursors, always issue explicit `CLOSE cursor_name;` to release server working memory immediately.
- [ ] **Beware of `FETCH LAST` on non-indexed columns:** If a cursor does not leverage an indexed reverse scan, `FETCH LAST` forces a complete table scan to locate the final record.
- [ ] **Tune batch sizes (1,000 - 5,000):** Fetching 1 row at a time burns CPU in driver/network latency; fetching 50,000 rows risks client OOM.
- [ ] **Handle client disconnects gracefully:** Always hook into connection abortion events to terminate long-running cursor transactions immediately when the downstream consumer disconnects.

---

## 8. Related Notes & Conceptual Navigation

- [[01_Database Cursors Fundamentals_ Server-Side vs. Client-Side Cursors|Database Cursors Fundamentals: Server-Side vs. Client-Side]]
- [[03_Database Cursors Trade-Offs_ Statefulness, Long-Running Transactions, and Production Pitfalls|Database Cursors Trade-Offs: Statefulness & Production Pitfalls]]
- [[04_Hands-On Benchmark_ Client-Side vs. Server-Side Cursors in Python (1 Million Rows)|Hands-On Benchmark: Client-Side vs. Server-Side Cursors in Python (1 Million Rows)]]
- [[04_SQL Pagination at Scale_ Why OFFSET Kills Performance and How Keyset (Seek) Pagination Solves It|SQL Pagination at Scale: Keyset vs. Offset]]
- [[05_Database Connection Pooling_ Mechanics, Node.js Benchmark, and Sizing Strategies|Database Connection Pooling]]

