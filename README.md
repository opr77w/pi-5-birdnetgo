# pi-5-birdnetgo

Garden BirdNET node. Raspberry Pi 5 with NVMe, same USB sound card (Clippy EM272 into C-Media `0d8c:0014`) and position as the original Pi 4 build. Live since the 09/11/2026 cutover from `pi-4-birdnet`; see [CUTOVER.md](CUTOVER.md) for what moved and when.

## Network and access

| | |
|---|---|
| Hostname | `pi-5-birdnetgo` (`pi-5-birdnetgo.local`) |
| IP | `192.168.7.228`, eero reservation |
| User | `pi`, passwordless sudo, key auth from the MacBook |
| Fleet | `PI-FLEET/hosts.json` |
| Web UI | `http://pi-5-birdnetgo.local:8080` |
| Frame kiosk | `http://pi-5-birdnetgo.local:8081` (Fugleramme) |

## Storage

- NVMe via the Pi 5 M.2 HAT. BirdNET-Go config, DB, models and clips all at `/srv/birdnet-go-app` and `/mnt/birdnet-go/external` (same paths as the Pi 4's USB layout, so the units didn't change). Live DB, WAL and SHM are excluded from backups; `birdnet-db-snapshot` takes a consistent `sqlite3 .backup` at 11:45 for the 12:00 Timeshift run. See [CHANGELOG-2026-11-06-storage.md](CHANGELOG-2026-11-06-storage.md).
- Bootloader `BOOT_ORDER=0xf461` (SD first, then NVMe): pull the SD to boot NVMe.

## Backup and restore

Timeshift only — no bare-metal image. Daily (5), weekly (3), monthly (2) snapshots, mirrored to the Synology by `timeshift-to-nas` at 05:00. Clips go separately via `birdnet-clips-to-nas` (rsync, 03:30). Full restore procedure in [RESTORE.md](RESTORE.md).

## What's in this repo

| Path | What |
|---|---|
| `birdnet-go.service` | BirdNET-Go systemd unit (Docker, `/srv/birdnet-go-app` paths) |
| `birdnet-crosscheck{,.service,.timer}` | Weekly cross-check against the BirdWeather station (Mon 07:40) |
| `birdnet-clips-to-nas{,.service,.timer}` | Daily rsync of clips to the NAS (03:30) |
| `birdnet-db-snapshot{,.service,.timer}` | Daily consistent DB copy before Timeshift (11:45) |
| `timeshift-exclude.json` | Timeshift exclusions (live DB/WAL/SHM, HLS tmpfs, caches) |
| `zz-persistent.conf` | Persistent journald, size-capped for SSD wear |
| `config/birdnet-go/config.yaml` | BirdNET-Go runtime config (locale, retention, thresholds) |
| `config/fugleramme/frame.env` | Fugleramme frame env (`DETECTOR_URL`, port 8081) |
| `config/etc/modprobe.d/alsa-birdnet.conf` | Pin USB card to ALSA index 0 by vendor/product ID |
| `config/etc/nut/{nut,upsmon}.conf` | NUT secondary of `pi-4-netutil.local` (Eaton 3S 550) |
| `config/etc/pi-notify.conf` + `config/usr/local/sbin/pi-notify` | HA webhook used by the timers on failure |
| `config/etc/sudoers.d/010_pi-nopasswd` | `pi` passwordless sudo, `verifypw = any` |
| `diagrams/estate_*` | Estate diagram (BirdNET node, UPS shutdown, backups). Dates from the Pi 4 build; still largely accurate |
| `reports/crosscheck-*` | Weekly cross-check reports |
| `CUTOVER.md`, `RESTORE.md` | 09/11/2026 cutover log; Timeshift restore procedure |
| `archive/pi-4/` | Pi 4-specific config and the original PI-4-BIRDNET README, kept for reference (pi-image-to-mac backups, USB-stick fstab, Pi 4 audio-off boot config, pre-06/11 service unit, Argon ONE case notes) |

## Operations

```
systemctl status birdnet-go
journalctl -u birdnet-go -f
docker logs birdnet-go
systemctl list-timers | grep -E 'birdnet|timeshift'
```

## Open items

- Test restore to an SD card (see [RESTORE.md](RESTORE.md)).
- Decide whether the Argon ONE V2 case (installed on the Pi 4 on 06/11/2026) is moving to the Pi 5. If yes, promote `archive/pi-4/argononed.conf` and `archive/pi-4/CHANGELOG-2026-11-06-argon.md` back up and re-install the Argon daemon on the Pi 5.
- Set a web UI password before any remote exposure.
