# Networking Topology

This document details the intended network topology for the `nst-events` platform. It defines how traffic flows from the public internet down to individual pods.

## 1. Global Request Flow

All public traffic to the platform follows a strict, encrypted path:

```text
[ Client (Mobile/Web) ]
        │ HTTPS (Port 443)
        ▼
[ Cloudflare ]                 <-- Edge Proxy, DNS, WAF, DDoS Protection
        │ HTTPS (Port 443)
        ▼
[ K3s LoadBalancer ]           <-- Cluster Node IPs
        │
[ Traefik Ingress Controller ] <-- TLS Termination (cert-manager), Host-based Routing
        │
        ├── Rule: Host(`dashboard.nst-events.nstsdc.org`)
        │    └── Service: nst-dashboard-service (Port 80) ──> Dashboard Pod (Port 3000)
        │
        └── Rule: Host(`api.nst-events.nstsdc.org`)
             └── Service: nst-api-service (Port 80) ──> API Pod (Port 3001)
```

## 2. Proposed Environments & Hostnames

> [!NOTE]
> **Pending DNS Confirmation**
> The following hostnames reflect standard practices and align with the existing `*.nstsdc.org` domains on the cluster. However, these remain **proposed** until explicit DNS records or wildcard routing via Cloudflare are formally verified.

### Production (`nst-events-prod`)
* **Dashboard**: `dashboard.nst-events.nstsdc.org`
* **API**: `api.nst-events.nstsdc.org`

### Staging (`nst-events-staging`)
* **Dashboard**: `staging-dashboard.nst-events.nstsdc.org`
* **API**: `staging-api.nst-events.nstsdc.org`

## 3. Internal Kubernetes Networking

Once traffic clears the Traefik Ingress controller, it traverses internal Kubernetes `Service` objects using standard cluster DNS.

### Internal Service Ports
* **Dashboard Service**: Exposes port 80, routing to pod port 3000.
* **API Service**: Exposes port 80, routing to pod port 3001.
* **Worker Service**: Internally accessible (optional, since it primarily pulls from the DB), exposing port 3002.
* **PostgreSQL Service**: Strictly internal, exposing port 5432.

### Database Network Security
The PostgreSQL database is strictly isolated from the public internet. 
- A `NetworkPolicy` defaults to `Deny` for all ingress traffic to the database.
- Explicit ingress rules allow connections **only** from pods labeled `app: nst-api` and `app: nst-worker` over port 5432.

## 4. Failure Troubleshooting Flowchart

If the application is unreachable, diagnose in this exact order:

1. **DNS Failure**: Does `ping api.nst-events.nstsdc.org` resolve to Cloudflare IPs?
2. **Cloudflare Failure**: Do Cloudflare analytics show blocked requests (WAF)? Are the origin servers (Node IPs) reachable from Cloudflare?
3. **TLS Failure**: Is the browser showing an SSL error? Check `k3s kubectl get certificate` to ensure cert-manager successfully provisioned the Let's Encrypt certificate.
4. **Traefik/Ingress Failure**: Check Traefik logs. Ensure the `Ingress` resource has the correct `Host` rules and is not using legacy `nginx` annotations.
5. **Service Failure**: Is the Kubernetes Service correctly mapping to the pod selector? Run `k3s kubectl get endpoints` to verify.
6. **Pod Failure**: Are the API/Dashboard pods crashing? Check `k3s kubectl logs` and `k3s kubectl describe pod`.
7. **Application Failure**: Are the pods running but returning 500s? Check application logs for database connection issues or unhandled exceptions.
