# Cabin Zigbee Automations

Home Assistant automations for the cabin's Zigbee2MQTT sensor deployment
(M920q stack). Covers leak detection, freeze/temperature monitoring, the
main water shutoff valve, heater control, entry lighting, and intrusion
deterrence.

## Status

**Not yet deployed.** Hardware ordered (SONOFF + THIRDREALITY, first round
via sonoff.tech and Amazon), arriving after the weekend. Zigbee2MQTT is
configured on the cabin stack but no devices are paired yet.

## Contents

- `automations/leak_freeze_automations.yaml` — full automation set:
  1. Leak detection push alert (all SNZB-05P / THIRDREALITY leak sensors)
  2. Freeze warning (mech room, kitchen, bathroom wall probe)
  3. Low battery notification
  4. Sensor-unavailable catch-all (6+ hours silent)
  5. Mesh health / weak link-quality warning
  6. Freeze-triggered heater auto-on/off (with hysteresis)
  7. Heater max-runtime safety guard (48h)
  8. Entry light dusk-to-dawn control
  9. Intrusion deterrence: radio + light + siren, gated by away-mode
  10. RF tripwire: passive link-quality anomaly detection (advisory only)

## Before deploying

This file still has **placeholder entity IDs** that only become real once
devices are paired in Zigbee2MQTT and renamed to match:

- `notify.mobile_app_YOUR_PHONE` — replace with your actual mobile app
  notify service (Developer Tools > Actions, search "notify").
- Friendly names (`leak_bosch_washer`, `temp_mech_room`,
  `probe_bathroom_wall`, `heater_mech_room`, `light_entry`,
  `deterrent_radio_light`, `router_tripwire_a/b`, `leak_spare_siren`,
  `door_front_contact`, `door_second_contact`, `motion_entry_occupancy`)
  — assign these as friendly names in Zigbee2MQTT when pairing each
  device, per the pairing order and setup notes at the top of the YAML
  file itself.
- `input_boolean.away_mode` — create as a Toggle helper in Home Assistant
  (Settings > Devices & Services > Helpers) before the deterrent/tripwire
  automations will work.

## Hardware inventory (first order round)

- SONOFF ZBDongle-E (coordinator)
- SONOFF SNZB-05P water leak sensor w/ cable — x3
- SONOFF SNZB-02WD IP65 temp/humidity — x2 (mech room, kitchen)
- SONOFF SNZB-02LD probe thermometer — x1 (bathroom outer wall pipe)
- SONOFF SNZB-04P door/window sensor — 2-pack
- SONOFF SNZB-03PR2 motion sensor — x1 (entry)
- SONOFF ZBMINIR2 switch (neutral, router-capable) — 2-pack (entry light)
- THIRDREALITY leak sensor, Drip Detect w/ 120dB siren — 4-pack
- THIRDREALITY smart plug, Gen3, energy monitoring — 4-pack (heater,
  deterrent plug, tripwire routers, spare for siren repurposing)
- Zigbee clamp-on ball valve actuator (3/4" lever, main shutoff)

Separately ordered: a Reolink PoE camera for outdoor coverage (not part
of the Zigbee mesh).
