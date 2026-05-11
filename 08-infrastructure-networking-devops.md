# Infrastructure, networking & DevOps

Edge, mesh, containers, and GitOps—patterns and minimal config snippets.

## API Gateway

Single entry for TLS, auth, **rate limiting**, routing, A/B splits, request/response transforms. Avoid putting business logic that belongs in domain services.

### Example (conceptual routes)

```yaml
# Pseudocode OpenAPI-style
/r/public/*   → service-public
/r/b/*       → service-b  # JWT validated at gateway
```

## Reverse proxy

Terminates client TLS, HTTP/2, buffering; forwards to upstream pools. **NGINX** `proxy_pass`, **Envoy** clusters.

### Example (NGINX fragment)

```nginx
location /api/ {
    proxy_pass http://api_upstream;
    proxy_set_header Host $host;
    proxy_set_header X-Request-Id $request_id;
}
```

## Load balancer

L4 (TCP) vs L7 (HTTP routing). Health checks (`/healthz`), connection draining, optional session stickiness.

## CDN

Edge caches static assets; **Cloudflare**/Fastly/Akamai also do WAF, DDoS scrubbing, edge workers. Cache-Control headers control behavior.

### Example (cache header)

```http
Cache-Control: public, max-age=31536000, immutable
```

## WAF

Rules for signatures, geo, bot scores, rate limits—sits before origin. Tune false positives; log blocked IDs for tuning.

## NGINX

Worker processes + event loop; huge install base for static files and **reverse proxy**. Often paired with **Certbot** for TLS.

## Envoy

xDS APIs for dynamic config; sidecar in **service mesh**; rich observability filters (access logs, tracing, RBAC at L7).

## Cloudflare

DNS + CDN + WAF + Workers. “Orange cloud” proxies traffic through their edge; understand bypass rules for debugging.

## Kubernetes (K8s)

Control plane schedules **pods** onto nodes; Declarative manifests (`Deployment`, `Service`, `Ingress`). Operators watch CRDs.

### Example (Deployment fragment)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api
spec:
  replicas: 3
  selector:
    matchLabels: { app: api }
  template:
    metadata:
      labels: { app: api }
    spec:
      containers:
        - name: api
          image: ghcr.io/org/api:1.2.3
          ports: [{ containerPort: 8000 }]
```

## Pod

Shared network namespace (one IP), optional volumes, optional init containers. Prefer one main container per pod unless tight sidecar (logging, mesh).

## Docker

`Dockerfile` builds image layers; `docker run` / `compose` for local. In prod, often built in CI and deployed to K8s without Docker daemon on nodes (containerd/CRI-O).

### Example (Dockerfile sketch)

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
CMD ["uvicorn", "app:app", "--host", "0.0.0.0", "--port", "8000"]
```

## Helm

Charts parameterize K8s YAML (`values.yaml`); `helm upgrade --install` manages releases and rollbacks.

## ArgoCD

Continuous GitOps: cluster state tracks git branch; drift reconciled automatically; supports app-of-apps pattern.

## Service mesh

Data plane sidecars intercept all pod traffic: mTLS identity, retries, timeouts, traffic split, metrics. Control plane (Istio, Linkerd) programs **Envoy** (Istio) or proprietary proxy.

## Stateless services

Any replica can handle any request if auth + data come from stores—ideal for **horizontal scaling** and fast rollouts.
