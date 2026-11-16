# 06/11/2026: Argon ONE V2 case, power button and fan

## Hardware change

- pi-4-birdnet moved from PoE to an Argon ONE V2 case, powered over USB-C from the UPS. The case jumper is set to always-on.

## What was installed

- `i2c-tools`, from apt. `python3-libgpiod` 2.2.1 and `python3-smbus` were already present. The case MCU answers on I2C bus 1 at 0x1a.
- Argon daemon files, installed by hand (see `argon/SHA256SUMS` for sources, paths and hashes):
  - `argononed.py` and helpers in `/etc/argon`.
  - The libgpiod button handler, watching GPIO4.
  - `argononed.service`.
  - The shutdown hook in `/usr/lib/systemd/system-shutdown/`. The directory had to be created, because the vendor path `/lib/systemd/system-shutdown` does not exist on trixie.
- Fan curve in `/etc/argononed.conf`: 55°C at 30%, 60°C at 55%, 65°C at 100%.

## Deliberately not done

The vendor installer `argon1.sh` was reviewed and not run. It would have done the following, none of which is wanted on this box:

- `apt-get upgrade -y`, an unattended full system upgrade.
- `rpi-eeprom-update`, plus Argon's EEPROM PSU script, which sets `PSU_MAX_CURRENT=5000` (a Pi 5 setting).
- Enable the serial UART in `config.txt`.
- Install the OLED, IR, UPS, dashboard and registration scripts.
- A time check against worldtimeapi.org.

## Verified

- Button held about 3s: clean shutdown. On the next boot no EXT4 journal recovery was needed on either the SD card or the USB stick.
- Power-on: the Pi booted, and BirdNET-Go came back on USB storage. The UI returned 302, the capture PCM was RUNNING and the DB quick_check was ok.
- Double tap: clean reboot (systemd-reboot.service, with no EXT4 recovery on the next boot). The case MCU still answers at 0x1a after the reboot.

## Follow-ups done the same day

- Persistent journal: the drop-in `/etc/systemd/journald.conf.d/zz-persistent.conf` (`etc/journald.conf.d/` in this repo) overrides Raspberry Pi OS's `Storage=volatile`. It is capped at 100M and one month. The previous boot's shutdown is now readable with `journalctl -b -1`.
- Removed `dtoverlay=i2c1-pi5`, a Pi 5 overlay, from the `[all]` section of `config.txt`. The backup is `/boot/firmware/config.txt.bak-20261106`. I2C still comes from `dtparam=i2c_arm=on` and `dtoverlay=i2c1`.
