# RBAC Orphan Cleanup — 2026-09-28

**Date:** 2026-09-28T17:55:49+05:30  
**Operator:** soham (via Antigravity)  
**Scope:** Delete two orphaned ServiceAccounts and their cluster-admin ClusterRoleBindings  
**Authorization:** Explicit user approval after read-only dependency analysis ([SA_DEPENDENCY_ANALYSIS.md](file:///Users/sohamdhande/Docs_Local/nst-events/SA_DEPENDENCY_ANALYSIS.md))

---

## Resources Deleted

| # | Kind | Name | Namespace |
|---|---|---|---|
| 1 | ClusterRoleBinding | `kernel-service-account` | (cluster-scoped) |
| 2 | ServiceAccount | `kernel-service-account` | `jupyterhub-ns` |
| 3 | ClusterRoleBinding | `cluster-admin-sa-binding` | (cluster-scoped) |
| 4 | ServiceAccount | `cluster-admin-sa` | `kube-system` |

---

## Pre-Deletion Dependency Verification

### `jupyterhub-ns/kernel-service-account`

| Check | Result |
|---|---|
| Pods referencing SA | **None** |
| Deployments referencing SA | **None** |
| StatefulSets referencing SA | **None** |
| DaemonSets referencing SA | **None** |
| Jobs referencing SA | **None** |
| CronJobs referencing SA | **None** |
| RoleBindings referencing SA | **None** |
| ClusterRoleBindings referencing SA | `kernel-service-account` → `cluster-admin` (the one being deleted) |
| ownerReferences | **None** |
| Helm releases in namespace | **None** |
| JupyterHub spawner config | Uses SA `hub`, **not** `kernel-service-account` |
| Long-lived Secret token | **None** |
| Running pods in jupyterhub-ns | **None** (namespace idle) |

### `kube-system/cluster-admin-sa`

| Check                              | Result                                                               |
| ------------------------------------| ----------------------------------------------------------------------|
| Pods referencing SA                | **None**                                                             |
| Deployments referencing SA         | **None**                                                             |
| StatefulSets referencing SA        | **None**                                                             |
| DaemonSets referencing SA          | **None**                                                             |
| Jobs referencing SA                | **None**                                                             |
| CronJobs referencing SA            | **None**                                                             |
| RoleBindings referencing SA        | **None**                                                             |
| ClusterRoleBindings referencing SA | `cluster-admin-sa-binding` → `cluster-admin` (the one being deleted) |
| ownerReferences                    | **None**                                                             |
| Long-lived Secret token            | **None**                                                             |

---

## Deletion Results

All four `kubectl delete` commands returned success:

```
clusterrolebinding.rbac.authorization.k8s.io "kernel-service-account" deleted
serviceaccount "kernel-service-account" deleted from jupyterhub-ns namespace
clusterrolebinding.rbac.authorization.k8s.io "cluster-admin-sa-binding" deleted
serviceaccount "cluster-admin-sa" deleted from kube-system namespace
```

---

## Post-Deletion Verification

### Resources confirmed gone (all return `NotFound`)

| Resource | Status |
|---|---|
| SA `kernel-service-account` in `jupyterhub-ns` | ✅ NotFound |
| SA `cluster-admin-sa` in `kube-system` | ✅ NotFound |
| CRB `kernel-service-account` | ✅ NotFound |
| CRB `cluster-admin-sa-binding` | ✅ NotFound |

### Controller recreation check

Re-queried all four resources ~10 seconds after deletion. **None were recreated** by any controller, operator, or Helm reconciliation loop.

### Workload health

| Check | Result |
|---|---|
| JupyterHub namespace | No resources (was already idle pre-deletion) |
| Unhealthy pods cluster-wide | Same 4 pre-existing failures (ImagePullBackOff / OOMKilled) — **unchanged from pre-deletion** |
| Kubernetes API | ✅ Control plane running at `https://127.0.0.1:6443` |
| CoreDNS | ✅ Running |
| Metrics-server | ✅ Running |

### No unexpected behavior

Zero anomalies observed. Pod health is identical before and after deletion.

---

## Current Subjects with `cluster-admin` (Post-Cleanup)

### Legitimate Boss/Admin Users (Rancher-managed — DO NOT TOUCH)

| Subject | Kind | Identity |
|---|---|---|
| `u-kpjdm` | User | Yashveer Singh (Codemaverick) |
| `u-m8n5w` | User | Krushn Dayshmookh |
| `u-oejzho4xcz` | User | Krit Garg |
| `u-plt4na7re5` | User | Kanishk Ranjan |
| `user-tsnjs` | User | Default Admin (Rancher bootstrap) |

### Legitimate Infrastructure ServiceAccounts

| Subject | Kind | Namespace | Purpose |
|---|---|---|---|
| `system:masters` | Group | N/A | K8s built-in superuser group |
| `helm-traefik` | ServiceAccount | `kube-system` | Traefik Helm controller |
| `helm-traefik-crd` | ServiceAccount | `kube-system` | Traefik CRD Helm controller |
| `longhorn-support-bundle` | ServiceAccount | `longhorn-system` | Longhorn diagnostics |
| `rancher` | ServiceAccount | `cattle-system` | Rancher server |
| `rancher-webhook` | ServiceAccount | `cattle-system` | Rancher webhook |
| `system-upgrade-controller` | ServiceAccount | `cattle-system` | K3s system upgrade controller |

### Removed (this cleanup)

| Subject | Kind | Namespace | Reason |
|---|---|---|---|
| ~~`kernel-service-account`~~ | ~~ServiceAccount~~ | ~~`jupyterhub-ns`~~ | Orphaned; zero workload references; JupyterHub uses `hub` SA |
| ~~`cluster-admin-sa`~~ | ~~ServiceAccount~~ | ~~`kube-system`~~ | Orphaned; zero workload references; no Helm/operator ownership |

---

## Summary

Two orphaned ServiceAccounts with unrestricted `cluster-admin` privileges were safely removed from the cluster. Both were manually created via `kubectl apply` (evidenced by `kubectl.kubernetes.io/last-applied-configuration` annotations), had no ownerReferences, no Helm management, and zero workload consumers. Their removal had zero impact on cluster operations. No controller has recreated them.
