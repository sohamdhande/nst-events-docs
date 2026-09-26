# Cluster Capabilities

This document details the verified capabilities of the college cluster that hosts the `nst-events` platform. 

> [!IMPORTANT]
> **Source of Truth**
> The information in this document is based on a verified read-only audit of the live cluster. It overrides any conflicting assumptions made in the repository's YAML files.

## 1. Verified Cluster Hardware & OS
* **Architecture**: 7-node Kubernetes cluster (`x86_64`).
* **OS**: Ubuntu 25.10 (Questing Quokka).
* **CPU Capacity**: Primary nodes feature 8 threads (Intel Core i7).
* **RAM Capacity**: Nodes range from ~16GB to ~64GB RAM, providing abundant headroom for `nst-events` staging and production workloads.
* **Storage**: Primary nodes have significant NVMe storage available for persistent volumes.

## 2. Kubernetes Configuration
* **Distribution**: K3s (versions spanning `v1.33` to `v1.36`).
* **Container Runtime**: `containerd`.
* **Networking**: Flannel/CNI for internal pod-to-pod networking.

## 3. Verified Cluster Services
The following services are actively running on the cluster and should be leveraged by `nst-events`:

* **Ingress Controller**: **Traefik** is the active ingress controller. It is exposed via a LoadBalancer service across the node IPs.
* **TLS Management**: `cert-manager` is installed and actively managing a `letsencrypt-prod` ClusterIssuer.
* **Persistent Storage**: **Longhorn** is installed and configured as a storage class with 3-way default replication.
* **Metrics**: `metrics-server` is running, allowing for CPU/RAM utilization tracking and potential Horizontal Pod Autoscaling (HPA).
* **GitOps**: Rancher Fleet (`cattle-fleet-system`) is present on the cluster.

## 4. Public Routing & DNS
* The cluster is accessible via Cloudflare.
* Public HTTPS traffic is actively flowing to existing applications on the cluster via `*.nstsdc.org` domains.
* **Routing Mechanism**: DNS records on Cloudflare map to the cluster. Traefik receives the traffic and routes it to the corresponding pods based on the HTTP Host header.

## 5. Security & Isolation
* Standard Kubernetes RBAC is enforced.
* The cluster supports strict namespace isolation, allowing us to safely run `nst-events-staging` and `nst-events-prod` alongside existing college workloads without interference.
