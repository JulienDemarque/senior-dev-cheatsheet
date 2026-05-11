# Distributed systems & data stores

Replication, consistency models, transactions, and storage primitives—plus SQL and operational snippets.

## Distributed systems

Multiple nodes over an unreliable network: partial failure is normal, clocks are not global, and “exactly once” end-to-end is usually *simulated* with at-least-once delivery + **idempotency**.

## Eventual consistency

If writes stop, replicas eventually agree. Reads may be stale mid-flight. Good for social counters, DNS; awkward for “did my payment land?” without extra protocol.

## Read-after-write consistency

User sees their own write immediately (sticky to **primary**, version token on read, or RYW guarantees in Cosmos-style APIs). Stronger UX than plain **eventual consistency**.

### Example (read your writes with version)

```text
Client sends If-Match: v3 on PUT; server rejects if not current.
GET returns ETag: v4 after write so client knows it sees own version.
```

## Replication lag

Time for a committed write on **leader** to appear on **followers**. Breaks naive “write then read replica” patterns; causes flaky tests if CI reads replica.

## Leader/follower replication

Single writer (**primary**), many read replicas apply the same log (often **WAL** or logical stream). Failover promotes a replica—lossy if unreplicated writes existed.

## Primary replica

Same as leader/primary + followers/replicas naming varies by vendor (Postgres “primary/standby”, MySQL “source/replica”, Mongo “primary/secondary”).

## WAL (write-ahead log)

Append-only durable log of page changes before applying to data files. Crash recovery: replay from last checkpoint. Also ships bytes to replicas.

## LSN (log sequence number)

Monotonic log position in **WAL** streams; replicas report `flush_lsn`, `replay_lsn` in Postgres for monitoring **replication lag**.

## Transactions

**ACID** bundle of operations: all commit or all abort; isolated from concurrent txs while they run; durable after commit ack.

### Example (Postgres)

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;  -- or ROLLBACK on error
```

### Example (distributed saga — idea)

Instead of one 2PC across vendors, use compensating actions: `ReserveInventory` then `ChargeCard`; if charge fails, `ReleaseInventory`.

## ACID

**Atomicity**: all-or-nothing. **Consistency**: invariants preserved (includes app rules, not only CAP). **Isolation**: concurrent txs see allowed intermediate states per isolation level. **Durability**: committed data survives crash (via **WAL** + fsync policy).

## BASE

**Basically Available**, **Soft state**, **Eventual consistency**: accept temporary inconsistency for uptime/partition survival—common in caches and wide-column stores.

## CAP theorem

When a **network partition** happens, you cannot have both linearizable reads/writes *and* total availability of both sides independently. Real systems pick per-operation latency/consistency (PACELC extends the tradeoff to normal operation too).

## Consistency

Overloaded term: (1) **ACID** invariants, (2) linearizable/serializable distributed reads/writes, (3) **eventual** replica convergence. Always disambiguate in design docs.

## Availability

In CAP sense: non-failing nodes keep answering requests (possibly stale). Different from monthly uptime SLA percentages.

## Partition tolerance

System continues operating despite network splits. CAP assumes partitions occur; the theorem constrains what you can guarantee *during* them.

## Deadlock

Cycle of lock waits. Prevention: global lock ordering `(resource_type, id)`; detection: timeouts + retry with backoff.

### Example (cycle)

```text
Tx1: LOCK A → wants B
Tx2: LOCK B → wants A   → deadlock; DB picks victim to abort
```

## Race condition

Outcome depends on scheduling; classic bug: read-modify-write without **atomic operation** or transaction.

### Example (lost update without protection)

```python
# two workers both read count=10, both write 11 — one increment lost
count = db.get("c")
db.set("c", count + 1)
```

Fix: `UPDATE counters SET v = v + 1 WHERE id = 'c'` or `SELECT ... FOR UPDATE`.

## Atomic operation

Single step visible as indivisible—DB single-statement updates, `CAS` in Redis, hardware compare-and-swap.

### Example (Redis INCR)

```redis
INCR pageviews:home
```

## Replication

Copies for durability and read scale: sync vs async, leader-based vs multi-leader (conflict resolution harder).

## Partitioning

Splitting data by key (**sharding**) so each shard fits on one machine. Not the same as “network partition” in CAP.

## Sharding

Hash(user_id) mod N, or range shards on time/org id. Rebalancing (consistent hashing, moved ranges) is operationally heavy.

## Distributed cache

Shared **Redis**/Memcached tier; watch hot keys, **TTL** stampede, and invalidation strategy (write-through vs cache-aside).

### Example (cache-aside)

```python
def get_user(uid):
    key = f"user:{uid}"
    hit = redis.get(key)
    if hit:
        return json.loads(hit)
    row = db.fetch_user(uid)
    redis.setex(key, 300, json.dumps(row))
    return row
```

## Consensus

Raft/Paxos elect a **leader** and replicate a log in order despite crashes. etcd, Consul, Kafka controller use variants.

## Causal consistency

If event B causally depends on A (same session, explicit happens-before), all observers see A before B—even if not linearizable globally.

## DDIA (Designing Data-Intensive Applications)

Martin Kleppmann’s book—mental models for **replication**, **partitioning**, stream vs batch, and consistency tradeoffs. Often cited as “read DDIA chapter 5” in backend discussions.
