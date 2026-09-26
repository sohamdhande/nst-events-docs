# 07 — Operational Runbooks, Disaster Recovery & Troubleshooting

This document serves as the master operations manual for **NST Events**. It defines the backup architecture, disaster recovery runbooks, administrative operational procedures, and a systematic 7-step troubleshooting flowchart.

---

## 1. The Four Tiers of Data Protection

To prevent catastrophic assumptions, cluster operators must understand the distinction between high availability and disaster recovery:

```text
[ Tier 1: Longhorn Storage Replication ] ──> Protects ONLY against physical node or disk failure.
                                              (Mirrors corruptions and DROP TABLE in milliseconds).

[ Tier 2: Longhorn Volume Snapshots ]    ──> Point-in-time copy-on-write diffs stored on the SAME disks.
                                              (Does NOT survive cluster loss, disk death, or volume deletion).

[ Tier 3: PostgreSQL Logical Dump ]       ──> Application-level SQL snapshot (pg_dump -Fc).
                                              (Immune to storage engine bugs; can restore to any PG instance).

[ Tier 4: Off-Cluster DR Backup ]         ──> Encrypted dump streamed to INDEPENDENT external cloud storage.
                                              (Survives complete campus datacenter destruction).
```

- **Replication is NOT a backup**: If a developer executes `DROP TABLE users;` or an invalid migration runs, Longhorn replicates the deletion to all three nodes within 2 milliseconds.
- **True Disaster Recovery Requires Tier 4**: An independent logical archive stored physically outside the college network.

---

## 2. Automated Daily Backup Strategy (`nst-postgres-backup`)

Database backups are automated via a native Kubernetes `CronJob` (`infrastructure/kubernetes/backup-cronjob.yaml`):

### Backup Specifications
- **Schedule**: `0 2 * * *` (Daily at 02:00 UTC / 07:30 IST off-peak).
- **Concurrency Policy**: `Forbid` (prevents overlapping backup runs if a previous run is delayed).
- **Execution Container**: `postgres:16-alpine` running on the internal pod network.
- **Backup Tooling**:
  ```bash
  pg_dump -h "$PGHOST" -U "$PGUSER" -d "$PGDATABASE" | gzip > "$BACKUP_FILE"
  ```
- **Destination**: **Cloudflare R2** object storage (S3-compatible, zero-egress fees, native fit for Cloudflare architecture).
- **Credentials**: Injected securely via Kubernetes secret `nst-backup-secrets`.

### Grandfather-Father-Son Retention Schedule
- **Daily Dumps**: Retained for **7 days**.
- **Weekly Dumps**: Retained for **4 weeks**.
- **Monthly Dumps**: Retained for **6 months**.
- **Annual Dumps**: Retained for **2 academic years**.

---

## 3. Disaster Recovery Matrix

The table below documents survival, automated response, and recovery runbooks across all nine failure modes:

| Failure Mode | What Survives | What Is Lost | Recovery Method | Target RPO | Target RTO | Recovery Type |
| :--- | :--- | :--- | :--- | :--- | :--- | :--- |
| **1. PostgreSQL Pod Crash** | Persistent volume, all data, all 3 Longhorn replicas. | Ephemeral container memory, uncommitted transactions. | K3s automatically restarts the pod; re-attaches existing volume. | **0 seconds** | **< 30 seconds** | **Automated** |
| **2. Physical Node Failure** | Longhorn replicas on other 5 physical storage nodes. | The crashed machine and the pod running on it. | K3s reschedules the StatefulSet pod to a healthy node; Longhorn CSI re-attaches volume. | **0 seconds** | **< 3 minutes** | **Automated** |
| **3. Single Disk Failure** | 2 remaining healthy replicas; active PostgreSQL pod. | 1 replica on the degraded physical drive. | Database remains 100% online on 2 replicas. Longhorn auto-rebuilds the 3rd replica after 600s. | **0 seconds** | **0 downtime** | **Automated** |
| **4. Database Corruption** | Raw storage volume, unaffected tables. | Corrupted tables, invalid indexes, or broken WAL stream. | Spin up fresh database pod; restore latest verified logical `pg_dump` from Cloudflare R2. | **< 24 hours** | **< 45 minutes** | **Manual** |
| **5. Accidental Table Drop (`DROP TABLE`)** | Storage volumes (holding the dropped state). | Dropped data, dropped table. | Spin up temporary database in staging; restore yesterday's backup; extract dropped table; re-import. | **< 24 hours** | **< 1 hour** | **Manual** |
| **6. Namespace Deletion (`kubectl delete ns nst-events`)** | Off-cluster R2 backups, other cluster namespaces. | Deployments, Services, Secrets, Ingress, and PVCs. | Re-apply manifests via GitOps / CI/CD pipeline; restore data from latest off-cluster R2 backup. | **< 24 hours** | **< 1.5 hours** | **Manual** |
| **7. Complete K3s Cluster Failure** | Off-cluster R2 backups, container images in GHCR. | All nodes, K3s etcd, local Longhorn storage volumes. | Re-install K3s, Longhorn, cert-manager; apply Kubernetes manifests; restore data from R2. | **< 24 hours** | **< 4 hours** | **Manual** |
| **8. Longhorn Storage Engine Failure** | Off-cluster R2 backups, application code. | All local volume replicas and Longhorn metadata. | Re-install Longhorn or temporarily switch to `local-path`; restore PostgreSQL from R2 backup. | **< 24 hours** | **< 2 hours** | **Manual** |
| **9. Total Campus Outage (Power / Datacenter Down)** | Off-cluster R2 backups, GitHub repo, Cloudflare DNS. | On-premise servers and campus power/network. | Repoint Cloudflare DNS to a temporary cloud instance (AWS/DO); deploy images; restore R2 backup. | **< 24 hours** | **< 4 hours** | **Manual** |

---

## 4. Disaster Recovery Restoration Runbook

In the event of database corruption or cluster re-provisioning, follow this exact restoration sequence:

### Step 1: Verify PostgreSQL Pod Is Ready
```bash
kubectl wait --for=condition=ready pod/nst-postgres-0 -n nst-events --timeout=120s
```

### Step 2: Download & Extract Latest Backup from Cloudflare R2
```bash
# Using AWS CLI configured for Cloudflare R2 endpoint
aws s3 cp s3://nst-events-db-backups/latest.sql.gz /tmp/restore_backup.sql.gz \
  --endpoint-url "https://${R2_ACCOUNT_ID}.r2.cloudflarestorage.com"

# Decompress
gunzip /tmp/restore_backup.sql.gz
```

### Step 3: Copy Dump into Database Pod & Execute `pg_restore`
```bash
# Copy file into container scratch space
kubectl cp /tmp/restore_backup.sql nst-events/nst-postgres-0:/tmp/restore_backup.sql

# Execute restore with clean replacement
kubectl exec -it nst-postgres-0 -n nst-events -- \
  psql -U postgres -d nst_events -f /tmp/restore_backup.sql

# Cleanup pod scratch
kubectl exec -it nst-postgres-0 -n nst-events -- rm -f /tmp/restore_backup.sql
```

### Step 4: Validate Application Pods Reconnect
```bash
kubectl rollout restart deployment/nst-api -n nst-events
kubectl rollout status deployment/nst-api -n nst-events
```

---

## 5. Routine Operational Runbooks

### Runbook A: Bootstrapping a Platform Administrator
Because no in-app self-service admin promotion exists, promote authorized operators out-of-band:

1. Ensure the administrator has signed in with their institutional Google account (`@adypu.edu.in`) at least once.
2. Port-forward the cluster database to your local workstation:
   ```bash
   kubectl port-forward -n nst-events svc/nst-postgres-service 5432:5432
   ```
3. Run the bootstrap script from the repository root:
   ```bash
   DATABASE_URL="postgresql://postgres:<PASSWORD>@localhost:5432/nst_events?schema=public" \
   pnpm --filter @nst/database bootstrap-admin -- coordinator@adypu.edu.in
   ```
4. When prompted, type `yes` to confirm the promotion.

### Runbook B: Rotating the CI/CD Deployer Token
If node `nst-n6` is serviced or credentials require periodic rotation:

1. Delete the active token secret from the cluster (immediately invalidates existing token):
   ```bash
   kubectl delete secret github-actions-deployer-token -n nst-events
   ```
2. Generate a fresh static service account token:
   ```bash
   cat <<EOF | kubectl apply -f -
   apiVersion: v1
   kind: Secret
   metadata:
     name: github-actions-deployer-token
     namespace: nst-events
     annotations:
       kubernetes.io/service-account.name: github-actions-deployer
   type: kubernetes.io/service-account-token
   EOF
   ```
3. Extract the new token and update `/home/github-runner/.kube/config` on `nst-n6`:
   ```bash
   kubectl get secret github-actions-deployer-token -n nst-events \
     -o jsonpath="{.data.token}" | base64 --decode
   ```

---

## 6. Systematic 7-Step Troubleshooting Flowchart

If the NST Events platform is unreachable or reporting errors, diagnose in this exact order to isolate the root cause rapidly:

```text
[ Step 1: DNS Resolution ]
  Does ping events.nstsdc.org resolve to Cloudflare Anycast IPs?
    NO  ──> Cloudflare DNS outage or misconfigured domain CNAME record.
    YES ──> Proceed to Step 2.

[ Step 2: Cloudflare Edge & Tunnel ]
  Is systemctl status cloudflared active on nst-n1? Check journalctl -u cloudflared.
    NO  ──> Node nst-n1 is down or cloudflared daemon crashed. Restart daemon.
    YES ──> Proceed to Step 3.

[ Step 3: TLS & Origin Certificates ]
  Is the browser showing an SSL error? Check kubectl get certificate -n nst-events.
    NO  ──> cert-manager failed challenge. Check Traefik HTTP-01 challenge routing.
    YES ──> Proceed to Step 4.

[ Step 4: Traefik Ingress Controller ]
  Check Traefik logs. Does kubectl get ingress -n nst-events have valid hosts?
    NO  ──> Host header mismatch or missing path prefix in ingress.yaml.
    YES ──> Proceed to Step 5.

[ Step 5: Kubernetes Services & Endpoints ]
  Run kubectl get endpoints -n nst-events. Do services have healthy target IPs?
    NO  ──> Selector mismatch or pods are failing readiness probes.
    YES ──> Proceed to Step 6.

[ Step 6: Application Pod Health ]
  Run kubectl get pods -n nst-events. Are pods in CrashLoopBackOff or OOMKilled?
    YES ──> Inspect kubectl logs <pod-name> -n nst-events and describe pod.
    NO  ──> Proceed to Step 7.

[ Step 7: Database & Storage Layer ]
  Is PostgreSQL accepting queries? Check kubectl exec nst-postgres-0 -- pg_isready.
    NO  ──> Longhorn volume detached or disk full. Check kubectl get sc, pvc, and longhorn UI.
```
