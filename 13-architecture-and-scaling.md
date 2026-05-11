# Architecture & scaling

How to grow systems: **horizontal scaling**, service boundaries, realtime edges, and state.

## Horizontal scaling

Add instances behind a **load balancer**; requires **stateless** app tier, externalized **session**, and data tier that can **shard** or read-scale.

### Example (HPA idea — Kubernetes)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: api
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: api
  minReplicas: 2
  maxReplicas: 20
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
```

## Vertical scaling

Bigger machine: fewer moving parts until you hit CPU/RAM ceiling or blast radius (one host dies = big outage).

## Microservices

Split by bounded context (billing vs inventory); independent deploys; network failures become normal—need **observability**, contract tests, and graceful degradation.

## Event-driven architecture

Services publish domain events to **Kafka**/bus; consumers react asynchronously. Watch ordering (per-partition keys), duplicate delivery, and schema evolution (Avro/Protobuf/JSON schema registry).

## Pub/Sub layer

Organization-wide event backbone vs ad-hoc **Redis pub/sub** per team—governance, ACLs, retention, replay matter at org scale.

## Service orchestration

Long-running sagas across services (Temporal, Cadence, AWS Step Functions): durable timers, retries, compensations—not the same as **Kubernetes** scheduling.

## Real-time updates

Choose transport: **SSE** (one-way, HTTP-friendly), **WebSocket** (bidirectional), or **polling** fallback. Authenticate at connection time.

## Websocket gateway

Dedicated tier (Socket.IO adapter with **Redis**, custom Go gateway, cloud managed websockets) to fan messages across many API nodes.

## Shared state

Centralize in DB/cache/object store; avoid hidden mutable singletons in app RAM across replicas.

## Stateful services

Sticky file writes, local embedded DB, or GPU session affinity—harder to migrate; prefer volumes + single leader or redesign toward external state.

## Ephemeral workers

Spot instances, serverless invocations: code must checkpoint to durable store; expect SIGTERM mid-job.

### Example (K8s preStop hook idea)

```yaml
lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 5"]
```

Drains connections before pod removal when combined with Service `publishNotReadyAddresses` tuning.
