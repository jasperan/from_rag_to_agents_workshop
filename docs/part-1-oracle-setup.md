# Part 1: Oracle AI Database Setup & Connection

## What You Are Working With

Oracle AI Database 23ai (and 26ai) is a **converged database** built for AI developers. It unifies relational, document, graph, and vector data in a single engine with native support for:

- **`VECTOR` column type** — stores embeddings as first-class SQL values
- **HNSW indexes** — approximate nearest-neighbour search directly in SQL
- **`VECTOR_DISTANCE()` function** — cosine, dot product, and Euclidean distance in SQL queries
- **Oracle Text indexes** — full-text keyword search with `CONTAINS()`
- **SQL Property Graphs** — graph traversal with `GRAPH_TABLE()`

This means your retrieval pipeline — keyword, vector, hybrid, and graph — all live in a single, queryable, ACID-compliant database.

## Your Environment

In this Codespace, Oracle AI Database is already running as a Docker service (`gvenzl/oracle-free:23-slim`). The service starts automatically and passes a healthcheck before your development container boots.

| Setting | Value |
|---|---|
| Host | `localhost` |
| Port | `1521` |
| Service name | `FREEPDB1` |
| SYS password | `OraclePwd_2025` |
| App user | `VECTOR` |
| App user password | `VectorPwd_2025` |

You will connect as the `VECTOR` user for all workshop tasks.

## TODO: Implement `connect_to_oracle`

**Why retry logic?** Docker healthchecks verify the container is running, but Oracle's listener can take a few extra seconds to become fully ready after the healthcheck passes. A retry loop makes the connection resilient to this transient window.

**What `oracledb.connect()` needs:**

```python
oracledb.connect(
    user="VECTOR",
    password="VectorPwd_2025",
    dsn="localhost:1521/FREEPDB1"
)
```

The `dsn` format is `host:port/service_name`.

**Complete solution:**

```python
import oracledb
import time

def connect_to_oracle(max_retries=3, retry_delay=5):
    user = "VECTOR"
    password = "VectorPwd_2025"
    dsn = "localhost:1521/FREEPDB1"

    for attempt in range(1, max_retries + 1):
        try:
            print(f"Connection attempt {attempt}/{max_retries}...")
            conn = oracledb.connect(user=user, password=password, dsn=dsn)
            print("Connected successfully!")
            with conn.cursor() as cur:
                cur.execute("SELECT banner FROM v$version WHERE banner LIKE 'Oracle%'")
                print(cur.fetchone()[0])
            return conn
        except oracledb.OperationalError as e:
            print(f"Attempt {attempt} failed: {e}")
            if attempt < max_retries:
                print(f"Retrying in {retry_delay}s...")
                time.sleep(retry_delay)
            else:
                raise
```

## Property Graph Privileges

The notebook also grants `CREATE PROPERTY GRAPH` privileges via a SYS connection. This is needed for Part 4 (graph retrieval). The helper function `ensure_property_graph_privileges()` is pre-built for you.

## Troubleshooting

**"ORA-12541: TNS:no listener"** — The listener is still starting. Wait 30 seconds and retry.

**"DPY-4011" or "Connection reset by peer"** — Database is still starting up. Wait 2-3 minutes.

**"ORA-01017: invalid username/password"** — Check you are using `VECTOR` / `VectorPwd_2025`.

**"Could not reach Oracle after all retries"** — Rebuild the Codespace: `Codespaces: Rebuild Container`.
