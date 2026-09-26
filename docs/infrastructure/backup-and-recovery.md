# PostgreSQL Backup & Recovery Strategy

This document outlines the backup, retention, and verification mechanisms for the `nst-events` PostgreSQL database.

---

## 1. Backup Status & Overview

| Category                              | Definition                                                                                                                                                                                                                                    |
| :------------------------------------ | :-------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **CURRENT STATE (REPO)**        | No backup manifests, scripts, or CronJobs exist in the`nst-events` repository.                                                                                                                                                              |
| **VERIFIED CLUSTER CAPABILITY** | Longhorn is running without any configured`backup-target` (`settings.longhorn.io/backup-target` is unconfigured; zero recurring backup jobs exist). No cluster-wide NFS, NAS, or MinIO backup targets are available on the cluster nodes. |
| **INTENDED ARCHITECTURE**       | Scheduled Kubernetes`CronJob` running automated `pg_dump -Fc`, encrypting with AES-256, generating SHA-256 checksums, and streaming directly to an independent, off-cluster S3-compatible bucket (e.g., Cloudflare R2 / AWS S3).          |
| **MISSING CONFIGURATION**       | Off-cluster storage bucket credentials; backup`CronJob` manifest; restoration testing script.                                                                                                                                               |
| **REQUIRED ACTION**             | Provision an off-cluster S3/R2 backup bucket; implement`backup-cronjob.yaml` in the deployment pipeline; automate weekly restore validation.                                                                                                |

---

## 2. Fundamental Distinctions: What is NOT a Backup

To avoid catastrophic assumptions, the following 4 tiers of data protection must never be treated as interchangeable:

```text
[ Tier 1: Longhorn Replication ] ──> Protects ONLY against physical node/disk loss.
                                      (Synchronously mirrors bad data, corruptions, and DROP TABLE).

[ Tier 2: Longhorn Snapshots ]    ──> Point-in-time copy-on-write diffs stored on the SAME cluster disks.
                                      (Does NOT survive disk loss, cluster loss, or volume deletion).

[ Tier 3: PG Logical Dump ]       ──> Application-level SQL/data snapshot (pg_dump -Fc).
                                      (Immune to storage engine corruption; can restore to any PG instance).

[ Tier 4: Off-Cluster DR Backup ] ──> Encrypted logical dump transmitted to INDEPENDENT external storage.
                                      (Survives complete cluster, storage, and datacenter destruction).
```

1. **Longhorn Replication is NOT a backup**: 3-way replication guarantees high availability during node crashes. However, if a developer accidentally drops a table or a migration corrupts data, Longhorn replicates that corruption to all 3 nodes within milliseconds.
2. **Longhorn Snapshots are NOT independent backups**: Snapshots reside on the exact same physical node disks as the active volume. If the cluster nodes fail or Longhorn's metadata is corrupted, snapshots are lost.
3. **True Disaster Recovery requires Tier 4**: An encrypted file stored physically outside the K3s cluster.

---

## 3. Actual Available Backup Destination (Cluster Audit)

A live inspection of the college K3s cluster confirmed:

* **No NFS / NAS Mounts**: Host mount inspection (`mount | grep -i -E 'nfs|cifs|smb'`) revealed zero networked storage volumes attached to the nodes.
* **No Cluster-Wide S3/MinIO**: The `minio` service in the `default` namespace has `0` endpoints (inactive). MinIO in the `wearwise` namespace is an isolated project-specific deployment.
* **Longhorn Backup Target**: `settings.longhorn.io backup-target` is empty.

### Conclusion & Required Decision

**There is currently NO verified, independent backup destination on the college cluster.**
Backups cannot be stored on the local nodes or Longhorn volumes. An external S3-compatible target—such as **Cloudflare R2** (zero-egress fees, native fit for the Cloudflare setup) or a dedicated off-cluster college backup server—must be provisioned before production data is loaded.

---

## 4. PostgreSQL Logical Backup Strategy

### Tooling & Format

* **Tool**: `pg_dump` executed from a lightweight container sharing network access to `nst-postgres-service`.
* **Format**: PostgreSQL Custom Archive (`-Fc`).
  - Native compression.
  - Allows flexible, granular restoration using `pg_restore` (single table, specific schema, or entire database).
  - Preserves spatial geometry tables and PostGIS indices.
* **Global Roles & Passwords**: Since user roles are managed via Prisma migrations and initialization scripts, cluster-wide `pg_dumpall --globals-only` is secondary, but a quarterly globals dump should be captured.

### Frequency & Schedule

* **Daily Full Dump**: Scheduled via Kubernetes `CronJob` at off-peak hours (02:00 UTC / 07:30 IST).
* **Pre-Deployment / Pre-Event Snapshots**: Ad-hoc manual trigger before running database migrations or major college-wide events.

### Target Objectives (Labeled as Targets until validated)

* **Target RPO (Recovery Point Objective)**: **24 hours** for standard daily dumps. (Target RPO can be reduced to 1 hour if WAL archiving / continuous archiving is added later).
* **Target RTO (Recovery Time Objective)**: **< 30 minutes** to provision a fresh PostgreSQL pod, restore the 10Gi logical dump, and re-point the API.

### Retention Schedule

Backups must be tagged and automatically pruned according to the grandfather-father-son scheme:

* **Daily Dumps**: Retain for **7 days**.
* **Weekly Dumps**: Retain for **4 weeks**.
* **Monthly Dumps**: Retain for **6 months**.
* **Annual Dumps**: Retain end-of-academic-year dumps for **2 years**.

### Encryption & Security

* **In-Flight**: TLS 1.3 enforced for transmission to the S3 bucket.
* **At-Rest**: Encrypted using AES-256 (via client-side GPG/Age encryption before upload, or server-side S3 encryption `AES256`/`aws:kms`).
* **Credentials**: S3 access keys and encryption passphrases must be injected via Kubernetes Secrets (`nst-backup-secrets`), never committed to git.

---

## 5. Verification & Restore Testing

A backup that has never been restored is not a verified backup.

### Automated Integrity Verification

Every backup run must:

1. Stream the compressed archive directly to an in-memory or ephemeral scratch space (`emptyDir`).
2. Calculate the **SHA-256 checksum**.
3. Verify `pg_dump` exited with code `0`.
4. Upload both the archive (`.dump.enc`) and the checksum (`.sha256`) to the backup bucket.

### Staging Restore Testing

* **Weekly Automated Verification**: A lightweight CronJob restores the latest production backup into an ephemeral database in the `nst-events-staging` namespace.
* **Data Sanitization Rule**: Before staging developers or automated test suites can access restored production data, a sanitization script must execute:
  - Mask student email addresses (`user_<id>@example.com`).
  - Purge push notification tokens (`DELETE FROM push_tokens`).
  - Purge refresh token hashes (`DELETE FROM refresh_tokens`).
  - Mask student names and phone numbers.
