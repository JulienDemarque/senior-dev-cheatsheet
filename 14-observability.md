# Observability

Logs, metrics, traces, and **audit logs**—what to emit and how agents consume them.

## Observability

Three pillars: **metrics** (aggregates), **logs** (discrete events), **traces** (request DAG across services). High cardinality labels explode cost—design intentionally.

### Example (structured log line — JSON)

```json
{"ts":"2026-05-11T12:00:01Z","level":"info","msg":"order_paid","order_id":"o9","trace_id":"4bf92f3577b34da6a3ce929d0e0e4736"}
```

### Example (Prometheus metric exposition — idea)

```text
http_requests_total{method="GET",path="/healthz"} 142
```

## Audit logs

Immutable append-only security trail: actor, action, object, before/after hash, IP, request id. Separate retention/access class from debug logs.

### Example (fields checklist)

```text
who (user_id / service_principal)
what (action verb + resource type/id)
when (UTC timestamp)
where (IP, geo coarse)
result (allow/deny)
request_id / trace_id for correlation
```

## Datadog

SaaS APM/logs/metrics/RUM; agents on VMs/K8s ship data. Watch ingestion costs and cardinality (`user_id` as tag = expensive).

## ddtrace

Auto-instrumentation for Django/**FastAPI**/Celery/Redis; creates spans with tags. Enable trace context injection (`traceparent`) for cross-service correlation.

### Example (env wiring)

```bash
export DD_SERVICE=api
export DD_ENV=prod
export DD_VERSION=1.4.2
ddtrace-run uvicorn app:app --host 0.0.0.0 --port 8000
```

### See also

[Reliability & resilience](./05-reliability-and-resilience.md) for what to log around retries and **idempotency** keys.
