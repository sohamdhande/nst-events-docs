# 01 — Physical Bare-Metal Nodes, OS & Scheduling Topology

This document details the physical hardware, operating system environment, node partitioning strategy, and Kubernetes pod scheduling rules that govern how **NST Events** runs across the college server infrastructure.

---

## 1. Physical Node Inventory

The platform is hosted on a dedicated, on-premise cluster of **seven physical bare-metal machines** situated in the campus server room. All nodes are interconnected on the high-speed campus Local Area Network (LAN).

```text
                               ┌────────────────────────────────────────────────────────┐
                               │                    CAMPUS LOCAL NETWORK                │
                               │                Subnet: 192.168.136.0/20                │
                               │                Gateway: 192.168.128.1                  │
                               └───────────────────────────┬────────────────────────────┘
                                                           │
       ┌───────────────────┬───────────────────┬───────────┴───────┬───────────────────┬───────────────────┐
       ▼                   ▼                   ▼                   ▼                   ▼                   ▼
┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│    nst-n1    │    │    nst-n2    │    │    nst-n3    │    │    nst-n4    │    │    nst-n5    │    │    nst-n6    │ ... nst-n7
│ ControlPlane │    │ Compute/Disk │    │ Compute/Disk │    │ Compute/Disk │    │ Compute/Disk │    │ Heavy Compute│
│192.168.136.145│   │192.168.136.146│   │192.168.136.147│   │192.168.136.148│   │192.168.136.149│   │192.168.136.150│
└──────────────┘    └──────────────┘    └──────────────┘    └──────────────┘    └──────────────┘    └──────────────┘
```

### Complete Hardware Specification Matrix

| Node | Kubernetes Role | Cluster Label | IP Address | MAC Address | CPU Threads | RAM | NVMe Disk | Primary Node Responsibilities |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **`nst-n1`** | Control Plane (Master) | `role=control` | `192.168.136.145` | `8c:ec:4b:79:52:81` | 8 (Intel Core i7) | 16 GB | 233 GB NVMe | K3s API Server, etcd state store, `cloudflared` tunnel, Traefik Ingress Controller, cert-manager, Rancher, Fleet. |
| **`nst-n2`** | Worker (Compute + Storage) | `role=compute` | `192.168.136.146` | `f4:8e:38:82:d2:25` | 8 (Intel Core i7) | 8 GB | 97.9 GB NVMe (24.4 GB free) | Longhorn Storage Node (Replica), compute workload execution. |
| **`nst-n3`** | Worker (Compute + Storage) | `role=compute` | `192.168.136.147` | Campus LAN DHCP/Static | 8 (Intel Core i7) | 8 GB | 97.9 GB NVMe (29.1 GB free) | Longhorn Storage Node (Replica), compute workload execution. |
| **`nst-n4`** | Worker (Compute + Storage) | `role=compute` | `192.168.136.148` | Campus LAN DHCP/Static | 8 (Intel Core i7) | 8 GB | 97.9 GB NVMe (33.9 GB free) | Longhorn Storage Node (Replica), compute workload execution. |
| **`nst-n5`** | Worker (Compute + Storage) | `role=compute` | `192.168.136.149` | Campus LAN DHCP/Static | 8 (Intel Core i7) | 8 GB | 97.9 GB NVMe (30.6 GB free) | Longhorn Storage Node (Replica), compute workload execution. |
| **`nst-n6`** | Worker (Heavy Compute + Storage) | `role=compute` | `192.168.136.150` | Campus LAN DHCP/Static | 8 (Intel Core i7) | 64 GB | 913.3 GB NVMe (108.1 GB free) | **Primary Workload Target**, Longhorn Storage Node (Replica), dedicated GitHub Actions CI/CD runner (`nst-n6-runner`). |
| **`nst-n7`** | Worker (Compute + Storage) | `role=compute` | Campus LAN IP | Campus LAN DHCP/Static | 8 (Intel Core i7) | 16 GB | 97.9 GB NVMe (56.7 GB free) | Longhorn Storage Node (Replica), secondary compute workload node. |

---

## 2. Operating System & Kubernetes Distribution

- **Operating System**: Ubuntu Server 25.10 (Questing Quokka).
- **Linux Kernel**: `6.17.0-generic` across all nodes.
- **Kubernetes Distribution**: **K3s** (`v1.33.6+k3s1`).
- **Container Runtime**: `containerd` (`v2.1.5`).
- **Networking CNI**: Flannel (VXLAN overlay on port 8472).
- **Embedded Components**:
  - **Embedded SQLite / etcd**: K3s control plane state storage on `nst-n1`.
  - **ServiceLB (svclb)**: K3s native daemonset exposing LoadBalancer services across physical node network interfaces.
  - **CoreDNS**: Internal cluster DNS (`cluster.local`) for automatic service discovery.
  - **metrics-server**: Gathers real-time CPU and memory metrics for `kubectl top nodes` and `kubectl top pods`.

---

## 3. Node Roles & Workload Partitioning Strategy

To guarantee platform resilience and prevent application workloads from starving critical control plane processes, the cluster strictly partitions responsibilities:

### A. The Control Plane Guardian (`nst-n1`)
Node `nst-n1` is the **brain and edge gateway** of the cluster. It is the only node that runs:
1. **K3s API Server (`https://192.168.136.145:6443`)**: Handles all `kubectl` interactions and cluster state scheduling.
2. **`cloudflared` Tunnel Daemon**: The single egress tunnel that connects the cluster to the worldwide Cloudflare edge.
3. **Traefik Ingress Controller**: Evaluates HTTP host and path rules for all inbound requests.
4. **cert-manager**: Manages Let's Encrypt TLS certificate lifecycle and renewal.
5. **Rancher & Fleet**: Cluster-level observability and GitOps deployment engines.

> [!IMPORTANT]
> **Isolation Policy: No Application Workloads on `nst-n1`**
> Because `nst-n1` manages cluster-wide ingress and control APIs, student application pods (`nst-api`, `nst-dashboard`, `nst-worker`, `nst-postgres`) are **strictly prohibited** from scheduling onto `nst-n1`. This eliminates noisy-neighbor CPU spikes or out-of-memory (OOM) events from disrupting the API server or ingress routing.

### B. The High-Capacity Primary Node (`nst-n6`)
Node `nst-n6` has distinct hardware advantages:
- **Massive RAM**: 64 GB (compared to 8–16 GB on other nodes).
- **Enormous Storage**: 913 GB NVMe drive (with > 108 GB immediately available for Longhorn block replication and container caching).
- **Dedicated CI/CD Host**: Runs the self-hosted GitHub Actions runner (`nst-n6-runner`). Building Docker containers and caching multi-stage layers on `nst-n6` avoids saturating the campus network uplink.

### C. Compute & Storage Nodes (`nst-n2` to `nst-n5`, `nst-n7`)
These nodes serve as distributed compute and storage workers. Each participates in the Longhorn storage pool and runs student application pods alongside student development environments (e.g., JupyterHub).

---

## 4. Workload Scheduling & Node Affinity Rules

Every workload manifest in the `nst-events` platform defines explicit **NodeAffinity** rules to enforce the scheduling strategy described above:

```yaml
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution:
      nodeSelectorTerms:
        - matchExpressions:
            - key: kubernetes.io/hostname
              operator: NotIn
              values:
                - nst-n1
    preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 80
        preference:
          matchExpressions:
            - key: kubernetes.io/hostname
              operator: In
              values:
                - nst-n6
      - weight: 20
        preference:
          matchExpressions:
            - key: kubernetes.io/hostname
              operator: NotIn
              values:
                - nst-n7
```

### How the Kubernetes Scheduler Interprets This:
1. **Hard Constraint (`requiredDuringScheduling...`)**:
   - `operator: NotIn, values: [nst-n1]`: Under no circumstances will an `nst-events` pod ever be scheduled on `nst-n1`. If only `nst-n1` is available, the pod will remain in `Pending` state rather than compromise the master node.
2. **Soft Preference 1 (`weight: 80`)**:
   - `operator: In, values: [nst-n6]`: The scheduler prioritizes placing pods onto `nst-n6` to take advantage of its 64 GB RAM and fast NVMe drive.
3. **Soft Preference 2 (`weight: 20`)**:
   - `operator: NotIn, values: [nst-n7]`: `nst-n7` is kept as a tertiary backup compute node unless `nst-n2` through `nst-n6` are fully booked.

---

## 5. Hardware Troubleshooting & Operational Discipline

Because the cluster is composed of physical bare-metal hardware on campus:

1. **Node Marked `NotReady`**:
   - In 90% of cases, a `NotReady` node is caused by someone physically unplugging power or Ethernet in the campus server room, or an upstream campus switch rebooting.
   - **Check the physical box first** (LED activity, power switch, network link lights) before attempting to debug K3s services.
2. **Checking Overall Cluster Health**:
   ```bash
   # Inspect all nodes, ready states, roles, and internal IPs
   kubectl get nodes -o wide

   # Check real-time CPU and RAM utilization per node
   kubectl top nodes

   # Check storage status across all Longhorn disks
   kubectl get nodes.longhorn.io -n longhorn-system
   ```
3. **Checking Node Workload Distribution**:
   ```bash
   # List all pods running in the nst-events namespace and their assigned physical node
   kubectl get pods -n nst-events -o wide
   ```
