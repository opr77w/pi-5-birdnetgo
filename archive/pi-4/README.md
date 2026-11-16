# PI-4-BIRDNET

Garden BirdNET node. Raspberry Pi 4B 4 GB, PoE HAT, single box doing capture and inference (iteration 1). Built 16/10/2026.

## Network and access

| | |
|---|---|
| Hostname | `pi-4-birdnet` (`pi-4-birdnet.local`) |
| IP | `192.168.4.37`, eero reservation, wired over PoE |
| User | `pi`, passwordless sudo, key auth from the MacBook Air |
| Fleet | in `PI-FLEET/hosts.json` as `pi-4-birdnet`, aliases `birdnet`, `garden` |
| Web UI | `http://192.168.4.37:8080`, no password yet (LAN only) |

## OS

Raspberry Pi OS 64-bit with desktop, Debian 13 Trixie, kernel 6.18, image 18/06/2026. Root on 32 GB SD card. Timezone Europe/London.

## Audio

- Mic: Clippy EM272M, plug-in power, into the CM108 USB sound card (`0d8c:0014`, Unitek Y-247A), mono, 48 kHz.
- Card pinned to ALSA index 0 in `/etc/modprobe.d/alsa-birdnet.conf` by vendor/product ID, so BirdNET-Go's saved device survives reboots.
- Onboard audio disabled: `dtparam=audio=off` and `dtoverlay=vc4-kms-v3d,noaudio` in `/boot/firmware/config.txt`. Needed because the HDMI audio device otherwise takes index 0 and the USB pin fails.
- Mixer: AGC off, capture gain 35/35 (+23 dB), stored with `alsactl store`.
- Bench noise floor: -76 dBFS RMS in a quiet room at full gain. Voice test peaked -13 dBFS.

## Storage

- Integral Fusion 64 GB USB stick, single GPT partition, ext4, label `birdnet`, 0% reserved. The first stick (Transcend JetFlash) died on its first cold boot after formatting and was replaced.
- fstab: `UUID=538271c7-b62f-4871-8f64-92a575e99e95 /mnt/birdnet-go/external ext4 defaults,noatime,nofail,x-systemd.device-timeout=10 0 2`
- `/mnt/birdnet-go/external` is the path the BirdNET-Go installer bind-mounts into the container as `/external` with mount propagation, so no edits to the unit file. Clips go to `/external/clips/`.
- Database and config stay on the SD card at `~/birdnet-go-app/`.

## BirdNET-Go

- Installed 16/10/2026 with `install.sh --silent` (Docker, image `ghcr.io/tphakala/birdnet-go:latest`, systemd unit `birdnet-go.service`). Run `install.sh` as `pi`, never sudo. Logs in `~/birdnet-install*.log`.
- Config `~/birdnet-go-app/config/config.yaml` (backup `.bak-*` beside it). Changed from installer defaults:
  - location 50.89, 0.45; locale en-uk
  - audio source `sysdefault` (card 0)
  - export: `/external/clips/`, type `flac`, retention by usage 80%
  - `birdnet.threads: 3` (leave a core for the desktop)
  - `birdnet.overlap: 2.2`, `falsepositivefilter.level: 2` (balanced), `dynamicthreshold.min: 0.3`
- Bats are not on this box: the AudioMoth is the bat unit. Don't enable BattyBirdNET here (card can't do 192 kHz).

## UPS (added 16/10/2026)

NUT secondary of pi-4-netutil's `upsd` (Eaton 3S 550, `eaton@192.168.4.227`, account `upsmon_secondary`). This Pi is PoE-fed from the switch on the UPS, so it has no UPS of its own to talk to.

- `/etc/nut/nut.conf` `MODE=netclient`; `/etc/nut/upsmon.conf` monitors `eaton@192.168.4.227` as secondary, `SHUTDOWNCMD "/sbin/shutdown -h +0"`, no `POWERDOWNFLAG`.
- On low battery (25%) netutil raises FSD; this Pi halts (Docker stop plus halt, about 15 s); netutil waits `HOSTSYNC` 30 s (raised from 15 on 16/10/2026 for this), halts after `FINALDELAY` 60 s, and the Eaton cuts output 45 s later. When mains returns the Eaton restores output, the switch reboots and PoE boots this Pi.
- Check: `upsc eaton@192.168.4.227`; `upsc -c eaton@192.168.4.227` on either Pi lists connected monitors (should show 192.168.4.37).
- Not yet tested end to end; netutil's `sudo upsmon -c fsd` would halt both Pis.

## Fugleramme (added 16/10/2026)

E-ink bird frame software ([arnegiacomo/fugleramme](https://github.com/arnegiacomo/fugleramme)), v0.28.1, running on this Pi in web-only mode (no panel yet). Reads BirdNET-Go's API at `http://127.0.0.1:8080`.

- Kiosk `http://192.168.4.37:8081`, admin `http://192.168.4.37:8081/admin`. Service `fugleramme-frame.service`.
- Install: shallow clone at `~/fugleramme` (full history is over 1 GB of artwork), `frame.env` pre-seeded with `DETECTOR_MODE=external`, `DETECTOR_URL=http://127.0.0.1:8080`, `FRAME_PORT=8081`, then `./install.sh -y`. SPI and I2C enabled for a future Inky Impression panel.
- `/etc/sudoers.d/010_pi-nopasswd` gained `Defaults:pi verifypw = any`: the installer's `sudo -v` otherwise prompts, because `pi` also matches the password-requiring `%sudo` rule.
- Panel decision pending: Inky Impression 13.3" or 7.3" on a separate frame-only Pi indoors (Zero 2 W or Pi 4, pointed at `http://192.168.4.37:8080`), or an ESP32 battery frame fetching the rendered image from this Pi.

## Operations

```
systemctl status birdnet-go
journalctl -u birdnet-go -f
docker logs birdnet-go
```
## Backup (added 16/10/2026, same method as pi-4-netutil)

Weekly bare-metal image of the SD card (`/dev/mmcblk0`) sent over SSH to the MacBook at `192.168.4.34`, landing in `~/PiBackups/pi-4-birdnet/current.img.zst` (previous image kept as `previous.img.zst`). Checksum verified end to end; result pushed to the phone via `pi-notify` (HA webhook, `/etc/pi-notify.conf`).

Differences from netutil: this Pi has no clone disk, so there is no nightly `rpi-clone` layer, and the image is taken from the live root with `birdnet-go` stopped for the duration (about 16 min; its SQLite DB is on the SD). The clips stick is not imaged: regenerable, 57 GB.

- Script `/usr/local/sbin/pi-image-to-mac`, key `/root/.ssh/pi-image-backup_ed25519`, state in `/var/lib/pi-image-backup/`.
- `pi-image-to-mac.timer` runs hourly; sends only when the last good image is older than 7 days and the Mac answers on port 22. Warns on the phone once a day after 10 days without a good image.
- Mac side: forced command `~/PiBackups/receive.sh pi-4-birdnet` on the key's `authorized_keys` line (`receive.sh` generalised 16/10/2026 to take the host as an argument; netutil's line unchanged).
- Force a run: `sudo /usr/local/sbin/pi-image-to-mac --force`. Logs: `journalctl -u pi-image-to-mac`.
- First image 16/10/2026: 2.4 GB compressed, 961 s.
- Restore: `zstd -dc current.img.zst | dd of=/dev/<sd> bs=4M` from the Mac, or Raspberry Pi Imager with the decompressed `.img`.
Update = re-run `~/install.sh` as `pi`, option 1.

## Open

- Set a web password (Settings > Security) before any remote exposure.
- Pull the 2.4 GHz keyboard/mouse receiver before deploying outdoors.
- 24 to 48 h indoor soak, mic out of a window, then review detections, temperature and disk growth.
- Iteration 2: mirror detections to TimescaleDB with the Ecowitt data; e-ink frame.

## Changelog

- 16/10/2026: initial build. Fleet entry, stick format (twice), ALSA pin, onboard audio off, BirdNET-Go install and tuning.
- 16/10/2026: weekly image backup to the MacBook added (pi-image-to-mac, pi-notify); first image verified.
- 16/10/2026: NUT secondary of pi-4-netutil for clean UPS shutdown; netutil HOSTSYNC 15 to 30.
- 16/10/2026: Fugleramme v0.28.1 installed web-only on port 8081, reading BirdNET-Go locally.
