# 02 — Edge Ingress, Cloudflare Tunnel, TLS & Traffic Flow

This document details how external network requests from student mobile phones, laptops, and administrative dashboards navigate the multi-tier networking topology to reach individual application pods running on the bare-metal cluster.

---

## 1. The Campus Edge Problem & The Tunnel Solution

### The Challenge of Campus Networks
Hosting enterprise-grade applications on physical campus servers presents three fundamental obstacles:
1. **No Static Public IP**: The campus Internet Service Provider (ISP) assigns dynamic IPs or places the campus behind Carrier-Grade NAT (CGNAT).
2. **Strict Firewall & Blocked Ports**: Institutional IT security prohibits inbound port-forwarding (ports 80 and 443) through the campus gateway router.
3. **DDoS & Vulnerability Exposure**: Opening public router ports directly exposes internal campus machines to brute-force scans and volumetric DDoS attacks.

### The Solution: Cloudflare Tunnel (`cloudflared`)
Instead of accepting inbound connections from the internet, the cluster **connects outward** to Cloudflare's global edge network via a persistent, encrypted tunnel:

```text
                                       PUBLIC INTERNET
                                              │
                                              │ HTTPS (Port 443)
                                              ▼
┌───────────────────────────────────────────────────────────────────────────────────────────┐
│                                  CLOUDFLARE GLOBAL EDGE                                   │
│  • Edge TLS Termination (Universal SSL)                                                   │
│  • DDoS Mitigation & Cloudflare WAF                                                       │
│  • DNS: events.nstsdc.org / api.nst-events.nstsdc.org                                      │
└─────────────────────────────────────────────┬─────────────────────────────────────────────┘
                                              │ Persistent Outbound Tunnel
                                              │ (QUIC / TLS Protocol)
                                              ▼
┌───────────────────────────────────────────────────────────────────────────────────────────┐
│ CAMPUS GATEWAY FIREWALL (Zero Inbound Ports Open)                                         │
└─────────────────────────────────────────────┬─────────────────────────────────────────────┘
                                              │
                                              ▼
┌───────────────────────────────────────────────────────────────────────────────────────────┐
│ CONTROL PLANE NODE: nst-n1 (192.168.136.145)                                              │
│                                                                                           │
│  ┌──────────────────────────────────────────────┐                                         │
│  │ cloudflared Systemd Daemon                   │                                         │
│  │ Tunnel Name: krushn-node1                    │                                         │
│  │ Local Forwarding:                            │                                         │
│  │   *.nstsdc.org ──> http://localhost:80       │                                         │
│  └──────────────────────┬───────────────────────┘                                         │
│                         │                                                                 │
│                         ▼ (Port 80)                                                       │
│  ┌──────────────────────────────────────────────┐                                         │
│  │ Traefik Ingress Controller (K3s)             │                                         │
│  │ Host & Path-Based Routing Engine             │                                         │
│  └──────────────────────┬───────────────────────┘                                         │
└─────────────────────────┼─────────────────────────────────────────────────────────────────┘
                          │ Flannel CNI Pod Overlay Network
                          ▼
            [ Internal Kubernetes Services ]
```

### Key Advantages of This Model:
- **No Open Inbound Ports**: The campus firewall drops 100% of incoming unsolicited packets. Only established outbound egress from `cloudflared` is permitted.
- **Zero IP Disclosure**: Attackers cannot discover the physical IP addresses (`192.168.136.145`) of the campus machines; DNS points exclusively to Cloudflare Anycast IPs.
- **Always-On Edge Security**: All traffic passes through Cloudflare's Web Application Firewall (WAF) before it can touch a campus node.

---

## 2. Cloudflare Tunnel Configuration (`krushn-node1`)

The tunnel daemon runs on node `nst-n1` as a systemd service (`cloudflared.service`). Its configuration file resides at `/etc/cloudflared/config.yml`:

```yaml
tunnel: krushn-node1
credentials-file: /etc/cloudflared/<TUNNEL_UUID>.json
origincert: /etc/cloudflared/cert.pem

ingress:
  # 1. Dedicated SSH access (Exact match evaluated first)
  - hostname: "nst-n1.nstsdc.org"
    service: ssh://localhost:22

  # 2. Rancher Cluster UI (HTTPS with origin certificate verification bypass)
  - hostname: "rancher.nstsdc.org"
    service: https://localhost:443
    originRequest:
      noTLSVerify: true

  # 3. Wildcard Catch-All for Applications -> Traefik Ingress
  - hostname: "*.nstsdc.org"
    service: http://localhost:80

  # 4. Mandatory Catch-All (Closes unmatched requests with 404)
  - service: http_status:404
```

> [!IMPORTANT]
> **Ingress Rule Ordering Rule**
> Cloudflare Tunnel evaluates rules **strictly top-to-bottom**. Exact hostnames (like `nst-n1.nstsdc.org` for SSH) must precede wildcard rules (`*.nstsdc.org`). If the wildcard appears first, SSH traffic is erroneously routed to Traefik's HTTP port, breaking remote terminal access.

---

## 3. TLS Encryption Architecture

The platform implements an **Edge-to-Origin Encrypted Architecture**:

1. **Client to Edge**: Users connect to `https://events.nstsdc.org` via HTTPS over port 443. Cloudflare terminates this connection using automated Universal SSL edge certificates.
2. **Edge to Tunnel**: Traffic travels through the encrypted Cloudflare Tunnel protocol directly to `nst-n1`.
3. **Internal Cluster TLS**: `cert-manager` runs inside the K3s cluster and maintains a `ClusterIssuer` named `letsencrypt-prod`. It negotiates with Let's Encrypt via HTTP-01 challenges through Traefik, storing certificates in Kubernetes secrets.
4. **Cloudflare SSL Mode**: Configured to **Full (Strict)** in the Cloudflare Dashboard. This guarantees Cloudflare strictly validates the origin certificate issued by `cert-manager`, preventing man-in-the-middle attacks on the connection.

---

## 4. Traefik Ingress & Path Routing Architecture

When `cloudflared` forwards HTTP requests to `localhost:80`, the **Traefik Ingress Controller** intercepts them and applies the routing rules defined in `infrastructure/kubernetes/ingress.yaml`:

```text
Inbound Request: https://events.nstsdc.org/v1/events
                      │
                      ▼
 ┌─────────────────────────────────────────────────────────┐
 │ Traefik Ingress Controller (Host: events.nstsdc.org)    │
 └────────────────────────────┬────────────────────────────┘
                              │
     ┌────────────────────────┴────────────────────────┐
     │ PathPrefix Match:                               │ PathPrefix Match:
     │  • /health                                      │  • / (Catch-All UI)
     │  • /ready                                       │
     │  • /auth                                        │
     │  • /v1                                          │
     │  • /webhooks                                    │
     ▼                                                 ▼
┌───────────────────────────┐                     ┌───────────────────────────┐
│ Service: nst-api-service  │                     │ Service:                  │
│ Port: 80 ──> Pod: 3001    │                     │  nst-dashboard-service    │
└───────────────────────────┘                     │ Port: 80 ──> Pod: 3000    │
                                                  └───────────────────────────┘
```

### Complete Ingress Routing Matrix

| Ingress Host | Path Prefix | Target Kubernetes Service | Target Container Port | Target Pod Workload | Purpose |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `events.nstsdc.org` | `/health` | `nst-api-service` | 3001 | `nst-api` | Cluster health check endpoint. |
| `events.nstsdc.org` | `/ready` | `nst-api-service` | 3001 | `nst-api` | Pod readiness check (validates database connection). |
| `events.nstsdc.org` | `/auth` | `nst-api-service` | 3001 | `nst-api` | Google OAuth initiation, callback, and session tokens. |
| `events.nstsdc.org` | `/v1` | `nst-api-service` | 3001 | `nst-api` | Core REST API and live SSE streaming channels. |
| `events.nstsdc.org` | `/webhooks` | `nst-api-service` | 3001 | `nst-api` | Inbound webhooks from third-party services. |
| `events.nstsdc.org` | `/` | `nst-dashboard-service` | 3000 | `nst-dashboard` | Next.js App Router serving all web pages. |
| `api.nst-events.nstsdc.org` | `/` | `nst-api-service` | 3001 | `nst-api` | Direct API access endpoint for mobile client and external tools. |

---

## 5. Solving the `/clubs` and `/users` Routing Collision

A critical design challenge in unified web-and-API architectures is **path namespace collisions**:
- The Next.js dashboard provides browser pages: `/clubs`, `/clubs/[id]`, and `/users`.
- The Express API provides REST endpoints: `/clubs`, `/clubs/:id`, and `/users/me`.

If Traefik routed all `/clubs` requests to the API, opening `https://events.nstsdc.org/clubs` in a browser would return `401 Unauthorized` JSON instead of rendering the web directory.

### The Solution: Next.js Context-Aware Reverse Proxy
1. **Traefik Level**: Traefik delegates all non-`/v1` traffic under `/` to `nst-dashboard-service`.
2. **Next.js Rewrite Engine (`next.config.ts`)**:
   - When a student navigates in their browser, Next.js renders the React App Router page.
   - When the dashboard's client-side `apiClient` executes a `fetch()` request, it attaches the HTTP header:
     ```http
     X-Client-Platform: web
     ```
   - Next.js evaluates `beforeFiles` rewrites:
     ```typescript
     {
       source: '/clubs/:path*',
       has: [{ type: 'header', key: 'x-client-platform' }],
       destination: 'http://nst-api-service/clubs/:path*'
     }
     ```
   - Any request carrying `x-client-platform` is transparently proxied to the backend `nst-api-service`. Browser page visits fall through cleanly and render the UI.

---

## 6. Server-Sent Events (SSE) & Cloudflare Timeout Resilience

The mobile app and dashboard subscribe to live real-time updates:
- `/v1/notifications/live`: User-specific real-time notifications.
- `/v1/events/:id/live`: Real-time attendee counters, waitlist updates, and live event announcements.

### The 100-Second Cloudflare Constraint
Cloudflare enforces a hard **100-second idle connection timeout** on all proxy connections. If no byte is transmitted across an open HTTP connection for 100 seconds, Cloudflare terminates the TCP socket.

### The Application Heartbeat Solution
To ensure persistent connections remain open indefinitely without dropping:
1. **Periodic Heartbeat**: `apps/api/src/modules/sse/sse.router.ts` fires an automatic heartbeat comment every **30 seconds**:
   ```json
   data: {"type":"heartbeat","payload":{"timestamp":"2026-09-10T07:15:00.000Z"}}
   ```
   Because data flows every 30 seconds, Cloudflare's 100s idle timer is continuously reset.
2. **Disabling Proxy Buffering**: The API explicitly returns:
   ```http
   X-Accel-Buffering: no
   Cache-Control: no-cache, no-transform
   Connection: keep-alive
   ```
   This prevents Traefik or Cloudflare from buffering chunks, ensuring live event updates render instantaneously on student devices.

---

## 7. Multi-Hop IP Spoofing Prevention (`trust proxy`)

Because requests travel through multiple proxy hops (Client -> Cloudflare Edge -> `cloudflared` on `nst-n1` -> Traefik -> `nst-api`), Express will see `127.0.0.1` or internal pod IPs if unconfigured.

However, naively setting `app.set('trust proxy', true)` introduces severe security risks: an attacker could inject forged `X-Forwarded-For` headers to bypass rate limits or spoof audit log origins.

### Hardened CIDR Whitelist
`apps/api/src/app.ts` configures `trust proxy` with strict internal subnets:
```typescript
app.set('trust proxy', [
  'loopback',
  'linklocal',
  'uniquelocal',
  '10.0.0.0/8',
  '172.16.0.0/12',
  '192.168.0.0/16'
]);
```
Express walks the `X-Forwarded-For` header right-to-left, strips trusted internal cluster and campus IP addresses, and correctly resolves the authentic client IP (originating from `CF-Connecting-IP`). This ensures rate limiters and audit trails record the actual student IP address.
