# Caching, Redis & rate limiting

**Redis** as cache, lock, pub/sub, and rate-limit backend—with CLI and Python-style patterns.

## Redis

Single-threaded command execution (mostly), in-memory with optional persistence (RDB snapshots, AOF). Data structures: strings, hashes, lists, sets, sorted sets, streams, HyperLogLog, bitmaps.

### Example (CLI)

```bash
redis-cli SET session:abc123 '{"user_id":42}' EX 3600
redis-cli GET session:abc123
```

## Redis lock

Correct pattern: `SET lock:resource token NX PX 30000` — only set if absent, with expiry. Unlock with Lua script that deletes **only if** value matches your token (avoid deleting someone else’s lock after slow GC pause).

### Example (set lock)

```redis
SET lock:payments:9 my-uuid-... NX PX 15000
```

Returns `OK` if acquired, `(nil)` if already held.

### Example (safe unlock — Lua idea)

```lua
if redis.call("GET", KEYS[1]) == ARGV[1] then
  return redis.call("DEL", KEYS[1])
else return 0 end
```

## Redis TTL

`EXPIRE`, `SET key val EX seconds`, or `PEXPIRE`. Expiry is probabilistic sampling—do not rely on millisecond precision. Stagger expirations to reduce **thundering herd**.

## Redis pub/sub

`PUBLISH` / `SUBSCRIBE`: no persistence, no ack—subscribers offline miss messages. Use **Redis Streams** or **Kafka** when you need durability.

### Example

```bash
# terminal A
redis-cli SUBSCRIBE alerts

# terminal B
redis-cli PUBLISH alerts "deploy done"
```

## Rate limiter

Enforce N requests per window per key (IP, user, API key). Centralize in **Redis** for distributed apps.

## Fixed window

Count in calendar bucket `2025-05-11T14:05`. Simple; bursts at boundary can do 2× allowed traffic across two windows.

### Example (INCR + EXPIRE — rough fixed window)

```redis
INCR ratelimit:192.0.2.1:minute:202505111405
EXPIRE ratelimit:192.0.2.1:minute:202505111405 60
```

## Sliding window

Use sorted set of timestamps, or Redis Cell module, or approximate algorithms (GCRA). Smoother than **fixed window** at edges.

### Example (sorted set sketch)

```python
# Pseudocode: remove entries older than now - window, ZADD now, ZCARD <= limit
```

## Token bucket

Refill rate `r`, burst capacity `B`. Common in APIs (`x-rate-limit-*` headers). Allows short bursts while bounding average rate.

## Leaky bucket

Smooth output rate (shaping); can queue or drop when queue full—used in network QoS and some gateways.

## TTL (time-to-live)

Universal idea: DNS SOA, cache entries, **JWT** exp, DNS negative cache. Shorter **TTL** → fresher data, more load on origin.
