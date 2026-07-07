# HA Automations — Reference (as of 2026-07-05)

All automations live in `automations.yaml`. This doc explains the *why* behind each,
since the YAML only shows the *what*. See [integrations.md](integrations.md) for the
underlying entities/integrations these depend on.

## Presence detection (foundation for several automations below)

- **`binary_sensor.house_occupied`** — template sensor (in `configuration.yaml`),
  `on` if `person.tim` OR `person.sara` is home.
- `person.tim` — tracked via `device_tracker.moto_g54_5g` (HA mobile app GPS/zone).
- `person.sara` — tracked via `device_tracker.gala35` (Netgear router presence).
- Netgear integration added 2026-07-05 to get router-based (network-presence) tracking,
  since Sara's phone doesn't run the HA app (battery-life concern) and Tim's GPS tracker
  alone doesn't cover her.
- **Gotcha:** UI-created (storage-based) `person` entries don't pick up `.storage/person`
  file edits without a full core restart — only YAML-configured persons respond to
  `person.reload`. Add device trackers to a person via the UI (Settings → People), not
  by hand-editing the storage file.

## Camera / Frigate detection

### Camera - Person Detected (`camera_person_detected`)
Push notification (image + Ignore 30m/2h/8h actions) when Frigate's person-occupancy
binary sensor goes `on`, for any of the 3 cameras (upper driveway, lower driveway, backyard).

- Gated by `timer.person_alert_inhibit` (idle) — the Ignore action starts this timer via
  **Camera - Person Alert Inhibit Handler**.
- Gated by `sun.sun == below_horizon` **OR** `binary_sensor.house_occupied == off` —
  i.e. always alert at night, and also alert during the day if the house is empty.
  (Originally darkness-only; extended once presence detection existed. This `or`
  condition is intentionally left as an isolated block so a future "away" refinement
  — e.g. once you have finer-grained per-person presence — is a one-line change.)
- Suppressed 9pm–5am **except this one** — see Quiet Hours below.

### Camera - Car Detected (`camera_car_detected`)
Same pattern as person detection, but for `binary_sensor.*_car_occupancy`, its own
`timer.car_alert_inhibit`, and its own inhibit handler. **Is** subject to quiet hours
(unlike person detection).

### Camera - Upper Driveway Person → TV Clip (`camera_upper_driveway_person_tv_clip`)
Wakes the living room TV, speaks a TTS announcement, and displays a Frigate snapshot,
when a person is detected on the upper driveway specifically — but only if the house
is occupied (no point waking the TV to an empty house).

This automation went through a lot of iteration — see
[the TV/casting saga below](#the-tv-casting-saga) for why it's built the way it is.
Current sequence:
1. Trigger: `binary_sensor.upper_driveway_person_occupancy` → `on`
2. Condition: `binary_sensor.house_occupied == on`
3. Condition: `media_player.hisense_55u7la_00a36ac8_television == off` — **only fires
   if the TV isn't already in use.** Added after the kids complained about being
   dumped out of a show when this automation hijacked the TV mid-viewing. If the TV's
   already on, this automation skips entirely — the phone push notification
   (`Camera - Person Detected`) still covers that case.
4. Turn on `media_player.hisense_55u7la_00a36ac8_television` (HomeKit entity)
5. Mute (`switch.hisense_55u7la_00a36ac8_mute` on) — guards against the TV waking to a
   live broadcast input playing an ad at full volume
5. `select_source` → `HDMI1` (where the Chromecast dongle is plugged in)
6. Unmute — now safe, we're off the broadcast input
7. `tts.speak` → "There is someone at the top driveway", via
   `media_player.lounge_room_chromecast`
8. 4s delay (let the TTS clip finish)
9. Cast `http://192.168.1.13:5000/api/upper_driveway/latest.jpg` to the Chromecast
10. 30s view delay
11. Turn off the Chromecast **first** (else the TV won't drop to standby — see below),
    then turn off the TV, then re-mute-off (leave unmuted for normal use)

**Known limitation:** TV hardware volume can't be reliably controlled from HA (see
integrations.md's HomeKit/DLNA notes). If the TTS/announcement is too quiet, bump the
TV's own volume up with its remote — that ceiling then applies to everything cast to it.

## The TV/casting saga

Worth documenting because it cost a lot of trial and error, and the conclusion changes
if the TV or Chromecast is ever replaced.

1. **First attempt:** DLNA (`media_player.hisense_vidaa_tv`, `dlna_dmr` integration) to
   play the actual Frigate clip.mp4. TV woke, but video playback failed — "network error."
2. Found `/homeassistant/docs/ct-105-jellyfin.md`, which documents Jellyfin's own fight
   with this exact TV: it rejects raw MP4 MIME types, needs forced TS transcoding, and
   has byte-range/seek quirks. This TV's DLNA implementation is just genuinely picky.
3. **Switched to showing a snapshot image** instead of video — much simpler payload.
   Worked over DLNA, but was flaky: needed a precise ~10s delay between waking the TV
   and sending the image, and DLNA's reported state (`playing`) didn't reliably reflect
   whether anything actually displayed (false positives).
4. **Tried casting to the Chromecast dongle** (HDMI1) instead of DLNA — user's idea,
   turned out much more reliable. Image cast worked first try, no delay tuning needed.
   Video (clip.mp4) still failed (stuck buffering) — this Chromecast is 1st-gen,
   and Google EOL'd its Cast SDK in April 2024 (same root cause as Jellyfin's "1st gen
   Chromecast incompatible" finding in ct-105-jellyfin.md). **Images work, video doesn't,
   on this specific device — not fixable.**
5. **Settled on:** Chromecast for image display, HomeKit `media_player` entity for
   TV power/source/mute (works reliably, official protocol). DLNA integration removed
   entirely 2026-07-05 since nothing uses it anymore and it was the least reliable path
   of the three.
6. **Standby issue:** turning the TV off while the Chromecast was still actively casting
   sometimes prevented the TV from dropping into standby (active HDMI signal from the
   source). Fixed by stopping the Chromecast *before* turning off the TV.
7. **Volume:** the HomeKit `media_player.hisense_55u7la_00a36ac8_television` entity has
   no volume characteristic at all (checked the entity registry — HA never even
   discovered one). DLNA volume commands didn't actually apply (same unreliability
   pattern as everything else DLNA on this TV). No reliable in-HA volume control exists
   today. **Real fix identified but not yet implemented:** Hisense/VIDAA TVs run a local
   MQTT broker on port 36669 (default creds `hisenseservice`/`multimqttservice`) that
   supports real volume get/set and full remote-key emulation — wrapped by community HA
   integrations (`ha_vidaatv`, `hacs_hisense_tv` on HACS). Deliberately not installed yet:
   it's an unofficial reverse-engineered integration (vs. HomeKit's official, currently-
   reliable one), and would need a one-time on-screen PIN pairing. Revisit if volume
   control becomes worth the reliability trade-off.

## Garage Door

### Garage Door - Nightly Close at 7pm or Unoccupied (`garage_door_nightly_close`)
Flashes a shed light warning, closes the garage door if open, kills a plug switch —
now triggered **either** at 7pm **or** whenever `binary_sensor.house_occupied` turns
`off` (refactored from a 7pm-only automation once presence detection existed).

### Shed Light - Follow Garage Door (`shed_light_garage_door`)
Unchanged from original — turns the shed light on/off following the garage door's
open/close state, independent of the above.

## Zoe's Bedroom AC — sleep-optimized schedule

Built from actual sleep-science research (see chat history / summarized below), not
just arbitrary times. All stages check `binary_sensor.house_occupied == on` first —
never heats an empty house.

| Time | Automation | Action | Why |
|---|---|---|---|
| 7pm | Evening Schedule | heat to 22.5°C if room < 23°C | comfortable pre-sleep warmth |
| 10pm | Evening Schedule | off | let the room cool — enhanced heat loss at sleep onset is associated with *more* slow-wave sleep, not less |
| 2am | Circadian Low Floor | heat to a 20°C floor if room < 20°C | protects against the body's circadian temperature nadir (~2-4h before wake) compounding with a too-cold room — this is when awakenings are most likely |
| 6am | Circadian Low Floor | bump to 22°C | reinforces the body's own natural pre-wake temperature rise, to ease the wake transition |
| 7:30am | Circadian Low Floor | off | end of schedule |

Research sources used to design this (see chat log for full citations):
core body temperature nadir timing (2-4h before wake), U-shaped relationship between
ambient temp and sleep disruption (optimal ~18-21°C, not the 22.5°C originally used for
evening comfort), and evidence that heat loss during sleep onset *increases* slow-wave
sleep rather than harming it.

Both AC automations push a notification on every actual on/off action — except within
quiet hours (see below), where the 10pm-off and 2am-floor notifications are suppressed
by design (always fall inside the window, so removed outright rather than gated with a
runtime condition that would always evaluate false).

## Climate - All AC Off When Unoccupied (`climate_all_ac_off_unoccupied`)
Turns off `climate.loft_aircon` and `climate.zoe_bedroom` when
`binary_sensor.house_occupied` → `off`. **Deliberately excludes** `climate.tess_climate`
— that's the Tesla's climate control, not a house AC, and auto-off would fight
pre-conditioning before a drive.

## AC Unavailable Alerts (`zoe_ac_unavailable_alert`, `loft_ac_unavailable_alert`)
Both house ACs use the `mitsubishi_wf_rac` custom integration, which talks to each
unit's own WiFi control dongle over a local HTTPS-like protocol (port 51443,
self-signed cert, "beaver" protocol). This dongle has a **known, unresolved upstream
reliability problem** — see
[GitHub issue #106](https://github.com/jeatheak/Mitsubishi-WF-RAC-Integration/issues/106)
— where it randomly stops responding and needs a manual power cycle. Confirmed while
debugging Zoe's AC going `unavailable` (2026-07-08): the dongle still answers ping
(network stack alive) but actively refuses the control port (nothing bound to it) —
and the vendor's own Smart M-Air app can't reach it either, so this isn't an HA/
integration bug, it's the device's embedded software hanging.

**No remote recovery is possible** — the integration exposes no reboot/reset command
(only swing direction and fan speed services), so there's nothing to call even when
briefly reachable. A true auto-recovery watchdog would need a smart plug powering the
dongle so something external to this protocol could power-cycle it; none exists yet
for either unit, so these automations are alert-only: if `climate.zoe_bedroom` or
`climate.loft_aircon` stays `unavailable` for 15+ minutes, send a push notification
so it can be power-cycled manually. Subject to quiet hours (not urgent enough to wake
anyone for).

## Quiet Hours (9pm–5am)

No phone notifications in this window, **except person detection** (the one thing you
always want to know about immediately). Implementation differs by automation:
- **Car detection:** real runtime condition (`condition: not` wrapping a `time` window
  check) since it can fire at any time of day.
- **Zoe AC 10pm-off / 2am-floor:** notify actions removed outright — their trigger times
  are fixed and always fall inside the quiet window, so a runtime check would be dead
  code.
- **Person detection:** untouched, always notifies — the explicit exception.
