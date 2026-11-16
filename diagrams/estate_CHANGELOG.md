# Home Pi estate diagram: changelog

Diagram files live in `diagrams/`. One spec (`.json`) renders both the SVG and the Excalidraw with the personal-style renderers kept in the pi-4-netutil repo (`diagrams/render/scripts/render_svg.py`, `render_excalidraw.py`); edit the spec and re-render rather than editing outputs by hand.

## 2026-10-16 · v1 PUBLISHED · `estate_2026-10-16_PUBLISHED.{svg,excalidraw,json}`

First version. Scope is what was built on 16/10/2026 across the three Pis, plus the existing pieces they attach to.

### What it shows

- **pi-4-birdnet** (new): Clippy EM272M → CM108 (pinned ALSA card 0, onboard audio off) → BirdNET-Go in Docker on :8080 → FLAC clips on the Integral 64 GB stick via the installer's `/external` mount. Fugleramme web-only on :8081 reading the local API. NUT secondary of netutil. Weekly image backup that stops birdnet-go while it runs.
- **pi-0-frame** (new, pending): Zero 2 W running Fugleramme frame-only against birdnet's API, Inky Impression panel not yet bought. Marked with risk bars: the Zero dropped off the LAN after the installer's reboot and is unverified.
- **pi-4-netutil** (changed today): NUT primary on the Eaton 3S 550 with `HOSTSYNC` 30 s; nightly rpi-clone SSD → SD; weekly image to the MacBook; `pi-notify` webhook to Home Assistant. rclone Dropbox sync shown as existing.
- **Power**: the Eaton feeds the PoE switch, eero and HA Green, so birdnet is on the UPS through PoE. The low-battery sequence is written out as a node because the ordering is the point.
- **Indoors**: MacBook as image target (`receive.sh` forced command, now per host), pi-fleet MCP as the admin path to every Pi, HA Green → iPhone TTW17 for all alerts, Dropbox.

### Choices

- Personal palette, not the DCT Clay style (personal project rule).
- Category = colour: plum bird pipeline, pine backups and notifications, ochre power and UPS, slate admin, stone existing/external.
- Admin ssh paths from pi-fleet are stated in its node rather than drawn: three dashed lines across every container added noise and no information.
- `pi-notify` on birdnet is not drawn separately; the birdnet image node says it notifies, and the one webhook edge from netutil stands for the mechanism.
- Edge labels kept to the few that carry a fact (FSD, PoE, USB, SPI, weekly over ssh).

### Known limitations

- Straight-line edges; the UPS → NUT primary line crosses the netutil clone nodes. Acceptable at this size.
- pi-5-loaded is not drawn: unchanged today and not yet on the UPS.
