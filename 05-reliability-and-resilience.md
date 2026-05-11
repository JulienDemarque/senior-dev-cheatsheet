# Reliability & resilience

Patterns for **timeout**s, **retry**, backoff, and safe deduplication—so transient faults don’t become outages.

## Timeout

Cap wait time on outbound calls, DB statements, locks, and child processes. Combine with cancellation where supported; log slow buckets for tuning.

### Example (Python `httpx`)

```python
import httpx

with httpx.Client(timeout=httpx.Timeout(5.0, connect=2.0)) as client:
    r = client.get("https://api.example.com/health")
```

### Example (Postgres statement timeout)

```sql
SET statement_timeout = '5s';
SELECT ...;
```

## Retry

Repeat on likely-transient failures (HTTP 429/503, timeouts, broken pipe). Do **not** blindly retry non-idempotent POSTs without **idempotency key** or server-side dedupe.

### Example (simple loop with cap)

```python
import time
import random

def call_with_retries(fn, max_attempts=5):
    delay = 0.1
    for attempt in range(max_attempts):
        try:
            return fn()
        except OSError:  # stand-in for timeouts / reset by peer — narrow to real transient types
            if attempt == max_attempts - 1:
                raise
            time.sleep(delay + random.uniform(0, delay * 0.1))  # jitter
            delay = min(delay * 2, 5.0)
```

## Exponential backoff

Double (or multiply by φ) sleep between attempts so a failing dependency gets breathing room. Cap the max delay.

## Jitter

Randomize delay (`sleep(base * (1 + random()))`) so retries don’t align into spikes (**thundering herd** / **retry storm**).

## Retry storm

Massive synchronized retries after an incident—can keep the system down. Mitigate: **jitter**, per-client budgets, circuit breakers, global “degraded mode” flags.

## Thundering herd

Many clients hit a cold cache or wake together (expired cache, cron). Mitigate: staggered **TTL**s, single-flight (`asyncio.Lock` around first fetch), request coalescing.

### Example (single-flight sketch)

```python
import asyncio
_locks = {}

async def get_or_load(key, loader):
    lock = _locks.setdefault(key, asyncio.Lock())
    async with lock:
        if key in cache:
            return cache[key]
        val = await loader(key)
        cache[key] = val
        return val
```

## Idempotency

`f(f(x)) = f(x)` from the system’s perspective: duplicate delivery does not corrupt state. Implement with natural keys (`UPSERT`), status checks, or **idempotency key** store.

### Example (HTTP PUT)

```http
PUT /users/42 {"name":"Ada"}
# repeating same body leaves user 42 in same final state
```

## Idempotency key

Client sends unique key per logical operation (`Idempotency-Key: uuid` on Stripe-style APIs). Server stores `(key → response)` for 24h+ and replays same response on duplicate.

### Example (header)

```http
POST /payments
Idempotency-Key: 7b291e1b-5c32-4f1a-9d0e-...
Content-Type: application/json

{"amount_cents": 5000, "currency": "usd"}
```

Server: `if redis.setnx("idem:"+key, "processing"): ... else: return cached body`.
