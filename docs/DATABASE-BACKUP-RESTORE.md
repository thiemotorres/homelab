# PostgreSQL Backup and Restore

This document describes the automated backup system for PostgreSQL and how to perform disaster recovery.

## Overview

The homelab uses an automated backup system that:
- Creates daily PostgreSQL dumps at 2 AM
- Compresses backups with gzip
- Uploads to Cloudflare R2 (S3-compatible storage)
- Retains backups for 7 days
- Sends Discord notifications on success/failure

## Architecture

```
┌─────────────────┐
│   PostgreSQL    │
│  (Infrastructure│
│    Namespace)   │
└────────┬────────┘
         │
         │ pg_dumpall
         ▼
┌─────────────────┐
│  Backup CronJob │
│  (Daily 2 AM)   │
└────────┬────────┘
         │
         │ rclone
         ▼
┌─────────────────┐      ┌─────────────────┐
│  Cloudflare R2  │◄────►│     Discord     │
│  (Object Store) │      │  (Notifications)│
└─────────────────┘      └─────────────────┘
```

## Files

- **CronJob**: `infrastructure/postgres/backup-cronjob.yaml`
- **Manual Backup**: `infrastructure/postgres/manual-backup-job.yaml`
- **Manual Restore**: `infrastructure/postgres/manual-restore-job.yaml`
- **Secrets**:
  - `postgres-credentials` - Database credentials
  - `r2-backup-credentials` - Cloudflare R2 access keys
  - `discord-webhook` - Discord notification URL

---

## Automated Daily Backups

### Schedule
- **Time**: 2:00 AM daily (UTC)
- **Cron**: `0 2 * * *`

### Backup Process

#### Step 1: Database Dump (Init Container)
```bash
# Creates compressed dump of ALL databases
pg_dumpall -h postgres-service -U $POSTGRES_USER | gzip > /backup/postgres_backup_YYYYMMDD_HHMMSS.sql.gz
```

**What gets backed up:**
- All databases (postgres, n8n, gork)
- All schemas and tables
- All users and permissions
- Database configurations

**Excluded from backups:**
- `app_auth` table (preserved cluster-specific data)

#### Step 2: Upload to R2 (Main Container)
```bash
# Configure rclone for Cloudflare R2
rclone copy /backup/postgres_backup_*.sql.gz r2:${BUCKET_NAME}/

# Cleanup old backups (older than 7 days)
rclone delete r2:${BUCKET_NAME}/ --min-age 7d --include "postgres_backup_*.sql.gz"
```

#### Step 3: Discord Notification
On success:
```json
{
  "title": "✅ PostgreSQL Backup erfolgreich",
  "fields": [
    {"name": "📁 Datei", "value": "postgres_backup_20251026_020000.sql.gz"},
    {"name": "💾 Größe", "value": "45.23 MB"},
    {"name": "📊 Gesamt Backups", "value": "7"},
    {"name": "🗓️ Retention", "value": "7 Tage"}
  ]
}
```

On failure:
```json
{
  "title": "❌ PostgreSQL Backup fehlgeschlagen",
  "description": "Upload zu R2 ist fehlgeschlagen. Bitte Logs prüfen!"
}
```

### View Backup Logs

```bash
# View CronJob schedule
kubectl get cronjob postgres-backup -n infrastructure

# List recent backup jobs
kubectl get jobs -n infrastructure | grep postgres-backup

# View logs from latest backup
LATEST_JOB=$(kubectl get jobs -n infrastructure --sort-by=.metadata.creationTimestamp | grep postgres-backup | tail -1 | awk '{print $1}')
kubectl logs job/$LATEST_JOB -n infrastructure -c upload-to-r2
```

### List Available Backups

```bash
# Create a temporary pod to list backups
kubectl run rclone-temp --rm -it --restart=Never \
  --image=rclone/rclone:1.68 \
  --namespace=infrastructure \
  --overrides='
{
  "spec": {
    "containers": [{
      "name": "rclone-temp",
      "image": "rclone/rclone:1.68",
      "command": ["/bin/sh"],
      "stdin": true,
      "tty": true,
      "envFrom": [{"secretRef": {"name": "r2-backup-credentials"}}]
    }]
  }
}' -- sh

# Inside the pod, configure rclone and list backups
cat > ~/.config/rclone/rclone.conf <<EOF
[r2]
type = s3
provider = Cloudflare
access_key_id = $AWS_ACCESS_KEY_ID
secret_access_key = $AWS_SECRET_ACCESS_KEY
endpoint = $S3_ENDPOINT
acl = private
EOF

rclone ls r2:$BUCKET_NAME/ --include "postgres_backup_*.sql.gz"
```

---

## Manual Backup

Use manual backup before major changes or migrations.

### Run Manual Backup

```bash
# Apply the manual backup job
kubectl apply -f infrastructure/postgres/manual-backup-job.yaml

# Monitor progress
kubectl get jobs -n infrastructure
kubectl logs -f job/postgres-backup-test -n infrastructure -c upload-to-r2

# Verify backup was created
# (Use rclone-temp pod method above to list backups)

# Cleanup
kubectl delete job postgres-backup-test -n infrastructure
```

---

## Disaster Recovery (Restore)

### Prerequisites

**⚠️ WARNING**: Restore will **overwrite** the current database. Ensure you understand the implications.

**Before restoring:**
1. Take a manual backup of current state (if database is accessible)
2. Scale down applications using the database
3. Identify which backup to restore

### Step 1: Scale Down Applications

```bash
# Scale down all apps using PostgreSQL
kubectl scale deployment gork-backend -n gork --replicas=0
kubectl scale deployment n8n -n n8n --replicas=0

# Verify pods are terminated
kubectl get pods -n gork
kubectl get pods -n n8n
```

### Step 2: Choose Backup to Restore

List available backups (see "List Available Backups" section above) and note the filename.

### Step 3: Run Restore Job

```bash
# Edit manual-restore-job.yaml to specify backup file (optional)
# Uncomment and set RESTORE_FILENAME environment variable:
# - name: RESTORE_FILENAME
#   value: "postgres_backup_20251026_020000.sql.gz"

# If not specified, the latest backup will be used

# Apply the restore job
kubectl create -f infrastructure/postgres/manual-restore-job.yaml

# Monitor progress
RESTORE_JOB=$(kubectl get jobs -n infrastructure | grep postgres-restore | tail -1 | awk '{print $1}')
kubectl logs -f job/$RESTORE_JOB -n infrastructure

# Expected output:
# Available backups:
# ...
# Downloading backup: postgres_backup_20251026_020000.sql.gz
# Installing PostgreSQL client...
# Restoring database...
# Restore completed successfully!
```

### Step 4: Verify Restore

```bash
# Connect to PostgreSQL
kubectl exec -it deployment/postgres -n infrastructure -- psql -U $POSTGRES_USER

# Check databases exist
\l

# Check table counts
\c n8n
SELECT COUNT(*) FROM workflow_entity;

\c postgres
SELECT COUNT(*) FROM <your_table>;

# Exit
\q
```

### Step 5: Restart Applications

```bash
# Scale applications back up
kubectl scale deployment gork-backend -n gork --replicas=1
kubectl scale deployment n8n -n n8n --replicas=1

# Verify applications are healthy
kubectl get pods -n gork
kubectl get pods -n n8n

# Test applications
curl https://gork.thiemo.click
curl https://n8n.thiemo.click
```

### Step 6: Cleanup

```bash
# Delete the restore job
kubectl delete job $RESTORE_JOB -n infrastructure
```

---

## Restore Scenarios

### Scenario 1: Complete Data Loss

**Symptoms**: PostgreSQL data volume corrupted/lost

**Steps**:
1. Delete PVC and recreate: `kubectl delete pvc postgres-pvc -n infrastructure`
2. Restart PostgreSQL deployment
3. Follow full restore procedure above

### Scenario 2: Accidental Data Deletion

**Symptoms**: Specific tables or data accidentally deleted

**Steps**:
1. Scale down only affected application
2. Restore to a specific point-in-time backup
3. Use `pg_restore` with `--clean --if-exists` for selective restore

### Scenario 3: Point-in-Time Recovery

**Symptoms**: Need to restore to specific date/time

**Steps**:
1. List backups and identify closest backup to desired time
2. Set `RESTORE_FILENAME` in manual-restore-job.yaml
3. Follow standard restore procedure

### Scenario 4: Migration to New Cluster

**Steps**:
1. Run manual backup on old cluster
2. Download backup from R2
3. Upload to new R2 bucket (or use same bucket)
4. Configure new cluster with R2 credentials
5. Run restore job on new cluster

---

## Backup Configuration

### Modify Backup Schedule

Edit `infrastructure/postgres/backup-cronjob.yaml`:

```yaml
spec:
  schedule: "0 2 * * *"  # Change to desired cron schedule
  # Examples:
  # "0 */6 * * *"    - Every 6 hours
  # "0 0,12 * * *"   - Daily at midnight and noon
  # "0 3 * * 0"      - Weekly on Sunday at 3 AM
```

### Modify Retention Policy

Edit `infrastructure/postgres/backup-cronjob.yaml`:

```yaml
env:
- name: RETENTION_DAYS
  value: "7"  # Change to desired retention in days
```

### Change Backup Location

Update R2 credentials in `infrastructure/postgres/sealed-secrets/r2-backup-credentials-sealed.yaml`:

```yaml
# Sealed secret containing:
# - AWS_ACCESS_KEY_ID
# - AWS_SECRET_ACCESS_KEY
# - S3_ENDPOINT (e.g., https://xxxx.r2.cloudflarestorage.com)
# - BUCKET_NAME
```

---

## Monitoring and Alerts

### Successful Backups

Successful backups send green Discord notifications with:
- Backup filename
- File size
- Total number of backups
- Retention period

### Failed Backups

Failed backups send red Discord notifications immediately.

**Common failure reasons:**
1. **Database connectivity**: PostgreSQL unreachable
2. **R2 connectivity**: Network issues or invalid credentials
3. **Disk space**: Insufficient space for backup file
4. **Permission issues**: RBAC or secret access problems

### Check Backup Health

```bash
# View CronJob status
kubectl get cronjob postgres-backup -n infrastructure

# Check for failed jobs
kubectl get jobs -n infrastructure | grep postgres-backup | grep -v Complete

# View recent job history
kubectl describe cronjob postgres-backup -n infrastructure
```

---

## Troubleshooting

### Backup Job Failing

**Check logs:**
```bash
FAILED_JOB=$(kubectl get jobs -n infrastructure | grep postgres-backup | grep -v Complete | tail -1 | awk '{print $1}')
kubectl logs job/$FAILED_JOB -n infrastructure -c postgres-dump
kubectl logs job/$FAILED_JOB -n infrastructure -c upload-to-r2
```

**Common issues:**

1. **Database connection refused**
   ```
   Solution: Check PostgreSQL is running
   kubectl get pods -n infrastructure | grep postgres
   ```

2. **R2 authentication failed**
   ```
   Solution: Verify R2 credentials
   kubectl get secret r2-backup-credentials -n infrastructure -o yaml
   ```

3. **Discord webhook failed**
   ```
   Solution: Non-critical, check webhook URL
   kubectl get secret discord-webhook -n infrastructure -o yaml
   ```

### Restore Job Failing

**Check logs:**
```bash
kubectl logs job/$RESTORE_JOB -n infrastructure
```

**Common issues:**

1. **Backup file not found**
   ```
   Solution: Verify backup exists in R2, check RESTORE_FILENAME
   ```

2. **Database already has data**
   ```
   Solution: pg_restore will attempt to drop/recreate
   Use --clean flag (already included in restore job)
   ```

3. **Permission denied**
   ```
   Solution: Verify ServiceAccount has access to secrets
   kubectl auth can-i get secret/r2-backup-credentials \
     --as=system:serviceaccount:infrastructure:postgres-backup-sa \
     -n infrastructure
   ```

### Manual Intervention

If automated restore fails, manual restore:

```bash
# 1. Copy backup from R2 to local machine
# (Use rclone locally or download from R2 console)

# 2. Copy to PostgreSQL pod
kubectl cp postgres_backup_20251026_020000.sql.gz \
  infrastructure/postgres-xxxxx:/tmp/backup.sql.gz

# 3. Restore manually
kubectl exec -it deployment/postgres -n infrastructure -- bash
gunzip -c /tmp/backup.sql.gz | psql -U $POSTGRES_USER -d postgres
```

---

## Best Practices

### Before Major Changes

1. **Run manual backup**
   ```bash
   kubectl apply -f infrastructure/postgres/manual-backup-job.yaml
   ```

2. **Verify backup success**
   ```bash
   # Check job completed
   kubectl get jobs -n infrastructure

   # Verify file in R2
   # (Use rclone-temp pod method)
   ```

3. **Test restore procedure** (on non-production cluster if available)

### Regular Testing

**Test restore quarterly:**
1. Create test namespace
2. Deploy test PostgreSQL instance
3. Restore backup to test instance
4. Verify data integrity
5. Cleanup test resources

### Security

1. **Rotate R2 credentials** annually
2. **Encrypt backups at rest** (R2 encryption enabled)
3. **Audit backup access** (monitor R2 access logs)
4. **Secure Discord webhook** (treat as sensitive credential)

---

## Database Migration

When migrating from local PostgreSQL to cluster:

Use the included migration script:

```bash
# Edit configuration in script
./migrate_database.sh

# Features:
# - Automated connection testing
# - kubectl port-forward setup
# - Excludes app_auth table (cluster-specific)
# - Verification steps
# - Automatic pod restart
```

See script for detailed configuration options.

---

## Costs

### Cloudflare R2 Pricing (as of 2025)

- **Storage**: $0.015/GB/month
- **Class A Operations** (writes): $4.50/million requests
- **Class B Operations** (reads): $0.36/million requests
- **Egress**: Free (no bandwidth charges)

**Estimated monthly cost** (7-day retention, 50MB avg backup):
- Storage: ~$0.02/month
- Operations: ~$0.01/month
- **Total**: ~$0.03/month

**Extremely cost-effective** compared to alternatives!

---

## References

- [PostgreSQL Backup Documentation](https://www.postgresql.org/docs/current/backup.html)
- [rclone Cloudflare R2 Guide](https://rclone.org/s3/#cloudflare-r2)
- [Kubernetes CronJobs](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/)

---

## Changelog

- **2025-10-26**: Documented existing backup/restore system
  - Daily backups at 2 AM
  - 7-day retention
  - Discord notifications
  - Manual backup/restore jobs
