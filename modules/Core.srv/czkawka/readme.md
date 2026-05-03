# czkawka — photo dedup on TrueNAS SCALE

Runs [Czkawka](https://github.com/qarmin/czkawka) as a Docker container on TrueNAS SCALE to deduplicate a photo archive against a Google Takeout dump.

## What this does

- Mounts a photo archive as **read-only** — nothing in it can ever be deleted or modified by the container
- Mounts a Google Takeout export as **read/write** — duplicates get removed from here only
- Finds and removes:
  - **Exact duplicates** — via Blake3 hash comparison
  - **Similar images** — catches low-resolution Google copies of original photos using perceptual hashing

## Compose

```yaml
services:
  czkawka:
    image: jlesage/czkawka
    ports:
      - "5800:5800"
    environment:
      - USER_ID=0
      - GROUP_ID=0
    volumes:
      - /mnt/Fast/czkawka:/config:rw
      - /mnt/Storage/Photos_archive:/photos_archive:ro
      - /mnt/Storage/takeout:/takeout:rw
```

Access the GUI at `http://<nas-ip>:5800`

## Scan workflow

1. **Duplicate Files scan** — Hash / Blake3, `photos_archive` set as Reference folder. Auto-select all except reference, delete.
2. **Similar Images scan** — Gradient hash, size 32, similarity ~10. Same reference setup. Review a sample before bulk deleting.

## Notes

- `:ro` on the archive is enforced at the kernel level — no app bug or accidental click can delete from it
- The `apps` group (GID 568) owns the datasets; `USER_ID/GROUP_ID=0` is used to ensure container access
- Czkawka does not cache scans — every run starts fresh
