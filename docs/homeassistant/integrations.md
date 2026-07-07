# HA Integrations — Reference (as of 2026-07-05)

Full integration list lives in Settings → Devices & Services; this doc covers the
ones relevant to the automations in [automations.md](automations.md), plus notes on
what changed recently and why. See `docs/archive/migration_context.txt` and
`docs/TODO.md` for the broader Proxmox/homelab migration this HA instance is part of.

## Frigate (`frigate`)
- Server: `192.168.1.13:5000` (CT 104 on Proxmox, see TODO.md for infra detail)
- 3 cameras: `camera.upper_driveway`, `camera.lower_driveway`, `camera.backyard`
- Per-camera occupancy sensors used throughout automations:
  `binary_sensor.<camera>_person_occupancy`, `..._car_occupancy`, `..._dog_occupancy`,
  `..._cat_occupancy`, `..._all_occupancy`
- Face recognition sensors also present: `sensor.frigate_{family,zoe,gracie}_last_camera`
- Snapshot/clip REST API used directly by automations (not just the HA entities):
  `http://192.168.1.13:5000/api/<camera>/latest.jpg`,
  `http://192.168.1.13:5000/api/events/<id>/{snapshot.jpg,clip.mp4}`
- **This is the reliable detection path.** Tapo's own native person-detection sensors
  (`binary_sensor.*_person_detection`) exist from the `tapo_control` integration but sit
  permanently `unavailable` and aren't wired into anything — Frigate superseded them.

## Netgear (`netgear`) — added 2026-07-05
- Router: Orbi Pro WiFi 6 ("SXR30 - peachester"), `192.168.1.1`
- Purpose: router-based device presence tracking, specifically for Sara's phone
  (GALA35) which doesn't run the HA companion app (battery-life preference)
- Key entities: `device_tracker.gala35`, `device_tracker.moto_g54_5g_2` (Tim's phone,
  tracked twice — once via mobile app GPS, once via router, both feed `person.tim`)
- **Gotcha hit during setup:** after adding the integration via the UI, entities showed
  `unavailable`/didn't exist until a `homeassistant.reload_config_entry` was forced —
  first poll doesn't always happen automatically on entry creation.
- Also provides a large set of `switch.*_allowed_on_network` (parental-control-style
  kill switches per device) and per-device link/signal sensors — not currently used by
  any automation, just came along with the integration.

## Person / presence
- `person.tim` — device trackers: `device_tracker.moto_g54_5g` (mobile app GPS/zone)
- `person.sara` — device tracker: `device_tracker.gala35` (Netgear)
- `binary_sensor.house_occupied` — template sensor (`configuration.yaml`), `on` if
  either person is home. This is the single source of truth every occupancy-gated
  automation reads from.

## TV control — HomeKit (`homekit_controller`) + Chromecast (`cast`)
The living room TV (Hisense 55U7LA) is controlled via **two separate integrations**,
each covering different things, plus a Chromecast dongle for casting:

- **HomeKit** (`media_player.hisense_55u7la_00a36ac8_television`): power on/off,
  input source select (HDMI1-4, Home, AirPlay, AV, TV Channels), via
  `switch.hisense_55u7la_00a36ac8_mute` for mute. **No volume control exposed** —
  confirmed by checking the entity registry, HA never discovered a volume
  characteristic for this accessory.
- **Google Cast** (`media_player.lounge_room_chromecast`): a 1st-gen Chromecast dongle
  plugged into HDMI1, used to cast Frigate snapshot images to the TV. **Can display
  images reliably; cannot play video** — this specific hardware's Cast SDK was EOL'd
  by Google in April 2024 (confirmed via the same failure mode documented for Jellyfin
  in `ct-105-jellyfin.md`). Not fixable; a native client (Fire Stick / Chromecast w/
  Google TV) is the only real fix for video casting, per that doc's own conclusion.
- **DLNA (`dlna_dmr`) — removed 2026-07-05.** Originally used for both TV wake and
  clip/image playback; proved unreliable throughout (optimistic "playing" states that
  didn't reflect reality, volume commands that silently didn't apply, video playback
  failures). Fully superseded by the HomeKit + Cast combination above; nothing
  references it anymore. See automations.md's "TV/casting saga" for the full story.
- **Not yet done, evaluated but deferred:** Hisense/VIDAA TVs expose a local MQTT
  broker (port 36669, default creds `hisenseservice`/`multimqttservice`) supporting
  real volume control and full remote-key emulation, wrapped by community HACS
  integrations (`ha_vidaatv`, `hacs_hisense_tv`). Would fix the volume-control gap
  properly, but deliberately not installed — it's an unofficial/reverse-engineered
  integration vs. HomeKit's official one, and would need a one-time on-screen TV PIN
  pairing. Revisit if volume control becomes a priority.

## MQTT (`mqtt` — Mosquitto broker)
Used for Frigate's `frigate/events` topic (legacy trigger, since replaced by the
`binary_sensor.upper_driveway_person_occupancy` state trigger — see automations.md).
Still the underlying transport Frigate's HA integration itself depends on. Separate
from the Hisense TV's own local MQTT broker mentioned above (different broker
entirely — bridging them was researched but not implemented, see automations.md's
TV/casting saga for what that would unlock).

## Climate
- `climate.loft_aircon` (Mitsubishi WF-RAC) — house AC, included in "all off when
  unoccupied" automation
- `climate.zoe_bedroom` (Mitsubishi WF-RAC) — house AC, has its own sleep-optimized
  schedule (see automations.md), also included in "all off when unoccupied"
- `climate.tess_climate` — **this is the Tesla's climate control**, not a house AC.
  Deliberately excluded from any auto-off-when-unoccupied logic.

## Tapo (`tapo_control`)
Three Tapo cameras (C310 fixed, 2×C500 PTZ) — RTSP feeds into Frigate is the primary
use now. Native Tapo motion/person-detection entities exist but are unused/unavailable
(superseded by Frigate). See TODO.md for camera-level infra detail (RTSP enable steps,
IP addresses, retention settings).
