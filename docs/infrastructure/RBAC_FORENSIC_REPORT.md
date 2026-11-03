# RBAC Forensic Audit Report

## 1. Analysis of the 5 Target Identities
The security audit initially flagged these five identities as "student accounts" with `cluster-admin` access. **Forensic analysis reveals this was a false positive: these are the legitimate, intended "Boss" administrators.**

### 1. u-kpjdm (Yashveer Singh / Codemaverick)
- **Role/Ownership:** Legitimate Infrastructure Admin (Boss Account)
- **Effective Permissions:** `*.*` (cluster-admin)
- **Direct/Inherited:** Direct via Rancher provisioning.
- **Relevant Bindings:**
  - `CRB: cattle-globalrolebinding-grb-vxwtm` -> `cattle-globalrole-admin`
  - `CRB: globaladmin-u-kpjdm` -> `cluster-admin`
- **Why it exists:** Rancher Global Admin assignment.
- **Intended/Unintended:** INTENDED
- **Remediation:** DO NOT REMOVE (Removing this breaks Yashveer's access).

### 2. u-m8n5w (Krushn Dayshmookh)
- **Role/Ownership:** Legitimate Infrastructure Admin (Boss Account)
- **Effective Permissions:** `*.*` (cluster-admin)
- **Direct/Inherited:** Direct via Rancher provisioning.
- **Relevant Bindings:**
  - `CRB: cattle-globalrolebinding-grb-rglcp` -> `cattle-globalrole-admin`
  - `CRB: globaladmin-u-m8n5w` -> `cluster-admin`
- **Why it exists:** Rancher Global Admin assignment.
- **Intended/Unintended:** INTENDED
- **Remediation:** DO NOT REMOVE (Removing this breaks Krushn's access).

### 3. u-oejzho4xcz (Krit Garg)
- **Role/Ownership:** Legitimate Infrastructure Admin (Boss Account)
- **Effective Permissions:** `*.*` (cluster-admin)
- **Direct/Inherited:** Direct via Rancher provisioning.
- **Relevant Bindings:**
  - `CRB: cattle-globalrolebinding-grb-admin-krit` -> `cattle-globalrole-admin`
  - `CRB: globaladmin-u-oejzho4xcz` -> `cluster-admin`
- **Why it exists:** Rancher Global Admin assignment.
- **Intended/Unintended:** INTENDED
- **Remediation:** DO NOT REMOVE (Removing this breaks Krit's access).

### 4. u-plt4na7re5 (Kanishk Ranjan)
- **Role/Ownership:** Legitimate Infrastructure Admin (Boss Account)
- **Effective Permissions:** `*.*` (cluster-admin)
- **Direct/Inherited:** Direct via Rancher provisioning.
- **Relevant Bindings:**
  - `CRB: cattle-globalrolebinding-grb-xmz5c` -> `cattle-globalrole-admin`
  - `CRB: globaladmin-u-plt4na7re5` -> `cluster-admin`
- **Why it exists:** Rancher Global Admin assignment.
- **Intended/Unintended:** INTENDED
- **Remediation:** DO NOT REMOVE (Removing this breaks Kanishk's access).

### 5. user-tsnjs (Default Admin)
- **Role/Ownership:** Legitimate Infrastructure Admin (Rancher Default Admin)
- **Effective Permissions:** `*.*` (cluster-admin)
- **Direct/Inherited:** Direct via Rancher provisioning.
- **Relevant Bindings:**
  - `CRB: cattle-globalrolebinding-globalrolebinding-8t9tc` -> `cattle-globalrole-admin`
  - `CRB: globaladmin-user-tsnjs` -> `cluster-admin`
- **Why it exists:** Default Rancher bootstrap admin.
- **Intended/Unintended:** INTENDED
- **Remediation:** DO NOT REMOVE.

### Dependency Impact of Removal
If the `globaladmin-*` and `cattle-globalrolebinding-*` bindings for these 5 accounts are removed, it will **immediately lock out the core infrastructure administrators** (Kanishk, Krushn, Krit, Yashveer, and the default admin) from managing the Kubernetes cluster and Rancher UI. This would have catastrophic dependency impact on cluster operability.

---

## 2. Other Privileged Subjects (Wildcard / Secret Readers)

*(Note: Standard controllers like `kube-system` garbage collectors, `cert-manager`, and Rancher system accounts are omitted for brevity unless suspicious).*

| SUBJECT | CURRENT PRIVILEGE | BINDING | WHY IT EXISTS | INTENDED/UNINTENDED | REMEDIATION |
|---|---|---|---|---|---|
| **SA: kernel-service-account** (ns: `jupyterhub-ns`) | `cluster-admin` / Wildcard | CRB: `kernel-service-account` | Likely a misconfiguration intended to let Jupyterhub spawn pods, but drastically overprivileged. | **UNINTENDED (CRITICAL)** | Remove CRB. Replace with a Role binding restricted to `jupyterhub-ns`. |
| **SA: cluster-admin-sa** (ns: `kube-system`) | `cluster-admin` / Wildcard | CRB: `cluster-admin-sa-binding` | Explicit generic SA mapped to `cluster-admin`. Potentially a backdoor or leftover debug account. | **UNINTENDED (HIGH)** | Delete CRB `cluster-admin-sa-binding` and the SA. |
| **User: u-b4qkhsnliz** (System) | `cluster-admin` / Wildcard | CRB: `crb-sxpl5qepks` -> `cluster-owner` | Rancher internal system account for local cluster. | INTENDED | Do nothing. |
| **User: u-mo773yttt4** (System) | `cluster-admin` / Wildcard | CRB: `crb-ymn4zyccca` -> `cluster-owner` | Rancher internal system account. | INTENDED | Do nothing. |
| **SA: admiralty** (ns: `admiralty`) | Read Secrets Cluster-wide | CRB: `admiralty` | Admiralty multi-cluster scheduler requires secret sync. | INTENDED | Ensure restricted if possible, otherwise accept risk. |
| **SA: enterprise-gateway-sa** (ns: `enterprise-gateway`) | Read Secrets Cluster-wide | CRB: `enterprise-gateway-controller` | Enterprise Gateway controller. | INTENDED | Accept risk. |
