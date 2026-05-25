# Backup Options for a Hermes Agent at Scale

The main spec (`PKM-AGENT-SPEC.md`) uses a single private GitHub repo for backup, which
works well under ~1GB. This document explores what to do when you outgrow that.

## Why GitHub Breaks Down

- **Repo size limit:** GitHub recommends repos stay under 5GB, hard limit ~10GB
- **File size limit:** 100MB per file without LFS; LFS gives 1GB free storage + 1GB/month bandwidth (more is $5/50GB)
- **Push performance:** large binary diffs (SQLite snapshots, images) bloat git history fast
- **Cost:** LFS data packs are cheap but add up; a 10GB store with daily snapshots can burn through bandwidth quickly

**Rule of thumb:** if your `~/.hermes/` data directory exceeds ~1GB or your SQLite DB exceeds ~100MB, it's time to move beyond plain git.

---

## Option 1: Git for Config + Restic for Data

**Best for:** 1-50GB, users who want encrypted incremental backups with free storage

[Restic](https://restic.net/) is a fast, encrypted, deduplicated backup tool. It handles
binary files efficiently and supports multiple storage backends.

**How it works:**
```bash
# One-time setup
restic -r b2:hermes-backup init

# Every 6 hours (cron)
sqlite3 ~/.hermes/knowledge.db ".backup /tmp/hermes-db-backup.db"
restic -r b2:hermes-backup backup /tmp/hermes-db-backup.db ~/.hermes/files/
restic -r b2:hermes-backup forget --keep-hourly 24 --keep-daily 30 --keep-monthly 12 --prune
```

**Storage backends:**
| Backend | Free Tier | Cost After | Notes |
|---------|-----------|------------|-------|
| Backblaze B2 | 10GB | $0.006/GB/month | Best value for most use cases |
| AWS S3 | None | ~$0.023/GB/month | More expensive, more features |
| Wasabi | None | $0.0069/GB/month | No egress fees, 1TB minimum billing |
| Local/SFTP | N/A | Whatever your second server costs | Good if you have another VPS |
| rclone mount | N/A | Depends on cloud provider | Restic writes to any rclone remote |

**Config stays in git** (the text-friendly stuff: `config.yaml`, skills, memory markdown).
Only binary/large data goes through restic.

**Pros:**
- Encrypted at rest (you hold the key)
- Deduplication -- unchanged blocks aren't re-uploaded
- Point-in-time restore with flexible retention policies
- Handles large files and binary data well

**Cons:**
- Extra tool to install and manage
- Need a storage account (B2 is free up to 10GB)
- Restore requires restic installed on the new machine

**Recovery:**
```bash
# On new VPS
restic -r b2:hermes-backup restore latest --target /restore
cp -r /restore/tmp/hermes-db-backup.db ~/.hermes/knowledge.db
cp -r /restore/root/.hermes/files/ ~/.hermes/files/
git clone https://github.com/YOU/hermes-config.git  # restore config
```

---

## Option 2: Rclone Sync to Cloud Storage

**Best for:** 1-20GB, users who want simplicity and already use a cloud provider

[Rclone](https://rclone.org/) is "rsync for cloud storage." It does straight file sync
without deduplication or encryption (though encryption can be layered on).

**How it works:**
```bash
# Every 6 hours
sqlite3 ~/.hermes/knowledge.db ".backup ~/.hermes/backups/knowledge.db"
rclone sync ~/.hermes/ remote:hermes-backup/ --exclude "sessions/**"
```

**Supported destinations (50+), most relevant:**
| Destination | Free Tier | Notes |
|-------------|-----------|-------|
| Google Drive | 15GB | Easy setup, familiar |
| Dropbox | 2GB | Small but simple |
| OneDrive | 5GB | If you have a Microsoft account |
| Backblaze B2 | 10GB | Cheapest paid option |
| S3-compatible | Varies | Any S3 provider works |

**Pros:**
- Dead simple -- it's just file sync
- No special restore tool needed (download files from cloud UI or rclone)
- Familiar cloud storage destinations
- Can browse backups directly in Google Drive / Dropbox

**Cons:**
- No deduplication -- full copy every time (bandwidth-heavy for large DBs)
- No built-in encryption (add `rclone crypt` wrapper if needed)
- No point-in-time snapshots (only latest state, unless destination has versioning)
- Cloud provider versioning/retention varies

**Recovery:**
```bash
rclone sync remote:hermes-backup/ ~/.hermes/
# Or just download from cloud storage UI
```

---

## Option 3: LiteStream for SQLite + Rclone/Restic for Files

**Best for:** users who can't afford to lose even 6 hours of data from the database

[LiteStream](https://litestream.io/) continuously replicates SQLite databases to S3-compatible
storage. It streams WAL changes in near-real-time, so you get point-in-time recovery
down to seconds, not hours.

**How it works:**
```yaml
# /etc/litestream.yml
dbs:
  - path: /root/.hermes/knowledge.db
    replicas:
      - type: s3
        bucket: hermes-litestream
        endpoint: s3.us-west-000.backblazeb2.com
```

```bash
# Runs as a sidecar process
litestream replicate
```

Files (PDFs, images) still need a separate backup mechanism -- restic or rclone on a cron.

**Pros:**
- Near-zero RPO (recovery point objective) for the database
- Minimal bandwidth -- only WAL changes are streamed
- Can restore to any point in time
- Lightweight, single-binary, designed specifically for SQLite

**Cons:**
- Only handles SQLite, not files -- need a second solution for images/PDFs
- Adds a running process to manage
- Requires S3-compatible storage (B2, Minio, AWS S3)
- Overkill if 6-hour snapshots are acceptable

**Recovery:**
```bash
litestream restore -o ~/.hermes/knowledge.db s3://hermes-litestream/knowledge.db
# Restore files separately from restic/rclone
```

---

## Option 4: VPS Provider Snapshots

**Best for:** users who want zero-effort backup and don't mind paying for it

Most VPS providers offer automated snapshots of the entire disk.

| Provider | Snapshot Cost | Frequency | Notes |
|----------|-------------|-----------|-------|
| Hetzner | 20% of server cost | Configurable | Whole-disk, easy restore |
| DigitalOcean | 20% of server cost | Weekly auto | On-demand also available |
| Linode/Akamai | $0.06/GB/month | On-demand | Manual or API-triggered |
| Vultr | $1-5/month | Auto daily | Configurable retention |

**Pros:**
- Zero configuration -- entire VPS is backed up
- Restore is "create new VPS from snapshot"
- Captures everything: OS, config, data, installed packages

**Cons:**
- Expensive relative to data-only backup
- Coarse granularity (daily at best, often weekly)
- Provider lock-in -- can't easily restore to a different provider
- Snapshots are not crash-consistent for SQLite (should still quiesce DB first)

---

## Comparison Matrix

| Approach | Max Practical Size | RPO | Cost (10GB) | Complexity | Encryption | Cross-Provider Restore |
|----------|-------------------|-----|-------------|------------|------------|----------------------|
| Git repo | ~1GB | 6h | Free | Low | No (unless private repo) | Yes |
| Restic + B2 | 100GB+ | 6h | ~$0.06/mo | Medium | Yes (built-in) | Yes |
| Rclone + cloud | 50GB+ | 6h | Free-$0.07/mo | Low | Optional | Yes |
| LiteStream + restic | 100GB+ | ~seconds (DB) | ~$0.06/mo | High | Varies | Yes |
| VPS snapshots | Whole disk | 24h | ~$1-5/mo | None | Provider-managed | No |

---

## Recommended Progression

1. **Start with git** (current plan) -- good to ~1GB, free, simple
2. **Graduate to restic + B2** when data exceeds 1GB -- 10GB free on B2, encrypted, deduplicated, still cheap
3. **Add LiteStream** only if you find that 6-hour RPO is too coarse for the database
4. **VPS snapshots** are a nice belt-and-suspenders addition at any stage but shouldn't be your only backup

The transition from stage 1 to stage 2 is straightforward: keep git for config, add a
restic cron for data. No need to over-engineer upfront.
