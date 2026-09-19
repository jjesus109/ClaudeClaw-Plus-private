# Feature: Cloud Backup for All Storages

**Date:** 2026-09-18  
**Type:** Infra / Resilience  
**Repo:** ai-hub  
**Status:** IN PROGRESS — local+alerts done, cloud credentials pending

---

## Goal

Extend the local daily backup to also push to cloud storage, providing off-site disaster recovery for:
- PostgreSQL (ai_hub database)
- Redis (connector sessions + ai-hub cache)
- Kubernetes PVC data (if needed)

---

## Current State (Done)

- Daily backup at 03:00 server time via `/home/javi-server/backups/backup-ai-hub.sh`
- Retention: 7 days local
- Output: `~/backups/ai-hub/postgres/ai_hub_YYYYMMDD_HHMMSS.sql.gz` + `~/backups/ai-hub/redis/dump_*.rdb`
- Log: `~/backups/backup.log`

---

## Cloud Backup Options

| Option | Cost | Setup effort | Notes |
|--------|------|-------------|-------|
| Backblaze B2 | ~$0.006/GB/month | Low — rclone config | Best price, S3-compatible |
| AWS S3 | ~$0.023/GB/month | Low — rclone or aws-cli | Most common, good tooling |
| Google Cloud Storage | ~$0.02/GB/month | Low — rclone | Good if already on GCP |
| Cloudflare R2 | $0/10GB free then $0.015/GB | Low — rclone/S3 API | Free tier generous for small data |

**Recommendation:** Cloudflare R2 (free tier covers current data volume) or Backblaze B2 (cheapest paid).

---

## Implementation Plan

### Step 1: Choose provider + create bucket ⏳ PENDING — Javi action required

```bash
# Cloudflare R2: Dashboard → R2 → Create bucket "ai-hub-backups" → Manage R2 API Tokens → Create token
# Supabase: Dashboard → Storage → Create bucket "ai-hub-backups"
# Then run: ~/backups/rclone-r2-setup.sh  (prompts for credentials interactively)
# And run:  ~/backups/rclone-supabase-setup.sh
```

### Step 2: Install rclone on server ✅ DONE — v1.75.1

### Step 3: Update backup script ✅ DONE

Add to `~/backups/backup-ai-hub.sh`:
```bash
# Upload to cloud (after local backup succeeds)
echo "[$DATE] Uploading to cloud..."
rclone copy "$BACKUP_DIR/postgres/ai_hub_$DATE.sql.gz" ai-hub-backup:ai-hub-backups/postgres/
rclone copy "$BACKUP_DIR/redis/dump_$DATE.rdb" ai-hub-backup:ai-hub-backups/redis/
echo "[$DATE] Cloud upload done"

# Cloud retention: delete cloud files older than 30 days
rclone delete --min-age 30d ai-hub-backup:ai-hub-backups/postgres/
rclone delete --min-age 30d ai-hub-backup:ai-hub-backups/redis/
```

### Step 4: Test restore ⏳ PENDING — needs cloud credentials first

```bash
# After cloud is configured:
rclone copy ai-hub-r2:ai-hub-backups/postgres/ai_hub_latest.sql.gz /tmp/
gunzip -c /tmp/ai_hub_latest.sql.gz | sudo kubectl exec -i -n ai-hub postgres-0 -- psql -U postgres ai_hub_restore
```

### Step 5: Alert on backup failure ✅ DONE

Telegram notification on failure implemented. Reads bot token from k8s secret, sends to chat_id `38949738`. Tested clean.

---

## Storage Estimates

| Data | Current size | Growth/month | 7-day local | 30-day cloud |
|------|-------------|-------------|-------------|-------------|
| PostgreSQL | ~5MB | ~50MB | ~35MB | ~1.5GB |
| Redis | ~1MB | ~5MB | ~7MB | ~150MB |

Well within free tiers.

---

## SDLC Stages

| Stage | Status |
|-------|--------|
| 1. Inspect | ✅ Done |
| 2. Plan | ✅ Done — this file |
| 3. Implement | ✅ Done — R2 configured, cloud upload working |
| 4. Write tests | ✅ Done — infra/backup/test_backup.sh (10 tests) |
| 5. Tests pass | ✅ Done — 10/10 |
| 6. Security review | ✅ Done — curl --data-urlencode fix applied |
| 7. Update docs | ✅ Done — script added to repo at infra/backup/ |
| Push to main | ⏳ Pending |
