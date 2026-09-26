# Traefik Ingress Controller

Traefik acts as the native Kubernetes Ingress Controller for the verified K3s cluster hosting `nst-events`. It is responsible for accepting inbound traffic from Cloudflare and routing it to the correct internal Kubernetes Services.

## 1. Repository Discrepancy

> [!WARNING]
> **Incorrect Ingress Annotations in Repository**
> The existing `infrastructure/kubernetes/ingress.yaml` file in the `nst-events` repository uses Nginx annotations (`nginx.ingress.kubernetes.io/rewrite-target`). 
> 
> **Verified Cluster Reality:** The cluster runs **Traefik**, not Nginx. The final deployment manifests must remove all Nginx-specific annotations.

## 2. Ingress Strategy

We will utilize standard Kubernetes `Ingress` resources rather than Traefik-specific `IngressRoute` CRDs. This ensures broader compatibility and simplifies integration with `cert-manager`.

### Host-Based Routing
Traefik evaluates the HTTP `Host` header provided by Cloudflare to determine the routing path:
* Requests with `Host: dashboard.nst-events.nstsdc.org` route to the Dashboard Service.
* Requests with `Host: api.nst-events.nstsdc.org` route to the API Service.

## 3. SSE (Server-Sent Events) Proxy Behavior

By default, reverse proxies often buffer responses. For Server-Sent Events, buffering prevents realtime data from reaching the client immediately.

### The Application Solution
The `nst-events` API explicitly sends the `X-Accel-Buffering: no` header in its SSE responses (`sse.router.ts`). 

While this header is primarily recognized by Nginx, Traefik natively handles chunked encoding and Server-Sent Events gracefully out-of-the-box without requiring aggressive buffering disables. The application's explicit header provides defense-in-depth in case an intermediate proxy is introduced.

## 4. Multi-Replica SSE Routing (Pending Audit)

> [!CAUTION]
> **State Consistency Warning**
> While the SSE keep-alive is resolved at the application level (30s heartbeat), a critical architectural check remains.
>
> If the API is deployed with **2+ replicas** (as intended for HA), Traefik will load-balance requests across the pods. When a client reconnects to the SSE stream (e.g., due to a network drop), they may hit a different API pod than their original connection. 
> 
> **Pending Audit**: The upcoming Worker/Database audit must confirm whether the backend architecture natively handles this via `pgmq` or PostgreSQL `LISTEN/NOTIFY`, ensuring events broadcast to all API pods regardless of which pod the client connects to.
