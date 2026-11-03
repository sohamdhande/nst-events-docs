# NST Compute Cluster Security Audit Report
**Date:** 2026-09-28
**Auditor:** Antigravity (using `soham` / `root` read-only access)

## Executive Assessment
The NST Compute Cluster demonstrates a strong focus on host-level security and perimeter defense, leveraging advanced techniques like kernel immutable locks (`chattr +i`) on mission-critical files, Cloudflare Tunnels for SSH, and kernel-level keystroke telemetry. However, **this strict host-level perimeter is severely undermined by critical vulnerabilities within the Kubernetes control plane.** 

Most alarmingly, several student accounts hold global `cluster-admin` RBAC bindings, meaning they can trivially bypass all host-level protections by launching privileged pods, accessing secrets, or manipulating the cluster. Additionally, the lack of Pod Security Standards (PSS), minimal microsegmentation (only 4 NetworkPolicies cluster-wide), and unencrypted secrets present substantial internal risks. The contrast between the heavily fortified host OS and the overly permissive Kubernetes environment constitutes the highest risk to the cluster.

---

## Executive Summary: Critical Findings

| Severity | Category | Finding | Impact |
|---|---|---|---|
| **CRITICAL** | K8s RBAC | Multiple student accounts have `cluster-admin` bindings. | Full cluster compromise; bypasses all host-level immutable locks. |
| **HIGH** | Pod Security | No Pod Security Standards (PSS) enforced on any namespace. | Attackers or workloads can run privileged pods, mount `/`, and escalate to host root. |
| **HIGH** | Network | Severe lack of microsegmentation (only 4 NetworkPolicies). | Lateral movement is unrestricted between almost all tenant namespaces. |
| **MEDIUM** | Secrets | K8s secrets are stored as plaintext Base64 `Opaque` types. | Any user with read access to secrets can compromise databases and external integrations. |
| **MEDIUM** | Host/Node | Software Version Drift across the 7 nodes. | Node `nst-n5` runs an older kernel (6.17.0-22). K3s and containerd versions are fragmented. |

---

## 1. HOST / NODE SECURITY
- **SSH Configuration:** Effectively hardened. Direct LAN access to port 22 is dropped via `iptables`. Authentication is strictly public-key based (`passwordauthentication no`), and root login is restricted. 
- **Immutable Files:** Kernel locks (`chattr +i`) are actively enforced on `/etc/passwd`, `/etc/shadow`, `/etc/gshadow`, `/etc/sudoers`, `/etc/ssh/sshd_config`, and `/etc/sudoers.d/bosses`.
- **Telemetry:** Comprehensive audit logging is active via `pam_tty_audit.so` and `/var/log/commands.log`.
- **Node Drift:** 
  - `nst-n1` and `nst-n6`: K3s v1.35.4
  - `nst-n2`, `nst-n3`, `nst-n4`, `nst-n5`: K3s v1.33.6
  - `nst-n7`: K3s v1.36.4
  - `nst-n5` is running an older kernel (`6.17.0-22-generic`) compared to the others (`6.17.0-41-generic`).

## 2. KUBERNETES RBAC
- **CRITICAL FINDING:** The following student/tenant users possess `cluster-admin` privileges via direct `ClusterRoleBindings`:
  - `u-kpjdm`
  - `u-m8n5w`
  - `u-oejzho4xcz`
  - `u-plt4na7re5`
  - `user-tsnjs`
- **Impact:** These users have unconditional control over the Kubernetes API, allowing them to extract all secrets, delete workloads, or deploy privileged pods that can take over the host nodes.

## 3. POD SECURITY STANDARDS (PSS)
- **Status:** Not Enforced.
- **Finding:** A cluster-wide check revealed **no** PSS labels (`pod-security.kubernetes.io/*`) on any namespaces. 
- **Privileged Pods:** Numerous pods run with `Privileged`, `RunAsRoot`, or Host IPC/PID/Net flags enabled (e.g., `longhorn-system`, `gpu-operator-resources`, and `hami-system`). Without PSS, any developer can deploy similar privileged pods.

## 4. NETWORK POLICIES & MICROSEGMENTATION
- **Status:** Severely Lacking.
- **Finding:** Only 4 NetworkPolicies exist across the entire cluster:
  - `cattle-fleet-local-system/default-allow-all`
  - `istio-system/allow-eastwest-relays-ingress`
  - `network-runtime/outbound-relay`
  - `nst-events/postgres-network-policy`
- **Impact:** By default, all pods can communicate with all other pods across namespaces, creating a massive flat network internally.

## 5. INGRESS & PUBLIC EXPOSURE
- **Status:** Extensive Exposure.
- **Finding:** Over 50 applications and APIs are exposed publicly via Traefik Ingress on port 80/443 (e.g., `events.nstsdc.org`, student apps, documentation, RAG services). 
- **Impact:** The attack surface is vast. Any vulnerability in a single exposed student application could be used to gain a foothold in the cluster, which lacks internal microsegmentation to contain the breach.

## 6. SECRETS MANAGEMENT
- **Status:** Basic / Unencrypted.
- **Finding:** All secrets (including `nst-db-admin-secrets`, AWS credentials, and API keys) are stored as standard `Opaque` Kubernetes secrets (Base64 encoded). 
- **Impact:** There is no integration with an external KMS, HashiCorp Vault, or Sealed Secrets. Anyone with `get secrets` RBAC in a namespace can read plaintext credentials.

## 7. SUPPLY CHAIN & IMAGE SECURITY
- **Status:** Unenforced.
- **Finding:** No Validating or Mutating Webhooks exist for image scanning or provenance verification (e.g., no Kyverno, Trivy, or OPA Gatekeeper).
- **Impact:** Users can deploy arbitrary unvetted Docker images from public registries directly into the cluster.

## 8. KUBERNETES CONTROL PLANE SECURITY
- **Status:** Secure.
- **Finding:** The K3s API server disables anonymous authentication (`401 Unauthorized`), protecting against unauthenticated reconnaissance.
- **Impact:** Strong protection against external unauthenticated probes.

## 9. STORAGE SECURITY
- **Status:** Standard.
- **Finding:** Longhorn is used for persistent storage. The CSI plugin and engine managers run as heavily privileged daemonsets (expected for storage, but widens the privileged footprint).

## 10. DATABASE SECURITY
- **Status:** Secured (Network level).
- **Finding:** `nst-postgres` is correctly isolated using `postgres-network-policy`. Furthermore, `iptables` rules exist on the host to explicitly log outgoing database connections (`[fw-audit] pg-conn:` on ports 5432 and 5440) for anomaly detection.

## 11. BACKUPS & DISASTER RECOVERY
- **Status:** Not Fully Audited.
- **Finding:** Longhorn typically provides volume snapshots, but application-level logical backups (e.g., `pg_dump`) need explicit verification.

## 12. OBSERVABILITY, MONITORING & ALERTING
- **Status:** Active.
- **Finding:** A `kubectl_wrapper.sh` script actively monitors for mutating cluster commands and sends real-time alerts to Slack/Telegram. Metrics-server and Performance Co-Pilot (`pmcd`) are running.

## 13. APPLICATION SECURITY (APPSEC) INTEGRATION
- **Status:** Lacking.
- **Finding:** No WAF (Web Application Firewall) or active AppSec sidecars (like ModSecurity) were detected on the Traefik ingress controllers. 

## 14. TLS / PKI MANAGEMENT
- **Status:** Standard.
- **Finding:** `cert-manager` is deployed and managing TLS certificates for Ingress resources (e.g., `letsencrypt-prod`). 

## 15. AUDIT LOGS & FORENSICS
- **Status:** Excellent.
- **Finding:** Host-level keystroke logging via `pam_tty_audit.so` is enabled for all SSH sessions. The `commands.log` file captures all executions reliably.

## 16. DOCUMENTATION & POLICY DRIFT
- **Status:** Significant Drift.
- **Finding:** The cluster documentation heavily emphasizes the absolute control of the Boss admins over the host kernel. However, K8s RBAC drift completely invalidates this posture by granting `cluster-admin` to multiple student accounts. The documentation is fundamentally out of sync with the cluster's actual security posture.
