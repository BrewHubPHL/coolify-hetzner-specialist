# Backups & Disaster Recovery

**Impact:** HIGH  
**Tags:** backups, s3, r2, snapshots

## Layers

| Layer | Tool | RPO/RTO |
|-------|------|---------|
| VPS disk | Hetzner snapshot | Minutes / minutes |
| Coolify instance | S3 backup of Coolify DB + keys | Hours |
| App databases | Coolify scheduled DB backup | Cron-defined |
| Object archives | R2/S3 bucket policies | Policy-defined |

## Database backups (Coolify)

Configure scheduled backups for Postgres/MySQL/Mongo with cron + S3-compatible destination.

Supported destinations include **Cloudflare R2**, AWS S3, B2, MinIO — see Coolify S3 integration docs.

```text
Cron → pg_dump / vendor backup → S3/R2 bucket
```

Test restore quarterly — backup without restore drill is wishful thinking.

## Coolify instance backup

Full instance migration: backup Coolify Postgres, `APP_KEY`, SSH keys — [official how-to](https://coolify.io/docs/knowledge-base/how-tos/backup-and-restore-coolify).

## R2 as backup target

1. Create R2 bucket + API token
2. Add S3 destination in Coolify with R2 endpoint URL
3. Lifecycle rules for cost control

Hand off R2 IAM details to `cloudflare-specialist`.

## Before risky changes

Checklist:

- [ ] Hetzner snapshot
- [ ] DB backup job completed successfully (check last execution)
- [ ] Document rollback command

## Automated cleanup

Enable Coolify Docker cleanup (images, build cache) — prevents disk-full outages that look like app failures.

## References

- [Database backups](https://coolify.io/docs/databases/backups)
- [Cloudflare R2 backups (Coolify)](https://coolify.io/docs/knowledge-base/s3/cloudflare-r2)
- [Backup and restore Coolify](https://coolify.io/docs/knowledge-base/how-tos/backup-and-restore-coolify)
