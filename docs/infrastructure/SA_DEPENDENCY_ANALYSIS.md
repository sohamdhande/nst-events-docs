# ServiceAccount Dependency Analysis

## 1. jupyterhub-ns/kernel-service-account

- **Creation Timestamp:** `2026-06-26T09:26:46Z`
- **Labels and Annotations:** No labels. Has `kubectl.kubernetes.io/last-applied-configuration` (indicates manual `kubectl apply`).
- **ownerReferences:** None.
- **Referenced by Workloads:** NONE. (No Pod, Deployment, StatefulSet, DaemonSet, Job, or CronJob uses this SA anywhere in the cluster).
- **Helm/Operator References:** None. (No Helm releases manage this).
- **RoleBindings/ClusterRoleBindings:** Referenced exactly once by ClusterRoleBinding `kernel-service-account`.
- **Exact Effective RBAC:** `*.*` on `*` (Full Global `cluster-admin`).
- **JupyterHub Inspection:** JupyterHub's `hub-config` ConfigMap dictates `c.KubeSpawner.service_account_name = 'hub'`. The spawner uses the legitimate `hub` ServiceAccount, not `kernel-service-account`. No long-lived Secret token exists for `kernel-service-account`.
- **Dependencies/Breakage Risk:** Zero. The SA is completely orphaned.

### ACTUAL PURPOSE
Likely an abandoned experiment by a developer trying to give Jupyter notebook kernels specific API permissions, but they mistakenly used `cluster-admin` and then abandoned the SA entirely in favor of the default `hub` account.

### MINIMUM REQUIRED PERMISSIONS
None (the SA is unused).

### SAFE REMEDIATION
**4. Remove the ServiceAccount entirely.**
Delete both the `kernel-service-account` ServiceAccount and the `kernel-service-account` ClusterRoleBinding. No controller or Helm release will recreate them.

---

## 2. kube-system/cluster-admin-sa

- **Creation Timestamp:** `2026-05-05T09:20:56Z`
- **Labels and Annotations:** No labels. Has `kubectl.kubernetes.io/last-applied-configuration`.
- **ownerReferences:** None.
- **Referenced by Workloads:** NONE. 
- **Helm/Operator References:** None.
- **RoleBindings/ClusterRoleBindings:** Referenced exactly once by ClusterRoleBinding `cluster-admin-sa-binding`.
- **Exact Effective RBAC:** `*.*` on `*` (Full Global `cluster-admin`).
- **Dependencies/Breakage Risk:** Zero. Like the above, there are no long-lived Secret tokens and no Pods using this SA.

### ACTUAL PURPOSE
This is an old, manually provisioned ServiceAccount created a few months ago, likely as a backdoor or a quick way to generate a `cluster-admin` token for a CI/CD pipeline or external script. Since Kubernetes >= v1.24 no longer auto-generates secret tokens for SAs, and no secret exists for it, it is currently dead.

### MINIMUM REQUIRED PERMISSIONS
None (the SA is unused).

### SAFE REMEDIATION
**4. Remove the ServiceAccount entirely.**
Delete both the `cluster-admin-sa` ServiceAccount and the `cluster-admin-sa-binding` ClusterRoleBinding. It is entirely orphaned. No automation will recreate it.
