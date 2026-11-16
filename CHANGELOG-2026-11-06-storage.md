# 06/11/2026: BirdNET-Go storage moved to USB, Timeshift coverage

## What changed

- All BirdNET-Go persistent storage (config, DB, logs, models) is now on the USB stick at `/mnt/birdnet-go/external/app`, bind-mounted to `/srv/birdnet-go-app`. Clips were already on USB at `/mnt/birdnet-go/external/clips`.
- `birdnet-go.service` now mounts `/srv/birdnet-go-app/{config,data}` and has `RequiresMountsFor=/mnt/birdnet-go/external /srv/birdnet-go-app`, so it will not start without the stick.
- The underlying SD directories `/mnt/birdnet-go/external`, `/mnt/birdnet-go/external/clips` and `/srv/birdnet-go-app` are `chattr +i`, so nothing can write to the SD card if the stick is absent.
- `birdnet-db-snapshot` (11:45 daily) writes a consistent `sqlite3 .backup` to `/srv/birdnet-go-app/db-snapshot/birdnet.db` before the 12:00 Timeshift run. The live DB, WAL and SHM are excluded from Timeshift.
- Timeshift now includes `/srv/birdnet-go-app`, excluding the live DB, cache, logs, the empty legacy `data/clips` and the HLS tmpfs.
- `birdnet-clips-to-nas` (03:30 daily) mirrors clips to `NetBackup/Pi-4-birdnet/birdnet-clips` on the DiskStation. HA is notified via `pi-notify` on failure.
- `sqlite3` package installed.

## Why

- SD wear and space: the DB and logs were on the SD card.
- `nofail` on the USB mount meant BirdNET-Go could silently write to the SD card when the stick was missing. An empty stray `clips/` folder was found there, created at 12:12 on 06/11/2026.
- Timeshift could not see the data. Its built-in `/mnt/*` exclude comes before user rules, and a `/home/*/**` rule excluded the old `~/birdnet-go-app`, so before this change only the SD image covered BirdNET data. The `/srv` bind mount is how Timeshift reaches the USB data.

## Rollback

- Original data: `/home/pi/birdnet-go-app.pre-usb-2026-11-06` (on SD).
- Backups of edited files: `/etc/fstab.bak-20261106`, `/etc/systemd/system/birdnet-go.service.bak-20261106`, `/etc/timeshift/timeshift.json.bak-20261106`.
- Remove the immutable flags with `chattr -i` on the underlying dirs; the stick must be unmounted, or reach them via `mount --bind / /tmp/rootview`.
