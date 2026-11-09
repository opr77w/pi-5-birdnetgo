# pi-5-birdnetgo: restore runbook

Written 09/11/2026. Backup model: Timeshift only (no bare-metal image; decided 09/11/2026).

## What is backed up, and where

| Data | Mechanism | Location |
|---|---|---|
| System, `/etc`, `/usr/local/sbin`, `/home/pi` (Fugleramme, `frame.env`), `/srv/birdnet-go-app/config` (config.yaml, BirdWeather token), DB snapshot | Timeshift daily (5 daily, 3 weekly, 2 monthly), mirrored by `timeshift-to-nas` at 05:00 | `NetBackup/Pi-5-birdnetgo/timeshift/snapshots/` |
| BirdNET-Go DB (consistent copy) | `birdnet-db-snapshot` timer | `/srv/birdnet-go-app/db-snapshot/birdnet.db` (inside Timeshift) |
| Clips | `birdnet-clips-to-nas` timer (rsync) | `NetBackup/Pi-5-birdnetgo/birdnet-clips/` |

Excluded from Timeshift by design: live `birdnet.db`/`-wal`/`-shm`, clips, caches, logs, the HLS tmpfs, `/mnt/netbackup`.

Key facts: host `pi-5-birdnetgo`, user `pi`, 192.168.7.228 (eero reservation), DiskStation 192.168.7.231, NUT server `pi-4-netutil.local`, bootloader `BOOT_ORDER=0xf461` (SD first, then NVMe).

## 1. Fresh OS

1. Raspberry Pi Imager on the Mac: device Raspberry Pi 5, OS Raspberry Pi OS (64-bit) desktop, target the NVMe (USB enclosure) or an SD card in an emergency.
2. OS customisation: hostname `pi-5-birdnetgo`, user `pi`, SSH on, Wi-Fi as before.
3. Boot. SD first wins, so remove any SD card to boot the NVMe.

## 2. Base access

```
echo 'pi ALL=(ALL) NOPASSWD: ALL' | sudo tee /etc/sudoers.d/010_pi-nopasswd && sudo chmod 440 /etc/sudoers.d/010_pi-nopasswd && sudo visudo -c
ssh-copy-id pi@pi-5-birdnetgo.local            # from the Mac
sudo apt update && sudo apt install -y nfs-common rsync
sudo mkdir -p /mnt/netbackup
sudo mount -t nfs -o vers=3,proto=tcp 192.168.7.231:/volume1/NetBackup/Pi-5-birdnetgo /mnt/netbackup
ls /mnt/netbackup/timeshift/snapshots/         # pick the newest, e.g. 2026-11-09_14-39-00
```

If the mount is refused, check DSM: Shared Folder > NetBackup > NFS Permissions must include 192.168.7.228.

## 3. Restore the system from the snapshot

Keep the fresh install's boot partition, fstab and machine identity; take everything else from the snapshot.

```
SNAP=/mnt/netbackup/timeshift/snapshots/<newest>/localhost
sudo cp -p /etc/fstab /root/fstab.fresh
sudo rsync -aHAX --numeric-ids --info=progress2 \
  --exclude='/boot/firmware/' --exclude='/proc/' --exclude='/sys/' --exclude='/dev/' \
  --exclude='/run/' --exclude='/tmp/' --exclude='/mnt/' --exclude='/media/' \
  --exclude='/lost+found' --exclude='/etc/fstab' --exclude='/etc/machine-id' \
  --exclude='/var/lib/dbus/machine-id' --exclude='/timeshift/' \
  "$SNAP/" /
```

Then re-add the NetBackup line to the restored fstab, using the fresh one's root and boot UUIDs:

```
192.168.7.231:/volume1/NetBackup/Pi-5-birdnetgo /mnt/netbackup nfs vers=3,hard,proto=tcp,_netdev,nofail,noauto,x-systemd.automount,x-systemd.idle-timeout=600,x-systemd.mount-timeout=30 0 0
```

`sudo mkdir -p /mnt/birdnet-go/external/clips && sudo chown -R 1000:1000 /mnt/birdnet-go` (the mount points are excluded from the rsync).

## 4. Restore BirdNET-Go data

```
sudo systemctl stop birdnet-go
sudo cp -p /srv/birdnet-go-app/db-snapshot/birdnet.db /srv/birdnet-go-app/data/birdnet.db
sudo chown 1000:1000 /srv/birdnet-go-app/data/birdnet.db
sudo rsync -aH --numeric-ids /mnt/netbackup/birdnet-clips/ /mnt/birdnet-go/external/clips/
```

## 5. Bring it up and verify

```
sudo systemctl daemon-reload
sudo reboot
```

After reboot, all of these should pass:

| Check | Command | Expect |
|---|---|---|
| BirdNET-Go | `systemctl is-active birdnet-go`, web UI on :8080 | active; detections appearing |
| BirdWeather | BirdNET-Go UI, integrations | uploads succeeding |
| Fugleramme | `systemctl is-active fugleramme-frame`, :8081 | active |
| UPS | `upsc eaton@pi-4-netutil.local ups.status` | `OL` |
| Timers | `systemctl list-timers \| grep -E 'birdnet\|timeshift'` | clips, db-snapshot, crosscheck, timeshift-to-nas listed |
| Timeshift | `sudo timeshift --list` | config present (UUID may need `sudo timeshift --snapshot-device /dev/nvme0n1p2`) |
| Remote access | Raspberry Pi Connect, VNC | online (Connect may need `rpi-connect signin` again) |

## Test restore log

| Date | Target | Snapshot | Result | Notes |
|---|---|---|---|---|
| _deferred_ | SD card | | | Skipped before cutover 09/11/2026; still to do |
