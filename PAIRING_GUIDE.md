# Zigbee Sensor Pairing & Install Manual

Supplemental install guide for the cabin's Zigbee2MQTT deployment on the
M920q. Read this before touching any hardware — pairing order matters, and
rushing it is the #1 cause of "device not found" headaches.

## Prerequisites

- [ ] ZBDongle-E coordinator physically plugged into the M920q (must be
      the M920q itself — Zigbee2MQTT needs direct USB access to the
      radio; it can't use a dongle on a different machine, even over
      Tailscale)
- [ ] Zigbee2MQTT frontend reachable: `http://<m920q-ip>:8080` — confirm
      "Bridge: online" with no errors before pairing anything
- [ ] Home Assistant reachable: `http://<m920q-ip>:8123`
- [ ] All devices unboxed, batteries not yet inserted (insert one at a
      time, per the sequence below)

## Core pairing rule

**Pair one device at a time.** Open "Permit join," pair a single device,
confirm it, rename it, close or let the join window expire, then move to
the next. Pairing multiple devices simultaneously causes join collisions
that are disproportionately annoying to untangle versus the few extra
minutes one-at-a-time costs.

## Pairing sequence

For each device:
1. In the Z2M frontend, click **Permit join (all)** (stays open ~254s by
   default).
2. Pop the battery cover / hold the device's reset pin (~5s, until the
   LED blinks) to trigger pairing.
3. Wait for it to appear in the Z2M device list before starting the next
   device.
4. Immediately rename it (**Friendly name** field) to the name in the
   table below — this is what makes Home Assistant's auto-generated
   entity IDs predictable and matches what the automations file expects.
5. Confirm it shows a recent **Last seen** timestamp and non-zero
   **Battery %** before moving on.

| # | Device | Friendly name | Notes |
|---|---|---|---|
| 1 | SNZB-04P door sensor #1 | `door_front_contact` | |
| 2 | SNZB-04P door sensor #2 | `door_second_contact` | |
| 3 | SNZB-03PR2 motion | `motion_entry_occupancy` | Place at entry |
| 4 | SNZB-05P leak (cable) #1 | `leak_bosch_washer` | |
| 5 | SNZB-05P leak (cable) #2 | `leak_lg_washer` | |
| 6 | SNZB-05P leak (cable) #3 | `leak_liebherr_fridge` | Cable under water line connection |
| 7 | SNZB-02WD temp/humidity #1 | `temp_mech_room` | Near LG washer |
| 8 | SNZB-02WD temp/humidity #2 | `temp_kitchen` | Behind fridge/dishwasher |
| 9 | SNZB-02LD probe | `probe_bathroom_wall` | Press probe to supply pipe on exterior wall, once paired |
| 10 | ZBMINIR2 switch #1 | `light_entry` | Wire to entry light circuit |
| 11 | ZBMINIR2 switch #2 | *(assign on install)* | Spare |
| 12 | THIRDREALITY leak #1 | `leak_alarm_mech_room` | |
| 13 | THIRDREALITY leak #2 | `leak_alarm_bathroom` | |
| 14 | THIRDREALITY leak #3 | `leak_spare_siren` | Spare unit, buzzer repurposed for intrusion deterrence — see automations file |
| 15 | THIRDREALITY leak #4 | *(assign on install)* | Spare |
| 16 | THIRDREALITY plug #1 | `heater_mech_room` | Mechanical room heater |
| 17 | THIRDREALITY plug #2 | `deterrent_radio_light` | Feeds a power strip with radio + lamp |
| 18 | THIRDREALITY plug #3 | `router_tripwire_a` | Position across chokepoint (hallway/entry) |
| 19 | THIRDREALITY plug #4 | `router_tripwire_b` | Position across chokepoint from A |
| 20 | Zigbee valve actuator | `main_water_valve` | Clamp on main shutoff lever |

## After all devices are paired

1. **Cross-check entity IDs.** In Home Assistant, go to
   **Settings > Devices & Services > Zigbee2MQTT** and confirm the
   auto-generated entity IDs match what's referenced in
   `automations/leak_freeze_automations.yaml`. Friendly names above
   should produce predictable IDs (e.g. `binary_sensor.leak_bosch_washer_water_leak`),
   but always verify rather than assume.
2. **Set the notification target.** Find your actual mobile app notify
   service under **Developer Tools > Actions** (search "notify") and
   replace every `notify.mobile_app_YOUR_PHONE` placeholder in the
   automations file with it.
3. **Create required helpers** (Settings > Devices & Services > Helpers):
   - `input_boolean.away_mode` — Toggle
   - `input_boolean.pause_automations` — Toggle
   - `input_select.cabin_time_window` — options: "Last Hour", "Last Day",
     "Last Week" (for the Cabin Controls dashboard)
4. **Deploy the automations file** — merge into `automations.yaml` or
   reference via `!include`, then reload automations
   (Developer Tools > YAML > Reload Automations, or restart HA).
5. **Test the full chain on one leak sensor** before considering this
   done — wet a paper towel, touch it to a probe/sensor, confirm:
   detection → push notification → main valve closes → siren behavior
   (if applicable) all fire correctly. Don't assume the automation logic
   is correct just because the YAML loaded without errors.
6. **Leave the main valve OPEN** before walking away from the test.

## Common pairing issues

- **Device won't join / times out:** confirm "Permit join" is still
  active (it expires after ~254s) and that you're within a few feet of
  the coordinator for the initial pair — range issues during pairing are
  more sensitive than steady-state mesh operation.
- **Device joins but shows "Interview failed" or incomplete info:**
  remove it from Z2M and re-pair; this is usually a mid-pairing
  interruption, not a bad device.
- **Battery % shows 0 or missing right after pairing:** normal for the
  first few minutes — give it a report cycle before assuming it's dead
  on arrival.
- **Weak link quality on a device far from the coordinator:** this is
  what the mains-powered plugs/switches are for — they act as routers.
  If a battery sensor shows persistently low LQI, consider whether a
  nearby router plug would help before assuming the sensor itself is
  faulty.

## Related docs in this repo

- `automations/leak_freeze_automations.yaml` — full automation logic,
  including setup notes at the top of the file for helpers and scripts
  referenced above.
- `dashboards/cabin_controls_dashboard.yaml` — status-driven Lovelace
  view for non-technical override/reset access (Active / Recent /
  Ready to roll / Out of Action).
