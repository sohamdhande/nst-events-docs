# Pod Security and Namespace Isolation Audit

**Date:** 2026-09-28
**Scope:** Read-only audit of Namespace isolation, Pod Security Standards (PSS), and NetworkPolicies.

---

## 1. Escaping and Privilege Escalation Vectors

Based on the audit of existing student/tenant workloads, the cluster is currently operating in a highly permissive state. 

### Can a student create a pod that:
- **Accesses the host filesystem?** **YES.** Numerous student pods are mounting `hostPath` volumes (e.g., `node-debugger-nst-n2` mounting host paths, `minepanel` mounting 3 host paths, `career-pyramids` mounting 2).
- **Accesses the Kubernetes API with excessive credentials?** **YES.** `automountServiceAccountToken: true` is set on almost every pod. While the cluster-admin orphaned SAs have been removed, students can still mount default tokens and potentially abuse local RoleBindings.
- **Uses host networking/PID/IPC?** **YES.** The `node-debugger-nst-n2-444tp` pod in the `default` namespace is currently running with `hostNetwork: true`, `hostPID: true`, and `hostIPC: true`. This is a complete host takeover vector.
- **Obtains another namespace's secrets?** **POTENTIALLY.** By using `hostPath` to mount `/var/lib/kubelet/pods` or `/etc/kubernetes`, a student could extract secrets belonging to other namespaces from the host.
- **Communicates freely with other tenant namespaces?** **YES.** There are **ZERO** NetworkPolicies in 98% of student namespaces. East-west traffic is completely unrestricted by default-allow behavior.

---

## 2. Pod Security Admission Policy Design (Proposed)

*Note: This is a design only. No labels have been applied.*

### Proposed PSS Assignments

| Namespace Class | Proposed PSS Level | Rationale |
|---|---|---|
| **Infrastructure/System** | `privileged` | Required for CNI, CSI, Kube-proxy, Node exporters. |
| **Platform/Management** | `privileged` | Required for Rancher, Longhorn (needs host mounts), Cert-Manager. |
| **GPU Resources** | `privileged` | `gpu-operator-resources` requires host mounts, hostPID, and privileged containers for NVIDIA drivers. |
| **Production App** | `baseline` | `nst-events` can run safely under `baseline` (blocking host namespaces and privileged mode). |
| **Student/Tenant** | `restricted` | All student workloads MUST run as non-root, drop all capabilities, and be blocked from host mounts/namespaces. |

### Anticipated Breakages Under Proposed Policy

If `restricted` is enforced on student namespaces today, the following actively deployed student workloads **will break**:

- **Privileged / Host Namespaces:** 
  - `default/Pod/node-debugger-nst-n2` (violates hostNetwork, hostPID, hostIPC)
- **HostPath Volumes:**
  - `default/Deployment/career-pyramids`
  - `default/Deployment/sentinal-portal`
  - `downloads/Deployment/downloads`
  - `nst-minepanel/Deployment/minepanel`
  - `amlc-ber-jy` (all 40+ ML jobs use a hostPath mount)
- **Missing `runAsNonRoot` / allowPrivilegeEscalation:**
  - **100% of student workloads** will fail admission. Currently, every single student deployment (e.g., `apps/*`, `awesomerag`, `overleaf`, `n8n`) runs without `runAsNonRoot: true` and defaults to `allowPrivilegeEscalation: true`.

---

## 3. NetworkPolicy Audit

### Findings
- **Zero Policies:** 84 out of 87 namespaces have absolutely zero NetworkPolicies.
- **Default-Allow:** Kubernetes defaults to "allow all" when no policies are present.
- **East-West Traffic:** Completely unrestricted. A pod in `yashsingh045` can freely ping or port-scan a pod in `prabhav-minecraft` or `nst-events`.
- **Ingress/Egress:** Unrestricted. Any student pod can initiate outbound connections to the internet or internal management interfaces (like the Longhorn UI or Rancher API).

---

## 4. Namespace Summary and Remediation Plan

### Raw Output Snapshot

| NAMESPACE | CLASS | CURRENT SECURITY | REQUIRED POLICY | BREAKAGES |
|---|---|---|---|---|
| `kube-system` | Infrastructure | `NONE` | `privileged` | None |
| `cattle-system` | Platform | `NONE` | `privileged` | None |
| `gpu-operator-resources` | Platform | `NONE` | `privileged` | None |
| `nst-events` | Production | `NONE` | `baseline` | Need to verify `runAsNonRoot` |
| `default` | Student/Tenant | `NONE` | `restricted` | `node-debugger`, `career-pyramids` |
| `apps` | Student/Tenant | `NONE` | `restricted` | All 28 Deployments |
| `amlc-ber-jy` | Student/Tenant | `NONE` | `restricted` | All ML Jobs |
| `nst-minepanel` | Student/Tenant | `NONE` | `restricted` | `minepanel` |

### Concrete Staged Remediation Plan

**Phase 1: Observation (Warn/Audit)**
1. Apply `pod-security.kubernetes.io/warn=restricted` and `pod-security.kubernetes.io/audit=restricted` labels to all Student/Tenant namespaces.
2. Monitor audit logs to catalog every specific pod that violates the standard without breaking them immediately.

**Phase 2: Network Isolation**
1. Deploy a default-deny Ingress/Egress NetworkPolicy to a test tenant namespace.
2. Roll out a global `deny-all-cross-namespace` NetworkPolicy to all Student/Tenant namespaces, explicitly whitelisting DNS (port 53 to `kube-system`) and specific required external ingress (via Traefik).

**Phase 3: Workload Refactoring**
1. Mandate that students update their Helm charts/manifests to include `securityContext: runAsNonRoot: true` and `allowPrivilegeEscalation: false`.
2. Migrate valid `hostPath` usage (like shared datasets in ML jobs) to `PersistentVolumeClaims` backed by Longhorn, eliminating the need for `hostPath`.

**Phase 4: Enforcement**
1. Remove offending `hostNetwork`/`hostPID` pods manually (like the `node-debugger`).
2. Apply `pod-security.kubernetes.io/enforce=restricted` to all Student/Tenant namespaces.
3. Apply `pod-security.kubernetes.io/enforce=baseline` to `nst-events`.
