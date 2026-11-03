# Least-Privilege Audit: Remaining cluster-admin ServiceAccounts

**Date:** 2026-09-28  
**Scope:** Read-only analysis of all 6 remaining non-User ServiceAccounts with `cluster-admin`  
**Constraint:** DO NOT MODIFY ANYTHING

---

## Individual ServiceAccount Analysis

### 1. `kube-system/helm-traefik`

| Field                             | Value                                                                                                 |
| -----------------------------------| -------------------------------------------------------------------------------------------------------|
| **Namespace**                     | `kube-system`                                                                                         |
| **Created**                       | `2025-12-25T18:42:35Z`                                                                                |
| **ownerReferences**               | `HelmChart/traefik` (K3s embedded Helm controller)                                                    |
| **Labels**                        | `objectset.rio.cattle.io/hash` (K3s objectset management)                                             |
| **Workloads Using It**            | `Job/helm-install-traefik` (Completed, 144d ago)                                                      |
| **Controller/Helm Ownership**     | K3s embedded Helm controller (`objectset.rio.cattle.io/owner-gvk: helm.cattle.io/v1, Kind=HelmChart`) |
| **CRB**                           | `helm-kube-system-traefik` → `cluster-admin`                                                          |
| **Additional RoleBindings**       | None                                                                                                  |
| **Current Effective Permissions** | `*.*` on `*` (full cluster-admin)                                                                     |

**Why cluster-admin exists:** K3s's embedded Helm controller creates a Job to install/upgrade Traefik. That Job runs under this SA and needs permissions to create CRDs, ClusterRoles, Services, Deployments, etc. across multiple namespaces. K3s hardcodes cluster-admin for Helm install jobs by design when no `spec.jobServiceAccount` is set in the HelmChart CR (currently `null`).

**Actual required permissions:** The Job is already completed. The SA is only needed during Traefik install/upgrade. When a K3s upgrade triggers a Traefik re-install, K3s will create a new Job using this SA. The Helm install needs broad permissions across CRDs, RBAC, namespaces, services, and IngressRoutes.

**Least-privilege alternative:** Set `spec.jobServiceAccount` in the HelmChart CR to a custom SA with narrowly-scoped permissions. However, determining the exact minimal set is complex (Traefik chart creates CRDs, ClusterRoles, Middleware CRDs, etc.).

**Safe to change?** ⚠️ **NO — HIGH RISK**

**Reason:** The CRB is managed by K3s's objectset controller (`objectset.rio.cattle.io`). If you delete or modify it, K3s will recreate it with `cluster-admin` on the next reconciliation loop. The HelmChart CR has `jobServiceAccount: null`, meaning K3s defaults to creating a `helm-{namespace}-{chartname}` SA with cluster-admin. To change this, you would need to modify the HelmChart CR's `spec.jobServiceAccount` field, which is a K3s-level change that could break Traefik upgrades if the replacement SA lacks any required permission. **Do not touch.**

---

### 2. `kube-system/helm-traefik-crd`

| Field | Value |
|---|---|
| **Namespace** | `kube-system` |
| **Created** | `2025-12-25T18:42:35Z` |
| **ownerReferences** | `HelmChart/traefik-crd` (K3s embedded Helm controller) |
| **Labels** | `objectset.rio.cattle.io/hash` |
| **Workloads Using It** | `Job/helm-install-traefik-crd` (Completed, 144d ago) |
| **Controller/Helm Ownership** | K3s embedded Helm controller |
| **CRB** | `helm-kube-system-traefik-crd` → `cluster-admin` |
| **Additional RoleBindings** | None |
| **Current Effective Permissions** | `*.*` on `*` (full cluster-admin) |

**Why cluster-admin exists:** Identical pattern to `helm-traefik`. This SA runs the CRD installation Job for Traefik's Custom Resource Definitions. CRD creation requires cluster-scoped write access to `apiextensions.k8s.io`.

**Actual required permissions:** Cluster-scoped write to `customresourcedefinitions` in `apiextensions.k8s.io`. Narrower than full cluster-admin, but K3s hardcodes this.

**Least-privilege alternative:** Same as above — would require setting `spec.jobServiceAccount` on the HelmChart CR.

**Safe to change?** ⚠️ **NO — HIGH RISK**

**Reason:** Same as `helm-traefik`. K3s objectset controller will recreate the cluster-admin CRB. Modifying the HelmChart CR risks breaking CRD upgrades during K3s node upgrades. **Do not touch.**

---

### 3. `longhorn-system/longhorn-support-bundle`

| Field | Value |
|---|---|
| **Namespace** | `longhorn-system` |
| **Created** | `2026-07-06T13:37:27Z` |
| **ownerReferences** | None |
| **Labels** | `app.kubernetes.io/managed-by: Helm`, `helm.sh/chart: longhorn-1.12.0` |
| **Workloads Using It** | **None** — no pods, no jobs, no deployments reference it |
| **Controller/Helm Ownership** | Longhorn Helm chart v1.12.0 (`meta.helm.sh/release-name: longhorn`) |
| **CRB** | `longhorn-support-bundle` → `cluster-admin` |
| **Additional RoleBindings** | None |
| **Current Effective Permissions** | `*.*` on `*` (full cluster-admin) |

**Why cluster-admin exists:** Longhorn's support-bundle feature creates an on-demand Job that collects diagnostic information from across the entire cluster (logs, events, node status, volume status across all namespaces). The Helm chart pre-provisions this SA and CRB so the support-bundle Job can be launched instantly when needed without RBAC setup.

**Actual required permissions:** When a support bundle is generated (manually triggered from Longhorn UI or API), the Job needs: `get/list` on `pods`, `nodes`, `events`, `pvs`, `pvcs`, `storageclasses`, `configmaps` across all namespaces, plus `get/list` on Longhorn CRDs (`volumes.longhorn.io`, `replicas.longhorn.io`, `engines.longhorn.io`, etc.).

**Least-privilege alternative:** A custom ClusterRole with `get`, `list`, `watch` (read-only) on the specific resources enumerated above. This would be significantly narrower than cluster-admin while still allowing support-bundle generation.

**Safe to change?** ⚠️ **NO — MEDIUM RISK (Helm reconciliation)**

**Reason:** The SA and CRB are managed by the Longhorn Helm release. If you delete or modify the CRB, the next `helm upgrade` of Longhorn (or Rancher's automatic Helm reconciliation) will recreate it with cluster-admin as specified in the chart's templates. To permanently fix this, you would need to either:
1. Fork the Longhorn Helm chart values to override the support-bundle RBAC (not supported by the chart), or
2. Accept the risk and patch after each upgrade.

Neither is practical. The SA is dormant (no pods use it until a support bundle is manually triggered), so the runtime exposure is very low. **Do not touch without forking the chart.**

---

### 4. `cattle-system/rancher`

| Field | Value |
|---|---|
| **Namespace** | `cattle-system` |
| **Created** | `2025-12-31T07:21:07Z` |
| **ownerReferences** | None |
| **Labels** | `app.kubernetes.io/managed-by: Helm`, `chart: rancher-2.13.1` |
| **Workloads Using It** | **Active:** `Deployment/rancher` → `Pod/rancher-7fbb9dcc78-kxt5c` (Running) |
| **Controller/Helm Ownership** | Rancher Helm chart v2.13.1 |
| **CRB** | `rancher` → `cluster-admin` |
| **Additional RoleBindings** | None |
| **Current Effective Permissions** | `*.*` on `*` (full cluster-admin) |

**Why cluster-admin exists:** Rancher is the cluster management platform. It manages users, RBAC, projects, namespaces, catalogs, fleet, multi-cluster management, and every aspect of Kubernetes resources. It is architecturally equivalent to an API server extension — it must be able to create, modify, and delete any resource in any namespace to fulfill its role.

**Actual required permissions:** Genuinely `*.*` on `*`. Rancher creates ClusterRoles, ClusterRoleBindings, Namespaces, CRDs, Secrets, Deployments, ServiceAccounts, and custom Rancher resources (Users, Projects, Clusters, GlobalRoles) across all namespaces.

**Least-privilege alternative:** None. Rancher by design requires cluster-admin. The Rancher project explicitly documents this as a hard requirement.

**Safe to change?** ❌ **NO — WILL BREAK RANCHER**

**Reason:** Removing or reducing this binding will immediately break the Rancher management server. User authentication, project management, namespace creation, Helm app deployments from Rancher UI, and all cluster management features will fail. The Helm chart will also recreate the CRB on the next reconciliation. **Do not touch.**

---

### 5. `cattle-system/rancher-webhook`

| Field | Value |
|---|---|
| **Namespace** | `cattle-system` |
| **Created** | `2025-12-31T07:23:46Z` |
| **ownerReferences** | None |
| **Labels** | `app.kubernetes.io/managed-by: Helm` |
| **Workloads Using It** | **Active:** `Deployment/rancher-webhook` → `Pod/rancher-webhook-65fd7d4ff-jcthk` (Running) |
| **Controller/Helm Ownership** | `rancher-webhook` Helm release |
| **CRB** | `rancher-webhook` → `cluster-admin` |
| **Additional RoleBindings** | None |
| **Current Effective Permissions** | `*.*` on `*` (full cluster-admin) |

**Why cluster-admin exists:** The Rancher webhook is a ValidatingWebhookConfiguration (29 webhooks) and MutatingWebhookConfiguration (7 webhooks) that intercepts and validates/mutates virtually every resource type in the cluster. To validate whether a user can create a resource, the webhook itself needs read access to the full RBAC tree, namespace metadata, Rancher CRDs, and more.

**Actual required permissions:** Broad read access across all resource types plus write access to its own webhook configurations, secrets (for TLS cert rotation), and Rancher CRDs. Effectively needs near-cluster-admin.

**Least-privilege alternative:** Theoretically possible but would require enumerating all 36 webhook rules and their validation dependencies. The Rancher project does not support or document a reduced-privilege mode for the webhook.

**Safe to change?** ❌ **NO — WILL BREAK RANCHER**

**Reason:** The webhook intercepts resource creation/modification cluster-wide. If it loses permissions to read the resources it validates, all API mutations will start failing (webhook denies requests it can't validate). The Helm chart manages this CRB and will recreate it. **Do not touch.**

---

### 6. `cattle-system/system-upgrade-controller`

| Field | Value |
|---|---|
| **Namespace** | `cattle-system` |
| **Created** | `2025-12-31T07:24:33Z` |
| **ownerReferences** | None |
| **Labels** | `app.kubernetes.io/managed-by: Helm` |
| **Workloads Using It** | `Deployment/system-upgrade-controller` (**scaled to 0 replicas**, no running pods) |
| **Controller/Helm Ownership** | `system-upgrade-controller` Helm release |
| **CRB** | `system-upgrade-controller` → `cluster-admin` |
| **Additional RoleBindings** | None |
| **Current Effective Permissions** | `*.*` on `*` (full cluster-admin) |

**Why cluster-admin exists:** The system-upgrade-controller (Rancher's SUC) manages K3s node upgrades. When a `Plan` CR is created, SUC creates privileged DaemonSet pods on each node that run the upgrade binary. This requires: creating pods with `hostPID`, `hostNetwork`, and `privileged` security contexts; reading/writing node labels and annotations; cordoning/draining nodes; and creating RBAC for the upgrade pods.

**Actual required permissions:** `nodes` (get/list/watch/update/patch), `pods` (create/delete/get/list), `daemonsets` (create/delete/get/list), `plans.upgrade.cattle.io` (get/list/watch/update), `jobs` (create/get/list), `secrets` (get — for plan config). The controller does genuinely need cluster-scoped access to nodes, but not full `*.*`.

**Least-privilege alternative:** A custom ClusterRole scoped to: `nodes`, `pods`, `daemonsets`, `jobs`, `secrets` (read), `plans.upgrade.cattle.io` (full), `events` (create). This would be significantly narrower than cluster-admin.

**Safe to change?** ⚠️ **NO — MEDIUM RISK (but currently dormant)**

**Reason:** The deployment is scaled to 0 replicas, so no pods currently run with this SA. However:
1. The CRB is managed by a Helm release. Modifying it manually will be overwritten on the next `helm upgrade`.
2. If Rancher scales the deployment back up for a K3s upgrade, a reduced ClusterRole could cause the upgrade to fail silently.
3. The Rancher SUC project does not document or support a non-cluster-admin mode.

**Do not touch.** The runtime risk is currently zero (0 replicas), and the Helm chart will revert any changes.

---

## Summary Classification

### 1. ServiceAccounts That Genuinely Require cluster-admin

| ServiceAccount | Namespace | Reason |
|---|---|---|
| `rancher` | `cattle-system` | Full cluster management platform; architecturally requires `*.*` |
| `rancher-webhook` | `cattle-system` | Validates/mutates all resource types cluster-wide via 36 admission webhooks |

### 2. ServiceAccounts That Could Theoretically Be Reduced But CANNOT Be Safely Changed

| ServiceAccount | Namespace | Why It Cannot Be Changed |
|---|---|---|
| `helm-traefik` | `kube-system` | K3s objectset controller will recreate cluster-admin CRB automatically |
| `helm-traefik-crd` | `kube-system` | K3s objectset controller will recreate cluster-admin CRB automatically |
| `longhorn-support-bundle` | `longhorn-system` | Helm chart manages CRB; next upgrade reverts any change |
| `system-upgrade-controller` | `cattle-system` | Helm chart manages CRB; dormant (0 replicas) but needed for K3s upgrades |

### 3. ServiceAccounts Whose Purpose Is Unclear

**None.** All six ServiceAccounts have clear, documented purposes with identifiable controllers.

### 4. Remediation Plan

**There are no ServiceAccounts that can be safely reduced at this time.**

Every remaining cluster-admin CRB is either:
- **Genuinely required** (Rancher, Rancher-webhook), or
- **Managed by an automated controller** (K3s objectset, Helm) that will immediately recreate the cluster-admin binding if you delete or modify it.

Manual patches would be overwritten on the next K3s upgrade, Helm upgrade, or Rancher reconciliation cycle, creating a false sense of security and potential upgrade failures.

### Recommended Long-Term Actions (Not for Immediate Execution)

1. **Traefik Helm SAs:** File a K3s feature request or set `spec.jobServiceAccount` on the HelmChart CRs to use a pre-provisioned least-privilege SA. Requires careful testing of Traefik chart's full resource creation footprint.
2. **Longhorn support-bundle:** File a Longhorn issue requesting support for a configurable RBAC scope in the support-bundle chart template.
3. **system-upgrade-controller:** File a Rancher SUC issue requesting a documented least-privilege ClusterRole. Since it's currently scaled to 0, the runtime risk is zero.
4. **Rancher + webhook:** Accept as architectural requirements. These are equivalent to the API server itself.
