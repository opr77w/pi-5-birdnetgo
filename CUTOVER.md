# Cutover: pi-4-birdnet to pi-5-birdnetgo

Date: 09/11/2026. Detection downtime about 9 minutes (17:17:50 to 17:26).

## What moved

| Item | From | To |
|---|---|---|
| BirdNET-Go config, DB, models (`/srv/birdnet-go-app`) | Pi 4 USB stick | Pi 5 NVMe, same path. DB sha256 matched and passed `integrity_check` |
| Clips (8,813 files, 4.7 GB) | Pi 4 USB stick `/mnt/birdnet-go/external/clips` | Pi 5 NVMe, same path. Pre-seeded live, delta at cutover |
| BirdWeather station token | Pi 4 config | Pi 5 config (same station, history continuous) |
| USB sound card and Clippy mic | Pi 4 | Pi 5 (bound by USB ID `0d8c:0014`, no config change) |
| Physical position, UPS power | Pi 4 | Pi 5 |

## Pi 5 state after cutover

- `birdnet-go.service` enabled, image pinned to the Pi 4's build (`sha256:4dec04b2...`, 14/10/2026). Upgrade deliberately later.
- `fugleramme-frame` web kiosk enabled on :8081 (reads BirdWeather station 31900, as the Pi 4 did).
- Timers enabled: `birdnet-clips-to-nas`, `birdnet-db-snapshot`, `birdnet-crosscheck`, `timeshift-to-nas`.
- NUT secondary enabled against `pi-4-netutil.local`.
- First verified detection: tawny owl, 0.96, 17:26, clip saved and posted to BirdWeather.

## Other changes

- pi-0-frame: detector repointed to `http://pi-5-birdnetgo.local:8080` in `frame.env`, the unit, and `detector/data/settings.json` (the saved setting overrides the CLI). Backups `*.bak-20261109`.
- pi-4-birdnet: birdnet-go, fugleramme and birdnet timers disabled, then powered off. USB stick untouched as the rollback copy. NUT client fixed to `pi-4-netutil.local` (had been failing since 06/11/2026). Keypair `~/.ssh/id_ed25519` added for the migration; its key is in the Pi 5's `authorized_keys`.

## Rollback

Power the Pi 4, move the sound card back, `sudo systemctl enable --now birdnet-go fugleramme-frame birdnet-clips-to-nas.timer birdnet-db-snapshot.timer birdnet-crosscheck.timer`, disable the same on the Pi 5, and repoint pi-0-frame. Detections made on the Pi 5 since cutover would not be in the Pi 4's DB.

## Post-cutover changes (09/11/2026, 17:33)

- Clip retention changed from `usage` 80% to `age` 30d (`config.yaml` backed up as `config.yaml.bak-20261109`).
- BirdNET-Go upgraded and re-pinned to the 31/10/2026 build (`sha256:5eebc826...`). DB snapshot taken first: `db-snapshot/birdnet.db.pre-upgrade-20261109`. Capture confirmed running after restart.
- Pi 4 migration key removed from the Pi 5's `authorized_keys`. Rollback to the Pi 4 is no longer a supported path (the upgraded DB may not open on the older build).

## Open items

- Test restore to an SD card (see RESTORE.md).
- Pi 4 private key `~/.ssh/id_ed25519` remains on the powered-off Pi 4; harmless now its public half is removed.
