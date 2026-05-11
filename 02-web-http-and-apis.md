# Web, HTTP & APIs

How browsers and services talk over HTTP: resources, pagination, streaming, and operational patterns—with small examples.

## CRUD

**Create, Read, Update, Delete**—the four persistence operations. Common HTTP mapping (not the only valid design):

| Operation | Typical HTTP | Body |
|-----------|--------------|------|
| Create | `POST /items` | new resource |
| Read | `GET /items/:id` | — |
| Update | `PUT` (replace) or `PATCH` (partial) | changed fields |
| Delete | `DELETE /items/:id` | — |

### Example (curl)

```bash
curl -X POST https://api.example.com/items -H 'Content-Type: application/json' -d '{"name":"draft"}'
curl https://api.example.com/items/1
```

## REST API

**Representational State Transfer**: resource-oriented URLs, standard verbs, stateless requests, meaningful status codes (`409 Conflict`, `422 Unprocessable`). Hypermedia (HATEOAS) is rare in practice; many “REST” APIs are pragmatically RPC-shaped subpaths under resource trees.

### Example (resource naming)

`GET /users/42/orders` — orders scoped under user `42` (clear ownership); avoid `GET /getUserOrders?id=42` when you want RESTful URLs.

## Endpoint

Concrete `(method, path)` surface, optionally with route params: `PATCH /v1/invoices/:id`. Gateways may add path prefixes (`/api/...`) and strip them before **reverse proxy** to upstream.

## Pagination

Splitting long lists into pages. Choose **offset** vs **cursor** based on stability and SQL cost.

## Offset pagination

`LIMIT` + `OFFSET` (or `page` × `page_size`). Easy for UIs with jump-to-page; **unstable** if rows shift between requests; **slow** for large offsets (DB still walks skipped rows).

### Example (SQL)

```sql
SELECT id, title FROM posts
ORDER BY created_at DESC
LIMIT 20 OFFSET 40;  -- "page 3" if size=20
```

## Cursor pagination

Client sends opaque `cursor` from previous response (often `(sort_value, id)` tuple encoded). Stable when rows insert/delete; scalable for “infinite scroll”.

### Example (pattern)

```sql
-- After cursor (created_at, id) = ('2024-01-15', 1001)
SELECT id, title, created_at FROM posts
WHERE (created_at, id) < ('2024-01-15', 1001)
ORDER BY created_at DESC, id DESC
LIMIT 20;
```

API: `GET /posts?cursor=eyJjIjo...` returning `{ "items": [...], "next_cursor": "..." }`.

## Validation

Layered checks: wire format → schema (JSON Schema, pydantic) → domain rules (balance sufficient, slug unique). Fail fast with stable error codes for clients.

### Example (pydantic sketch)

```python
from pydantic import BaseModel, Field

class CreateItem(BaseModel):
    name: str = Field(min_length=1, max_length=200)
    qty: int = Field(ge=1)
```

## Middleware

Plugs into the server pipeline: runs before/after your route handler. Order matters (e.g. request ID first, auth before business logic, metrics last).

### Example (Starlette-style idea)

```python
# Pseudocode: ASGI middleware wraps inner app
async def middleware(scope, receive, send):
    if scope["type"] == "http":
        # add header, log, reject if unauthenticated, ...
        ...
    await inner_app(scope, receive, send)
```

## Webhook

Inbound HTTP **POST** from a provider when something happens. You must verify signatures (HMAC of raw body with shared secret), respond `2xx` quickly, and process asynchronously for heavy work.

### Example (verify HMAC — pattern)

```python
import hmac, hashlib

def verify(payload: bytes, signature: str, secret: bytes) -> bool:
    mac = hmac.new(secret, payload, hashlib.sha256).hexdigest()
    return hmac.compare_digest(mac, signature)
```

Use **idempotency** (`event_id` dedupe store) because providers **retry** on non-2xx.

## Polling

`setInterval` / cron repeatedly calling `GET /status`. Simple; wastes bandwidth and adds latency vs **SSE**/**WebSocket** when updates are frequent.

### Example (bash)

```bash
while sleep 2; do curl -s https://api.example.com/builds/9/status; done
```

## WebSocket

Upgrade from HTTP to bidirectional framed messages (`ws://` / `wss://`). Good for collaborative editors, games, streaming RPC. You must handle auth (often cookie on upgrade or token in query—tradeoffs), heartbeats, and horizontal scale (often **Redis** or dedicated gateway for pub/sub across nodes).

### Example (browser)

```javascript
const ws = new WebSocket("wss://chat.example.com/room/1");
ws.onmessage = (ev) => console.log(JSON.parse(ev.data));
ws.send(JSON.stringify({ type: "msg", body: "hi" }));
```

## SSE (server-sent events)

One-way server→client over HTTP. Response `Content-Type: text/event-stream`; events separated by double newlines. Browser **`EventSource`** auto-reconnects.

### Example (minimal event format)

```
data: {"price": 12.34}

data: {"price": 12.35}

```

### See also

[EventSource](./12-python-web-stack.md#eventsource) in the Python/browser stack file.

## Streaming

Send body incrementally so time-to-first-byte is low and RAM stays bounded. Pairs with **chunked** transfer in HTTP/1.1 or data frames in HTTP/2.

## Chunked transfer encoding

Body as `length + chunk` sequences ending with zero-length chunk. Lets you stream without knowing `Content-Length` upfront.

## Backpressure

Slow consumer must slow the producer (bounded queue, `await writable()`, drop policy, or HTTP/2 flow control). Without it, buffers grow until OOM.

### Example (async queue with max size)

```python
import asyncio
q = asyncio.Queue(maxsize=100)
# await q.put(x) blocks when full → natural backpressure
```

## Realtime systems

Latency-sensitive UX or control paths: combine **WebSocket**/**SSE**, push gateways, or **Redis pub/sub** for fan-out. **Read-after-write** consistency matters when users expect their action to appear instantly.

## Signed URL

URL includes crypto parameters (signature, expiry, optional IP restriction) so the holder can perform one operation without ambient credentials.

## S3 presigned URL

AWS SDK builds a URL your frontend or partner uses to `PUT`/`GET` directly to S3—your API only signs, bytes bypass app servers.

### Example (boto3 idea)

```python
import boto3
s3 = boto3.client("s3")
url = s3.generate_presigned_url(
    "put_object",
    Params={"Bucket": "my-bucket", "Key": "uploads/x.pdf", "ContentType": "application/pdf"},
    ExpiresIn=3600,
)
# client PUTs file to `url`
```

## GCS signed URL

Same pattern for Google Cloud Storage: service account signs `GET`/`PUT` URL with `Expires` and signature query params.

## Upload proxying

File bytes flow: client → your app → object storage. Lets you virus-scan, resize, or auth in one place; costs bandwidth/CPU on app tier. Direct upload via **signed URL** offloads bytes.
