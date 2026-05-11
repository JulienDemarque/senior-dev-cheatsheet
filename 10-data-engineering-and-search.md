# Data engineering & search

Storage engine ideas (**LSM tree**, **B-tree**), pipelines, and search—plus safe SQL patterns.

## ETL

Batch extract from sources, transform (clean/join/aggregate), load into warehouse tables. **ELT** flips order: load raw to lake, transform in SQL/Spark inside the warehouse.

### Example (minimal SQL transform)

```sql
INSERT INTO dw.facts_orders (order_id, amount_cents, day)
SELECT id, amount_cents, date_trunc('day', created_at)
FROM staging.orders_raw;
```

## Data orchestration

DAG of tasks with schedules, retries, SLAs, and data sensors (wait until partition exists). Tools: Airflow, Dagster, Prefect, Temporal for workflows.

## LSM tree

Writes buffer in **MemTable**, flush immutable sorted SSTables, reads merge levels; **compaction** reclaims space and bounds read amplification. Great write throughput; tuning `level0` stalls matters.

## B-tree

Page-oriented balanced tree; each node holds many keys (high fanout) fitting disk pages. Postgres primary key B-tree index is the default mental model.

## MemTable

Mutable in-memory write buffer before SSTable flush; durability requires **WAL** fsync policy tradeoffs.

## Compaction

Merge SSTables, drop tombstones, maintain level invariant. Write amplification vs read amplification tradeoff.

## Sequential writes

Append-only logs and LSM flushes sequentialize disk IO—high throughput on HDDs and still friendly on NVMe.

## Random writes

Small updates scattered across pages cause read-modify-write on storage; B-trees mitigate with buffer pool but still pay cost vs append-only.

## Full-text search

Inverted index: term → posting list (doc ids + positions/weights). Query: intersect posting lists, score (BM25).

## Elasticsearch

Lucene indices sharded across nodes; REST JSON query DSL; aggregations for logs/metrics. Operate with care: JVM heap, shard count, ILM for retention.

### Example (minimal query)

```json
POST /articles/_search
{
  "query": { "match": { "title": "distributed systems" } },
  "size": 10
}
```

## OpenSearch

API-compatible fork focused on AWS ecosystem; similar cluster concepts (indices, shards, snapshots).

## SQL injection

Untrusted input concatenated into SQL becomes executable code. Fix: **only** parameterized queries / bound parameters.

### Example (unsafe — never do this)

```python
query = f"SELECT * FROM users WHERE name = '{name}'"
```

### Example (safe — parameterized)

```python
cur.execute("SELECT * FROM users WHERE name = %s", (name,))
```

ORMs still allow raw SQL footguns—always bind parameters.
