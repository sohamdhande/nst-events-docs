# Cloudflare Configuration

Cloudflare sits at the edge of the `nst-events` networking topology, acting as the primary DNS provider, WAF (Web Application Firewall), and DDoS mitigation layer.

## 1. DNS & Origin Routing

The `nst-events` platform relies on Cloudflare's proxying capabilities.
* **DNS Resolution**: `A` or `CNAME` records for the proposed domains (e.g., `api.nst-events.nstsdc.org`) must point to the public IPs of the K3s cluster nodes, or be routed through a Cloudflare Tunnel if the cluster is strictly private.
* **Proxy Status**: Must be set to **Proxied (Orange Cloud)** to leverage Cloudflare's edge features and hide the origin server IPs.

## 2. SSL/TLS Mode

> [!IMPORTANT]
> **Full (Strict) Mode Required**
> Cloudflare's SSL/TLS encryption mode must be set to **Full (Strict)**.

Because the K3s cluster automatically provisions valid Let's Encrypt certificates via `cert-manager` for Traefik, Cloudflare must strictly validate the origin certificate. Setting this to "Flexible" will cause infinite redirect loops, and setting it to "Full" without "Strict" negates origin authentication.

## 3. SSE (Server-Sent Events) Compatibility

The `nst-events` platform heavily utilizes Server-Sent Events for realtime updates (e.g., `/v1/notifications/live` and `/v1/events/:id/live`).

### The Connection Timeout Constraint
Cloudflare imposes a strict **100-second idle connection timeout** on all proxy requests. If a connection remains open without data transfer for 100 seconds, Cloudflare will terminate it.

### The Application Solution
The `nst-events` repository correctly addresses this constraint natively. The `sse.router.ts` implementation includes a **30-second heartbeat** (`type: 'heartbeat'`). This guarantees data flows across the connection well within Cloudflare's 100-second window, keeping the SSE streams alive indefinitely.

## 4. Rate Limiting & Proxy Headers

Because all public traffic routes through Cloudflare, every incoming request to the API will appear to originate from Cloudflare's IP addresses.

* **Trusted Proxies**: The Express.js backend must be configured to trust proxy headers.
* **Client IP Resolution**: Rate-limiting middlewares and audit logs must utilize the `CF-Connecting-IP` or `X-Forwarded-For` headers to identify the actual client IP, rather than the Cloudflare edge IP.
