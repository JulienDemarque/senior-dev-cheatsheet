# Messaging & background jobs

Brokers, consumers, **acknowledgement** semantics, and Python **Celery** patterns.

## Message broker

Decouples producers and consumers with durable or transient storage, routing (topics, exchanges), and delivery guarantees (at-most-once, at-least-once, exactly-once within broker limits).

## Pub/Sub

Many subscribers get a copy of each message (fan-out). Good for notifications; pair with per-consumer **queue** for work distribution.

## Queue (messaging)

Competing consumers: each message usually processed once. Contrast with **Queue (data structure)** in algorithms.

## RabbitMQ

AMQP model: **exchange** routes to **queues** via bindings; DLX for **dead letter queue**; TTL and priority queues supported.

### Example (declare queue — pika-style idea)

```python
channel.queue_declare(queue="tasks", durable=True)
channel.basic_publish(exchange="", routing_key="tasks", body=payload, properties=pika.BasicProperties(delivery_mode=2))
```

## Kafka

Partitioned append-only log; ordering per partition; consumer groups coordinate partition assignment; retention by time/size—not a classic task **queue** but great for event sourcing and pipelines.

### Example (kafka-console-producer)

```bash
echo '{"id":1}' | kafka-console-producer --broker-list kafka:9092 --topic orders
```

## Worker

Pull loop: receive message → process → **ack** / **nack**. Scale workers horizontally; watch poison messages filling **DLQ**.

## Background jobs

Offload slow or unreliable work (email, PDF, webhooks) from HTTP handlers; return `202 Accepted` + job id when useful.

### Example (Celery task)

```python
from celery import Celery
app = Celery("proj", broker="redis://localhost:6379/0")

@app.task
def send_welcome_email(user_id: int):
    ...
```

## Celery

Python task queue: **broker** (Redis/RabbitMQ), result backend optional, beat for periodic tasks. Configure `acks_late`, visibility timeout (Redis), and idempotent tasks.

## Broker

Message/task transport + persistence metadata. If broker dies unreplicated, queued work is lost—design HA clusters.

## Dead Letter Queue (DLQ)

After N failures or TTL expiry, route to DLQ for inspection/replay. Stops **poison message** infinite loops.

## Poison message

Always throws (bad schema, bug); fix code or skip with manual intervention; metrics on DLQ depth alert you.

## Retry queue

Delay re-delivery: RabbitMQ TTL+DLX pattern, Kafka separate retry topic with backoff consumer, SQS visibility timeout bump.

## Priority queue

Higher priority messages dequeued first; risk starvation—cap low-priority age or use weighted fair queueing.

## Acknowledgement

Signal success so broker deletes message or commits offset. At-least-once = possible duplicates → **idempotency** downstream.

## Late acknowledgement

Ack **after** processing (safe with prefetch limits). Early ack before work finishes risks message loss on worker crash.

### Example (Celery)

```python
@app.task(acks_late=True)
def flaky_job(x):
    ...
```

## Task executor

Thread pool vs prefork vs **async** workers (`gevent`, `asyncio` experimental)—match pool type to **I/O-bound** vs **CPU-bound** tasks.
