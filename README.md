# Cabin Zigbee Automations

Home Assistant automations for the cabin's Zigbee2MQTT sensor deployment
(M920q stack). Covers freeze/temperature monitoring, heater control, entry
lighting, and intrusion deterrence. Leak detection and the main water
shutoff valve are deliberately **not** covered here — see Status below.

## Status

**Automation file not yet deployed.** The cabin Zigbee2MQTT mesh is live and
the current leak sensors plus `main_water_valve` are paired. The automation
must still pass a Home Assistant configuration check and be reloaded with the
operator present before it is treated as active.

**Leak detection and `main_water_valve` are owned by FaceoftheCabin's
`WorkflowRuleService`, not by this repo.** This file used to contain a leak
automation (notify + `switch.turn_off` on `main_water_valve`, confirm/
unconfirmed check); it was removed 2026-08-15 once `WorkflowRuleService`
took over that exact trigger/action with its own idempotency, edge-detection,
command-confirmation, and no-automatic-reopen guard, already proven live
against the real valve. Two independent systems each able to close the same
safety-critical actuator was a real duplicate-controller risk, not a
hypothetical one — see `docs/ontology.yaml`'s `trigger_water_leak_detected`
in the `FaceoftheCabin` repo. Test the leak → notify → valve-off chain from
that side; this repo's leak-sensor pairing only needs to get each sensor
reporting cleanly over Zigbee2MQTT/MQTT (see PAIRING_GUIDE.md), nothing HA-
side needs to react to it.

## Contents

- `automations/leak_freeze_automations.yaml` — automation set (leak
  detection/valve shutoff intentionally excluded, see Status above):
  1. Freeze warning (mech room, kitchen, bathroom wall probe)
  2. Low battery notification
  3. Sensor-unavailable catch-all (6+ hours silent)
  4. Mesh health / weak link-quality warning
  5. Freeze-triggered heater auto-on/off (with hysteresis)
  6. Heater max-runtime safety guard (48h)
  7. Entry light dusk-to-dawn control
  8. Intrusion deterrence: radio + light + siren, gated by away-mode
  9. RF tripwire: passive link-quality anomaly detection (advisory only)

## Before deploying

This file still has **placeholder entity IDs** that only become real once
devices are paired in Zigbee2MQTT and renamed to match:

- `CABIN_ALERT_NTFY_TOPIC` in FaceoftheCabin's M920q environment must be set.
  Automations publish `cabin/event/{severity}` and FaceoftheCabin owns the
  notification destination; no Home Assistant mobile-app service is assumed.
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

## Setup and hardware pairing

See [PAIRING_GUIDE.md](./PAIRING_GUIDE.md) for the full Zigbee device
pairing sequence and post-pairing checklist.

## Order confirmed

First hardware round ordered 2026-07-18 via sonoff.tech (order #sn42122,
$180.03 after SOANNI30 discount) and Amazon. Expected processing Monday;
cabin lost power/connectivity same day, so pairing is on hold until both
resolve.

## Key hardware decisions

- **ZBMINIR2** over ZBMINIL2 for the entry light — confirmed neutral
  wiring available, so the neutral-required router-capable switch made
  more sense than the no-neutral end-device version.
- **THIRDREALITY Drip Detect** leak sensors over the cheaper Gen2 (WL2)
  variant — Gen2 drops the 120dB local siren entirely, which defeats the
  original requirement (audible alarm independent of HA/internet).
- **THIRDREALITY Smart Plug Gen3** over Gen2 — stronger repeater/RF
  module, and firmware updates that don't interrupt power (relevant for
  the heater and deterrent plugs specifically).
- **Zigbee clamp-on valve actuator** on the main shutoff instead of a
  full smart water monitor system (Moen Flo, etc.) — ~$35-70 vs.
  $330-550+, and the flow-metering those systems add isn't needed since
  leak detection is already handled by the Zigbee sensors.
- The spare THIRDREALITY leak sensor's `water_leak_buzzer` (separate
  Zigbee2MQTT-exposed property from `water_leak` detection) is
  repurposed as an intrusion-deterrence siren — confirmed this doesn't
  create false leak alerts since the two properties are independent.

## Repo

https://github.com/smrekarfamilia-sudo/CabinSensorAutomationDetection
