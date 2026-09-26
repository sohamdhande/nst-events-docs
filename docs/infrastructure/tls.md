# TLS & Encryption Configuration

This document outlines the TLS encryption strategy for the `nst-events` platform, ensuring secure, encrypted communication from the end-user to the Kubernetes cluster.

## 1. Verified Issuer (`cert-manager`)

The verified K3s cluster has `cert-manager` installed and actively running. 
A `ClusterIssuer` named `letsencrypt-prod` is already configured and verified to be working across the cluster.

## 2. TLS Architecture

We utilize an **Edge-to-Origin Encrypted Architecture**:

1. **Client to Edge (Cloudflare)**: Cloudflare manages the public-facing edge certificates (Universal SSL) and terminates the initial connection.
2. **Edge to Origin (Cluster)**: Cloudflare establishes a new, strictly encrypted connection to the K3s Traefik ingress using the certificates provisioned by `cert-manager`.

*Note: Cloudflare's SSL/TLS mode must be set to "Full (Strict)" to ensure it validates the `cert-manager` certificates.*

## 3. Certificate Issuance Strategy

Instead of relying on a wildcard certificate (which often requires complex DNS-01 challenges), we will utilize standard HTTP-01 challenges to provision exact certificates for our subdomains.

When deploying the `Ingress` resources for `nst-events`, we simply annotate them for `cert-manager`:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: nst-events-ingress
  annotations:
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
    # Note: Nginx annotations removed
spec:
  tls:
  - hosts:
    - dashboard.nst-events.nstsdc.org
    - api.nst-events.nstsdc.org
    secretName: nst-events-tls
```

Upon application of this manifest, `cert-manager` will automatically negotiate with Let's Encrypt, complete the HTTP-01 challenge via Traefik, and store the resulting certificate in the `nst-events-tls` Secret for Traefik to use.

## 4. HTTP to HTTPS Redirection

* **Cloudflare Layer**: "Always Use HTTPS" should be enabled in the Cloudflare dashboard. This catches unencrypted traffic at the edge and redirects it immediately, preventing unencrypted traffic from ever reaching the cluster.
* **Traefik Layer**: Traefik is natively configured to handle HTTPS traffic on port 443 via the certificates provided by `cert-manager`.
