# Python web stack

**FastAPI**, **Django**, servers (**Uvicorn**, **Gunicorn**), and **ASGI** vs **WSGI**—with runnable-ish snippets.

## FastAPI

Type hints + pydantic v2 models → automatic **validation** and OpenAPI. Native async routes; dependency injection for DB sessions and auth.

### Example

```python
from fastapi import FastAPI
from pydantic import BaseModel

app = FastAPI()

class Item(BaseModel):
    name: str
    price: float

@app.post("/items")
async def create_item(item: Item):
    return {"saved": item.model_dump()}
```

## Django

Batteries included: ORM, migrations, admin, auth, forms. Run async views on ASGI server; still watch blocking ORM in async paths—use `sync_to_async` or async-native drivers thoughtfully.

## Django Ninja

Typed API layer on Django with pydantic-like schemas—closer to **FastAPI** ergonomics while keeping Django’s ORM and admin.

## Uvicorn

ASGI server (`uvicorn app:app --workers 4`). Workers are processes; each runs an event loop. Good default for **FastAPI**.

## Gunicorn

Process manager for **WSGI**; for ASGI apps use `uvicorn.workers.UvicornWorker` as worker class.

### Example (Gunicorn + Uvicorn worker)

```bash
gunicorn app:app -k uvicorn.workers.UvicornWorker -w 4 -b 0.0.0.0:8000
```

## ASGI

Async callable `(scope, receive, send)`; supports HTTP, **WebSocket**, lifespan. Starlette/FastAPI built on it.

## WSGI

Callable `environ, start_response` — synchronous request/response. Blocking worker model unless greenlets/eventlet patches (legacy patterns).

## StreamingResponse

Stream bytes from sync iterator, async iterator, or file-like object—good for CSV export or **SSE**.

### Example (SSE-ish lines)

```python
from fastapi.responses import StreamingResponse

async def gen():
    for i in range(3):
        yield f"data: {i}\n\n"

@app.get("/stream")
async def stream():
    return StreamingResponse(gen(), media_type="text/event-stream")
```

## Async generator

`async def` with `yield` — Starlette/FastAPI accept these for streaming bodies.

## EventSource

Browser API: `new EventSource("/events")` — reconnects automatically; server must set correct media type and flush periodically.

## Psycopg2

Blocking Postgres driver; fine under **Gunicorn** sync workers. For asyncio stacks prefer **asyncpg** or Psycopg 3 async mode.

### Example (parameterized query)

```python
cur.execute("SELECT * FROM users WHERE email = %s", (email,))
```

## ORM

Maps tables ↔ objects; migrations track schema. Escape hatch: `raw()` / `text()` still needs parameter binding to avoid **SQL injection**.

## PostgreSQL

Row-level locking (`FOR UPDATE`), JSONB, partial indexes, extensions. Common HA: streaming replica + Patroni/etcd.

### Example (advisory lock — pattern)

```sql
SELECT pg_advisory_lock(hashtext('job:42'));
-- do critical section
SELECT pg_advisory_unlock(hashtext('job:42'));
```
