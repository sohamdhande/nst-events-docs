# 06 — Defense-in-Depth, RBAC, Secrets & Token Isolation

This document details the multi-layered security architecture protecting the **NST Events** platform, its data, and the underlying college bare-metal infrastructure against internal and external threats.

---

## 1. Multi-Layer Defense-in-Depth Model

Security is enforced across seven discrete boundaries, ensuring that a compromise at any single layer cannot breach the entire system:

```text
 ┌──────────────────────────────────────────────────────────────────────────────────────────┐
 │ LAYER 1: CLOUDFLARE EDGE                                                                 │
 │ • Edge TLS Termination & Strict Origin Verification                                      │
 │ • Web Application Firewall (WAF) & Automated DDoS Scrubbing                              │
 │ • Global Anycast Routing (Hides campus server IP addresses)                              │
 └────────────────────────────────────────┬─────────────────────────────────────────────────┘
                                          │
                                          ▼
 ┌──────────────────────────────────────────────────────────────────────────────────────────┐
 │ LAYER 2: CAMPUS PERIMETER & NETWORK TUNNEL                                               │
 │ • Zero inbound ports open on campus firewall (100% outbound encrypted tunnel egress)     │
 │ • cloudflared daemon runs with minimal OS permissions on nst-n1                          │
 └────────────────────────────────────────┬─────────────────────────────────────────────────┘
                                          │
                                          ▼
 ┌──────────────────────────────────────────────────────────────────────────────────────────┐
 │ LAYER 3: KUBERNETES NETWORK ISOLATION (NetworkPolicy)                                    │
 │ • Default-deny ingress on PostgreSQL (Port 5432)                                         │
 │ • Strict pod whitelist: Only app: nst-api and app: nst-worker permitted                   │
 │ • No direct communication from dashboard or external namespaces to database             │
 └────────────────────────────────────────┬─────────────────────────────────────────────────┘
                                          │
                                          ▼
 ┌──────────────────────────────────────────────────────────────────────────────────────────┐
 │ LAYER 4: CONTAINER & POD HARDENING                                                       │
 │ • automountServiceAccountToken: false (Prevents pod breakout token harvesting)           │
 │ • Non-root execution: Multi-stage containers drop privileges to system user node         │
 │ • Lean production images (no TypeScript compiler or build tools in runner containers)    │
 └────────────────────────────────────────┬─────────────────────────────────────────────────┘
                                          │
                                          ▼
 ┌──────────────────────────────────────────────────────────────────────────────────────────┐
 │ LAYER 5: LEAST-PRIVILEGE KUBERNETES RBAC                                                 │
 │ • Dedicated ServiceAccount for deployments: github-actions-deployer                      │
 │ • Scoped Role restricted strictly to nst-events namespace                                │
 │ • Denied access to read sensitive secrets and prohibited from cluster-admin privileges   │
 └────────────────────────────────────────┬─────────────────────────────────────────────────┘
                                          │
                                          ▼
 ┌──────────────────────────────────────────────────────────────────────────────────────────┐
 │ LAYER 6: APPLICATION-LEVEL SECURITY & TOKEN HANDLING                                     │
 │ • No access tokens stored in browser localStorage (Eliminates XSS token harvesting)      │
 │ • Stateless JWTs in HTTP-Only, Signed, SameSite cookies                                  │
 │ • Hardened trust proxy CIDR whitelist prevents X-Forwarded-For IP spoofing               │
 │ • Global and per-route rate limiting via Express rate-limit                              │
 └────────────────────────────────────────┬─────────────────────────────────────────────────┘
                                          │
                                          ▼
 ┌──────────────────────────────────────────────────────────────────────────────────────────┐
 │ LAYER 7: ADMINISTRATIVE ACCESS & BOOTSTRAP INTEGRITY                                     │
 │ • Zero in-app self-service admin promotion or invite APIs                                │
 │ • Promotion to PLATFORM_ADMIN requires physical/terminal database CLI execution          │
 └──────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Secrets Architecture & Ingestion

Sensitive credentials are strictly decoupled from container images and source code. All secrets are managed through native Kubernetes `Secret` resources in the `nst-events` namespace.

### The Four Core Secret Objects

#### 1. Database Admin Credentials (`nst-db-admin-secrets`)
Used by PostgreSQL for initialization and authentication:
- `POSTGRES_USER`: Database superuser username (`postgres`).
- `POSTGRES_PASSWORD`: High-entropy database password.
- `POSTGRES_DB`: Primary database name (`nst_events`).

#### 2. API Runtime Secrets (`nst-api-secrets`)
Injected into the `nst-api` pods via `envFrom`:
- `DATABASE_URL`: Full PostgreSQL connection string (`postgresql://...`).
- `JWT_SECRET`: 256-bit cryptographic key used to sign and verify user JWT tokens.
- `COOKIE_SECRET`: Cryptographic salt for signing HTTP cookies.
- `GOOGLE_CLIENT_ID` & `GOOGLE_CLIENT_SECRET`: OAuth 2.0 credentials for student Google Workspace login.
- `ALLOWED_EMAIL_DOMAINS`: Institutional email domain whitelist (`adypu.edu.in,newtonschool.co`).
- `ALLOWED_ORIGINS`: CORS whitelist (`https://events.nstsdc.org`).

#### 3. Worker Runtime Secrets (`nst-worker-secrets`)
Injected into the `nst-worker` pod via `envFrom`:
- `WORKER_DATABASE_URL`: Dedicated database connection string for background job processing.
- `EXPO_ACCESS_TOKEN`: Bearer token for authenticating push notification batches with Expo's push servers.

#### 4. Off-Cluster Backup Secrets (`nst-backup-secrets`)
Used by the automated backup CronJob:
- `R2_ACCOUNT_ID`: Cloudflare account identifier.
- `R2_ACCESS_KEY_ID` & `R2_SECRET_ACCESS_KEY`: S3-compatible API credentials for Cloudflare R2 bucket writes.
- `R2_BUCKET_NAME`: Target backup bucket (`nst-events-db-backups`).

---

## 3. Pod & Kubernetes Hardening

### Token Auto-Mount Prevention
By default, Kubernetes mounts a service account token into every pod at `/var/run/secrets/kubernetes.io/serviceaccount/token`. If an application has an arbitrary file read vulnerability, an attacker could steal this token and query the K3s API server.
- All `nst-events` deployments explicitly declare:
  ```yaml
  automountServiceAccountToken: false
  ```
  The API, Dashboard, and Worker pods cannot access Kubernetes APIs.

### Non-Root Container Execution
All application Dockerfiles (`docker/Dockerfile.*`) use multi-stage builds. After building assets with `pnpm`, runtime stages change file ownership and execute under the unprivileged `node` user (UID 1000).

---

## 4. Administrative Security: The Bootstrap Rule

To eliminate privilege escalation vulnerabilities (e.g., IDORs in user invite endpoints, privilege escalation bugs), **NST Events has NO in-app admin invite or promotion endpoints**:
- A student or club lead cannot invite a user to become a `PLATFORM_ADMIN`.
- No self-service admin promotion API exists.
- Promotion to `PLATFORM_ADMIN` is **deliberately manual, one-time, and out-of-band**:

```bash
# Operator establishes a local secure tunnel to the cluster database
kubectl port-forward -n nst-events svc/nst-postgres-service 5432:5432

# Operator executes the standalone bootstrap utility
DATABASE_URL="postgresql://postgres:<PASSWORD>@localhost:5432/nst_events?schema=public" \
pnpm --filter @nst/database bootstrap-admin -- coordinator@adypu.edu.in
```

### Safety Guarantees of the Bootstrap Script:
- **Prerequisite**: The target user must have authenticated via Google OAuth at least once (guaranteeing that their Google email is verified and their user record exists).
- **Confirmation Guard**: Requires the operator to type `yes` before mutating the database.
- **Strict Scope**: Mutates `global_role` only. Does not touch club roles, student profiles, or data.
- **Isolation**: The bootstrap script is never packaged into the web application or API runtime containers.

---

## 5. Environment Isolation: Staging vs. Production

The cluster enforces logical multi-tenancy using Kubernetes namespaces:

| Security Domain | Production (`nst-events-prod` / `nst-events`) | Staging (`nst-events-staging`) | Isolation Guarantee |
| :--- | :--- | :--- | :--- |
| **Kubernetes Namespace** | `nst-events` | `nst-events-staging` | Kernel-level cgroup and namespace isolation. |
| **Persistent Volume** | `postgres-data-prod` (10Gi) | `postgres-data-staging` (5Gi) | PVCs cannot be cross-mounted across namespaces. |
| **NetworkPolicy** | Rejects all non-prod traffic | Rejects all non-staging traffic | Staging pods cannot open TCP sockets to production PostgreSQL. |
| **Routing Domain** | `events.nstsdc.org`<br>`api.nst-events.nstsdc.org` | `staging-events.nstsdc.org`<br>`staging-api.nst-events.nstsdc.org` | Separate Cloudflare routes and Traefik backends. |
| **Backup Storage** | Write access to `s3://nst-backups/prod/` | Write access to `s3://nst-backups/staging/` | Separate IAM API keys for off-cluster storage buckets. |

---

## 6. Cluster Authority, Host Hardening & Incident Defense

To guarantee cluster integrity and eliminate plausible deniability or unauthorized database manipulation, the bare-metal environment enforces host-level authority boundaries and out-of-band telemetry:

### 1. Host-Level Access & SSH Authority Perimeter
- **Enforced SSH Public Key Authentication**: Password authentication is disabled for `clusteradmin`. All administrative access requires a registered, cryptographic SSH key (e.g. `ed25519`).
- **Dynamic Identity Attribution**: Every connection is matched to the specific public key comment and verified email stored in `authorized_keys`. Anonymous logins via shared passwords are prohibited.
- **Kernel-Level Immutable Auditing**: Linux `auditd` rules are enforced with the immutable flag (`-e 2`). The kernel records all process executions (`execve`), access to credential databases (`/etc/shadow`, `/etc/passwd`), and modifications to `/home/clusteradmin/.ssh/authorized_keys` directly to `/var/log/audit/audit.log`.

### 2. Developer Access & Namespace Isolation (Zero-Cluster-Admin Policy)
- **Host Privilege Boundary**: Developers are strictly excluded from the host OS `sudo` group on `nst-n1`. Host root access is restricted strictly to designated cluster infrastructure administrators.
- **Scoped Kubernetes RBAC**: Developers needing access to specific workloads (e.g. student projects or standalone deployments) receive dedicated, namespace-scoped `Role` and `RoleBinding` credentials.
- **Namespace Confinement**: Developer credentials possess zero permissions in the `nst-events` production namespace, cannot read database secrets, and cannot alter cluster-wide `StorageClass`, `Ingress`, or `NetworkPolicy` resources.

### 3. Database Pod Protection & Tripwire Defenses
- **NetworkPolicy Enforcement**: Direct TCP access to `nst-postgres` (Port 5432) is blocked from all external namespaces and unauthorized pods via `postgres-network-policy`.
- **Database Port-Forward Tripwires**: Background daemons (`kube-node-health`) continuously monitor for unauthorized `kubectl port-forward` commands, interactive root subshell breakouts (`sudo -i`, `sudo bash`), and destructive SQL keywords (`DROP DATABASE`, `prisma migrate reset`).
- **Automated Escalation**: Any process matching destructive signatures is captured alongside full process tree metrics (`ps auxf`, active network sockets) and transmitted out-of-band to administrators.

### 4. Real-Time Out-of-Band Telemetry
- **PAM Session Telemetry**: Pluggable Authentication Module (PAM) integration transmits real-time session notifications upon authentication, detailing the authenticated key identity, target account, session ID, and timestamp.
- **Command & Namespace Telemetry**: Kubernetes CLI wrappers capture the executing user identity, target namespace (`-n` / `--namespace`), and executed command arguments, streaming audit events asynchronously without introducing execution latency.

